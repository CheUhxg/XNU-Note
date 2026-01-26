# XNU 内存安全的下一个世代：kalloc_type

## 背景介绍

这篇文章深入分析 Apple 在 XNU 内核中引入的 kalloc_type 分配器，这是一套针对内存时序安全（temporal safety）问题的硬化方案。从 iOS 15 开始，Apple 就把 kalloc_type 作为核心防御机制逐步推广，到 iOS 16 时已经覆盖了约 95% 的内核代码。这套系统的目标很明确：让大部分内存破坏漏洞在本质上变得不可靠，把利用难度提到一个新的台阶。

整个设计理念建立在一个简单但有力的原则上——**类型隔离**（type isolation）。核心思想是一旦某个内存地址被分配给某种类型，那么在程序的整个生命周期内，这个地址就只能被该类型或相似类型复用，从而大幅限制 Use-After-Free（UAF）漏洞的利用空间。

## 内存安全问题概览

在进入技术细节之前，先简单回顾一下内存安全的分类。文章把内存安全问题分成五大类：

1. **时序安全（Temporal Safety）**：对象访问必须发生在分配和释放之间，违反这条的是 UAF 和 double-free
2. **空间安全（Spatial Safety）**：访问不能超出分配的边界，违反这条的是 OOB（越界访问）
3. **类型安全（Type Safety）**：对象的类型规则不能被意外改变，违反这条的是类型混淆
4. **确定性初始化（Definite Initialization）**：使用前必须正确初始化，否则可能导致信息泄露或更严重的问题
5. **线程安全（Thread Safety）**：并发访问需要正确同步，违反这条的是数据竞争

kalloc_type 主要针对的是第一类——时序安全问题，因为 UAF 历来是内核利用链中最常见的起点。

## 利用链的层次

作者花了相当篇幅解释为什么要在利用链的早期阶段设置防御。一条典型的内核利用链是这样的：

```
漏洞 → 受限的内存破坏 → 强力的内存破坏 → 任意读写 → 控制流劫持 → 任意代码执行
```

越早在链条上设置障碍，攻击者需要绕过的约束就越多，因为早期阶段的约束会和具体漏洞的限制叠加在一起。比如在"受限的内存破坏"这一步就卡住攻击者，远比等到攻击者已经拿到任意读写后再去做控制流完整性检查要有效得多。到了后期，攻击者手里的工具已经足够强大，绕过缓解措施的方法往往是即插即用的。

这也是 kalloc_type 的设计思路：在攻击者还没有拿到强力原语之前，就让他们的 UAF 利用变得极不可靠。

## 类型隔离的核心原理

类型隔离的想法并不新鲜。WebKit 的 IsoHeap、Chrome 的 PartitionAlloc、以及 grsecurity 的 AUTOSLAB 都在做类似的事情。但在 kalloc_type 之前，主流的操作系统内核还没有大规模部署这类技术。

### 为什么类型隔离有效？

考虑一个经典场景：攻击者有一个 `struct iovec` 的 UAF，然后想办法把这块内存复用成 `struct timespec`。

```c
struct iovec {
    char  *iov_base;  // 指针字段
    size_t iov_len;   // 数据字段
};

struct timespec {
    time_t tv_sec;    // 数据字段
    long   tv_nsec;   // 数据字段
};
```

如果这两个结构体能重叠，`iov_base` 这个指针字段就会和 `tv_sec` 这个数据字段占用同一块内存。攻击者可以通过合法的 timespec API 去写 `tv_sec`，从而控制 `iov_base` 的值，然后再通过 iovec 的读写接口去访问任意地址。这就是经典的"指针/数据混淆"。

类型隔离把这条路堵死了：如果一个地址曾经被用作 iovec，那么它以后只能是 iovec（或者已释放的 iovec，或者完全不可访问的内存）。攻击者再也无法把它变成 timespec，自然也就无法构造出指针/数据的重叠。

### 指针与数据的本质区别

文章还提出了一个很有洞察力的观点：内存可以粗略分为"控制"和"数据"两类。

- **控制**：指针、引用计数、长度、union 标签等，用来组织数据
- **数据**：几乎所有其他东西，程序实际操作的内容

系统接口通常允许直接读写数据，但对控制字段只能间接操作（比如通过 API 递增引用计数，而不是直接写）。这种设计是有道理的：控制字段的具体数值往往只在特定地址空间和特定时刻有意义，是实现细节，不应该暴露给外部。

关键的观察是：**纯数据类型虽然容易被攻击者控制，但本身对利用来说是无趣的**。攻击的本质是夺取"控制"，而不是简单地修改数据。因此，把纯数据类型完全隔离开来，既有安全好处（防止数据/控制的混淆），也有性能好处（可以对纯数据放宽一些昂贵的安全检查）。

## XNU 分配器的演进

### iOS 14 之前：zone_map 和单一 kalloc

在 iOS 14 之前，XNU 使用一个简单的 zone allocator。系统启动时会划出一块虚拟内存作为 zone_map，然后按尺寸创建一系列 kalloc zone（kalloc.16、kalloc.32 等）。每个 zone 管理若干"chunk"（连续的几页内存），chunk 又被均分成固定大小的 element。

这种设计的问题在于：同一尺寸的所有对象都混在一起。攻击者可以轻松完成以下流程：
1. 分配大量含 UAF 的对象
2. 触发 UAF 释放其中一个
3. 释放其他对象，让包含悬空指针的 chunk 变空
4. 触发 zone GC，让这块虚拟地址归还系统
5. 分配大量其他类型的对象，复用这个地址，形成类型混淆

这套流程太稳定了，几乎是内核利用的标准操作。

### iOS 14：zone sequestering 和 kalloc heaps

iOS 14 引入了两个关键改进：

**Zone Sequestering**：zone 不再把空闲的虚拟地址完全归还给系统，而是保留在一个新的列表（z_pageq_va）里。这样，某个 zone 用过的 VA 就永远不会被其他 zone 复用。单类型 zone（比如专门给 proc 或 thread 用的 zone）的 UAF 不再能被直接类型混淆；即使在传统的 kalloc zone 里，也只能在同一尺寸类别内混淆。

```c
struct zone {
    // ...
    zone_pva_t z_pageq_empty;   // 完全空闲的页
    zone_pva_t z_pageq_partial; // 部分使用的页
    zone_pva_t z_pageq_full;    // 完全满的页
    zone_pva_t z_pageq_va;      // 只保留 VA、无物理内存的页（新增）
    // ...
};
```

**Kalloc Heaps**：把传统的 kalloc 拆分成多个"堆"（heap），每个堆是一组按尺寸组织的 zone 集合。最重要的是 KHEAP_DEFAULT（给核心内核用）和 KHEAP_DATA_BUFFERS（给纯数据用）。`kalloc()` 变成 `kheap_alloc(KHEAP_DEFAULT, ...)`，新增的 `kalloc_data()` 则走 `kheap_alloc(KHEAP_DATA_BUFFERS, ...)`。

**Zone Submaps**：zone_map 被进一步细分成多个 submap：VM submap、general submap、data submap。VM 和 general submap 中的 zone 都会被 sequester，而 data submap 的 zone 不会，因为纯数据的 UAF 通常不太容易被利用，而且数据分配常在性能关键路径上。

这套机制大幅限制了跨 zone 的类型混淆，但同一尺寸类别内的混淆仍然可行。

### iOS 15：kalloc_type 登场

iOS 15 引入了 kalloc_type，在每个尺寸类别内部进一步细分。不再是所有 32 字节的对象都挤在 kalloc.32 里，而是根据类型的"签名"（signature）分散到 kalloc.type0.32、kalloc.type1.32 等多个 zone 中。

## kalloc_type 的技术实现

### 类型签名机制

kalloc_type 的核心是编译器生成的类型签名。每个类型按 8 字节粒度被描述成一串数字：

```c
typedef enum {
    KT_GRANULE_PADDING = 0,  // 填充
    KT_GRANULE_POINTER = 1,  // 指针
    KT_GRANULE_DATA    = 2,  // 数据
    KT_GRANULE_DUAL    = 4,  // 未使用
    KT_GRANULE_PAC     = 8   // PAC 保护的指针
} kt_granule_t;
```

比如 `struct iovec` 的签名是 "12"：第一个 8 字节是指针（iov_base），第二个 8 字节是数据（iov_len）。

可以用 lldb 查看：
```
(lldb) showstructpacking iovec
0000,[  16] (struct iovec)) {
    0000,[   8] (void *) iov_base /* pointer -> 1 */
    0008,[   8] (size_t) iov_len  /* data    -> 2 */
}

__builtin_xnu_type_signature(struct iovec) = "12"
```

### kalloc_type_view 结构

每个分配点都有一个 `kalloc_type_view` 结构，在编译时静态生成，存放在 `__DATA_CONST.__kalloc_type` 段：

```c
struct kalloc_type_view {
    struct zone_view        kt_zv;          // 分配算法选中的 zone
    const char             *kt_signature;   // 类型签名字符串
    kalloc_type_flags_t     kt_flags;       // 标志位
    uint32_t                kt_size;        // 类型大小
    void                   *unused1;
    void                   *unused2;
};
```

`kalloc_type()` 是一个宏，会在调用点自动生成对应的 view：

```c
#define kalloc_type_2(type, flags) ({                                      \
    static KALLOC_TYPE_DEFINE(kt_view_var, type, KT_SHARED_ACCT);          \
    __unsafe_forge_single(type *, kalloc_type_impl(kt_view_var, flags));   \
})
```

### 启动时的分配算法

系统启动时，分配算法会处理所有的 kalloc_type_view：

1. **按签名排序**：在每个尺寸类别内，把相同签名的 view 分到一组
2. **前缀合并**：如果签名 A 是签名 B 的前缀，就把它们视为同一个"signature group"。比如 "12211" 和 "122112"，如果没有其他签名是 "122111"，就会被当成同一组
3. **随机分桶**：把 signature group 随机分配到有限数量的 kalloc.type* zone 中

这套机制的妙处在于**随机性**：每次启动时，哪些类型分到同一个 zone 是不确定的。攻击者无法写一个"万能"的 exploit，必须在运行时根据当前启动实例的分桶情况动态调整策略。

### 内存预算与分桶数量

虽然理想情况是每个类型一个 zone（完美隔离），但这会导致内存碎片化急剧增加。经过权衡，Apple 决定分配 **200 个 zone** 来做类型隔离，按各尺寸类别中 signature group 的数量按比例分配。

在 iOS 16 beta1 的 kernelcache 里：
- 总共 3574 个命名的非变长类型
- 1822 个不同的签名
- 合并后 1482 个 signature group

200 个 zone 要装 1482 个 signature group，平均每个 zone 要放 7-8 个 group，这就是"bucketed type isolation"的由来。

## 变长分配的支持

固定大小的类型比较好处理，但变长分配是个大挑战。iOS 15 的 kalloc_type 最初只支持固定大小，后来扩展到支持以下两种变长模式：

1. 单一类型的数组
2. 固定大小的头部 + 单一类型的数组

特别值得注意的是**指针数组**被单独隔离出来。因为攻击者太喜欢用 out-of-line Mach port 数组这类"通用复用对象"了，把它们隔离能大幅提高利用难度。

```c
struct kalloc_type_var_view {
    kalloc_type_version_t   kt_version;
    uint16_t                kt_size_hdr;        // 头部大小
    uint32_t                kt_size_type;       // 单个元素大小
    zone_stats_t            kt_stats;
    const char             *kt_name;
    zone_view_t             kt_next;
    zone_id_t               kt_heap_start;      // 选中的 kheap 起始 zone ID
    uint8_t                 kt_zones[KHEAP_NUM_ZONES];
    const char             *kt_sig_hdr;         // 头部签名
    const char             *kt_sig_type;        // 元素签名
    kalloc_type_flags_t     kt_flags;
};
```

## 编译器支持与自动化采纳

kalloc_type 需要手动采纳，这是个艰巨的任务。为什么不自动化？

作者列出了三种可能的方案：
1. **运行时推断类型**：比如用分配回溯（backtrace）— 但对小对象来说性能代价太高
2. **编译器 pass 自动重写**：难以准确推断类型，对"分配包装器"效果很差
3. **手动接口 + 工具辅助**：虽然费力，但能保证正确性和性能

最终选择了第三种，并且开发了强大的编译器支持。Apple 的内部 Clang 增加了 `-Wxnu-typed-allocators` 警告，能在编译时检测：

- 使用过时的无类型分配 API（kalloc()、IOMalloc() 等）
- free 时的类型签名不匹配
- 用数据分配器分配包含指针的类型
- 用 kalloc_type() 分配头部+纯数据数组的变长类型

编译器还引入了新的 builtin：
- `__builtin_xnu_type_signature()`：获取类型签名
- `__builtin_xnu_type_summary()`：获取类型的粒度信息的按位或

### 解决"指针藏在数据里"的问题

一个常见问题是指针被存储在 `uintptr_t`、`vm_address_t` 这类整数类型里。为此引入了 `xnu_usage_semantics` 属性：

```c
#define __kernel_ptr_semantics __attribute__((xnu_usage_semantics("pointer")))
#define __kernel_data_semantics __attribute__((xnu_usage_semantics("data")))

// 使用示例
typedef uint64_t mach_vm_offset_t __kernel_ptr_semantics;

struct shared_file_mapping_slide_np {
    mach_vm_address_t sms_address __kernel_data_semantics;  // 用户态指针，对内核来说是数据
    mach_vm_offset_t  sms_file_offset __kernel_data_semantics;
    // ...
};
```

通过代码审查和自动化工具，找出所有这类情况并标注，确保签名准确。

## 安全性分析

### 与 IsoHeap 和 PartitionAlloc 的对比

作者把 kalloc_type 和另外两个知名的类型隔离分配器做了详细对比：

**类型隔离强度**：
- **IsoHeap**（最强）：每个类型单独的 heap，绝对隔离
- **kalloc_type**（次之）：随机分桶隔离，同一 bucket 内可复用
- **PartitionAlloc**（最弱）：Chrome 只有 4 个 partition，同一 partition 和 size class 内可任意复用

**元数据保护**：
- **kalloc_type**（最强）：完全外部化，连 freelist 都不在元素内部，UAF 无法攻击分配器元数据
- **PartitionAlloc**（次之）：有 freelist，但元数据区有 guard page 保护
- **IsoHeap**（最弱）：freelist 在元素内部，可能被 UAF 篡改（虽然有保护机制）

**采纳难度**：
- **IsoHeap**：手动采纳，但很容易；只支持 C++
- **PartitionAlloc**：手动采纳，但通过 PartitionAlloc-Everywhere 能覆盖大部分；隔离效果一般
- **kalloc_type**：手动采纳，工具辅助；工作量较大，但支持 C 和 C++；自动化可靠性高

### 随机分桶的优势

虽然随机分桶会降低隔离强度（相比给每个类型单独 zone），但有独特的好处：

1. **降低确定性配对**：如果把签名相似的类型总是分在一起，攻击者会找到稳定的利用对象，exploit 在所有设备上都有效。随机分桶让这种配对变得不稳定。

2. **放大 strict free 的价值**：`kfree_type()` 会检查指针是否属于正确的 zone，释放到错误 zone 会 panic。随机分桶让"错误的 free"更容易在开发阶段被捕获。

3. **不可绕过的随机性**：即使攻击者泄露了所有类型的分桶信息，也只是知道"哪些替换类型在这次启动中可行"，无法让他们预先选好的替换对象在任意设备上都有效。

4. **迫使多路径 exploit**：攻击者必须准备多套利用策略，针对不同的替换类型，然后在运行时动态选择。这大幅增加了开发和维护成本。

### 签名分布的统计数据

iOS 16 beta1 (build 20A5283p) iPhone 13 Pro 的 kernelcache：

| Signature Group | Size Class | 包含的签名数量 |
|----------------|-----------|--------------|
| "1211" | 32 | 228 |
| "12111112122212121111" | 160 | 149 |
| "121111" | 48 | 102 |
| "11" | 16 | 73 |
| "12" | 16 | 57 |

最大的 signature group "1211" 里有 228 个类型，主要来自少数几个 kext（54 个 IO80211 开头，53 个 AppleBCMWLAN 开头）。第二大的 group 是 IOService，这是很多驱动的基类。

**关键统计**：
- 71% 的 signature group 只包含一个签名（最佳情况）
- 随机选择一个类型，中位数是在包含 4 个签名的 group 里
- 平均每个类型所在的 group 有 32.5 个签名
- 29.6% 的类型在独占的 signature group 里

**分桶统计**（8 次启动的数据）：
- 中位数：每个 bucket 11 个类型
- 平均：每个 bucket 18 个类型
- 最小 bucket：3 个类型（size class 32768）
- 最大 bucket（中位数）：270 个类型（size class 32）

### kalloc_type 的已知弱点

作者非常坦诚地列出了几个主要弱点：

**1. 签名碰撞（Signature Collisions）**

随机选择的类型中位数有 3 个其他类型在同一 signature group 里，这意味着仍然存在确定性的复用对象。而且：
- C++ 继承让 sibling 类有相同的前缀，碰撞更多
- IOKit 驱动比核心 XNU 有更多碰撞
- 指针数组都有相同签名，OSArray、out-of-line port array、IOSurfaceClient array 可以互相复用

**2. 简单签名的局限**

当前的 "12" 签名把所有非指针值都当成数据，但很多非指针控制字段（size、offset、index、refcount）其实更像指针。这些字段被标记成 "2" 后，可能和真正的数据字段形成有意义的重叠。

**3. 指针藏在数据里**

虽然有 `__kernel_ptr_semantics` 可以标注，但找出所有这类情况很难。在 regular kalloc_type zone 里，这主要影响签名碰撞；但在 **data submap** 里问题更严重，因为 data submap 的缓解措施本来就比较弱。如果 data submap 里的 UAF 能控制一个隐藏的指针字段，攻击者根本不需要和 kalloc_type 打交道。

**4. 纯数据+非指针控制的混合**

只包含数据和非指针控制字段的类型会被路由到 data submap，和完全由攻击者控制的数据放在一起，成为更容易的目标。

**5. Union 类型**

Union 本身就是在说"同一块内存可以有不同的解释"，违背了类型隔离的初衷。虽然 iOS 16 beta1 已经把包含指针/数据 union 的类型减少到只有 36 个（实际 31 个不同的类型），但仍然存在。

**6. 采纳不完整**

没有采纳 kalloc_type 的分配点会落回 default heap，失去类型隔离保护。这是个持续的风险。

## 可持续性和未来工作

Apple 在持续改进这套系统：

**编译器工具链**：
- 基于 clang-tidy 的自动化采纳工具
- `-Wxnu-typed-allocators` 强制检查
- 编译期检测多种使用错误

**签名方案改进**：
- 研究如何区分非指针控制字段和纯数据
- 增加签名多样性，同时避免把功能相同的类型打散

**具体优化**：
1. **ipc_kmsg 拆分**（iOS 16）：不再在 data submap 里存储内核指针
2. **PAC 保护指针**：从 typed allocation 到 data allocation 的指针都加 PAC
3. **kfree_type() 清零指针**：释放时顺便把传入的指针清零，减少悬空指针

**长期目标**：
- 继续提高采纳率
- 最终完全移除 default heap
- 禁止分配包含 pointer/data union 的类型

## 总结

从 iOS 14 的 kheaps、data split 和 zone sequestering，到 iOS 15 的 kalloc_type，再到 iOS 16 的全面推广，Apple 在 XNU 内核分配器的加固上走了很长一段路。kalloc_type 用随机分桶的类型隔离，在几乎零性能开销和零内存开销的前提下，把 UAF 的利用难度提升了一个数量级。

这套系统的精妙之处在于：它不追求完美隔离，而是通过随机性让攻击者无法写出"放之四海而皆准"的 exploit。即使泄露了所有类型的分桶信息，攻击者仍然需要为每个漏洞准备多套利用策略，并且在运行时动态选择。这种设计哲学让防御方在资源受限的情况下取得了最大的安全收益。

当然，kalloc_type 不是银弹。签名碰撞、data submap 里的隐藏指针、以及未完全采纳的代码仍然是潜在的攻击面。但作为一个在数十亿设备上实际部署的缓解措施，它展示了如何把学术界的类型隔离思想转化为工业级的实现——快速、内存高效、实用。这对整个行业的防御性缓解研究都有很高的参考价值。
