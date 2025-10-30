# ObjC 基础、指针与内存布局思维

> 面向 iOS 内核利用学习的语言与内存模型速成。聚焦“对象如何在内存中长什么样”“消息是怎么被调用的”“哪些字段可控/可伪造”。

---

## 学习目标
- 能用 runtime 术语解释 ObjC 的对象模型（对象、类、元类、isa、SEL、IMP、缓存）。
- 理解 ARC/MRC 引用计数、weak/SideTable、AutoreleasePool 的基本机制与常见陷阱。
- 掌握对象与 block 的内存布局要点（对齐、偏移、tagged pointer、nonpointer isa）。
- 能把 C++ vtable 与 ObjC 消息发送/方法缓存的关系讲清楚，理解二者在逆向与利用中的差异点。
- 了解 arm64/arm64e 的调用约定与 PAC 对 objc 指针（isa/IMP）的影响。
- 具备基础逆向视角：能从二进制/内存里识别 class_ro/rw、method_t/ivar_t、cache_t 等结构；会用 LLDB 与 runtime API 做枚举与验证。
- 能将“指针与布局思维”映射到常见利用套路（伪对象、方法分发劫持、泄露与定位）。

---

## 1. 语言与运行时模型概览

> 可以把 Objective-C 理解为“C 语言 + 运行时库”，面向对象与动态分发都由 runtime 在运行时完成。

### 1.1 从 C/C++ 到 ObjC 的快速映射
- C++ 虚函数表（vtable）≈ ObjC 的“消息发送 + 方法缓存（cache）”。
- 调用 `obj.method(arg)`（C++）≈ 发送消息 `objc_msgSend(obj, @selector(method:), arg)`（ObjC）。
- 接口（C++ 纯虚类）≈ 协议（Protocol）。
- 分类（Category）≈ 给已有类“动态加方法”的机制（C++ 无原生等价物）。
- 析构函数（C++）≈ `-dealloc`；但 ObjC 里很多清理交给 ARC/运行时完成。
- `nullptr` 调用崩溃（C++）vs 向 `nil` 发送消息是“安全无操作并返回 0/NULL”（ObjC）。

一句话总结：ObjC 的“面向对象”其实是“给 C 指针发消息”，编译器把语法糖翻译为对 runtime 的函数调用。

### 1.2 对象与类
- `objc_object`：最小对象头，核心字段是 `isa` 指针，用来找到“这个对象属于哪个类”。
- `objc_class`：类本身也是对象（也有 `isa`），其 `isa` 指向“元类（meta-class）”。元类中保存“类方法”的调度信息。
- 继承与元类结构：

```
实例对象 ──isa──▶ 类对象（保存实例方法表）
   ▲                 │
   │          superclass（继承）
   │                 ▼
  ...            父类对象

类对象 ──isa──▶ 元类对象（保存类方法表）
元类对象的 superclass 指向“父类的元类”，最终收敛到根元类。
```

典型结构（简化，arm64）：
```c
struct objc_object { 
    isa_t isa; // 可能是 nonpointer isa（位域混合标志位）
};

struct objc_class /* : objc_object */ {
    isa_t isa;            // 指向元类
    Class superclass;     // 父类
    cache_t cache;        // 方法缓存（SEL -> IMP）
    class_data_bits_t bits; // 指向 class_rw_t/class_ro_t 的包装
};
```

常见问题：
- 为什么“类方法”和“实例方法”能同名不冲突？——它们分别存放在“元类的方法列表”和“类的实例方法列表”。
- 为什么伪造对象常要控制 `isa`？——因为方法查找、属性访问都由 `isa` 决定走向。

### 1.3 消息发送（objc_msgSend）与方法缓存
- 语法糖示例：
  - `[obj doThing:x y:y]` 会被编译为 `objc_msgSend(obj, @selector(doThing:y:), x, y)`。
  - 其中 `@selector(name:)` 生成一个唯一的 SEL（类似方法名的 interned 符号）。
- 查找流程（简化）：
  1) 查当前类的 `cache_t`（哈希）。命中则直接取到 IMP（C 函数指针）。
  2) 未命中则查方法列表，不在则沿 `superclass` 逐级向上。
  3) 若仍不存在，进入“方法解析/转发链”：
     - `+resolveInstanceMethod:`/`+resolveClassMethod:` 动态添加实现；
     - `-forwardingTargetForSelector:` 指定一个“备胎对象”；
     - `-forwardInvocation:` 最后机会，自行分发或抛错。
- 与 C++ 的差异：C++ 的虚表在编译期固定；ObjC 的方法查找和转发是运行时可变的，更动态。
- nil 语义：向 `nil` 发送消息返回 0/NULL（不会崩溃），这在“失败即无”的路径上非常实用，也可能掩盖 bug。

利用相关思维：
- 伪造对象 + 指向可控“类/方法表”的 `isa`，可诱使系统路径调用到你布置的 IMP；
- 或借助“消息转发”把控制流转移到自定义逻辑。

### 1.4 代码对照：从语法到 runtime

ObjC 声明/实现：
```objective-c
@interface Person : NSObject {
    int _age; // 实例变量（ivar）
}
@property(nonatomic, assign) int age; // 属性语法糖：生成 getter/setter
- (void)say:(NSString *)words;       // 实例方法声明
@end

@implementation Person
@synthesize age = _age; // 把属性绑定到具体 ivar
- (void)say:(NSString *)words {
    printf("%s\n", [words UTF8String]);
}
@end
```

等价的调用展开（概念上）：
```c
id p = objc_msgSend(objc_getClass("Person"), sel_registerName("new"));
// 等价于 [[Person alloc] init] 的一种形式
objc_msgSend(p, sel_registerName("setAge:"), 18);
objc_msgSend(p, sel_registerName("say:"), CFSTR("hello"));
```

要点：方法实现本质是 C 函数指针（IMP），其第一个参数是 `self`，第二个是 `_cmd`（当前 SEL）。

### 1.5 分类（Category）、协议（Protocol）、KVC/KVO 与常见 Hook 点

这一节讲“动态扩展与动态派发”的四个关键工具。它们常被用来“加功能、做埋点、替换实现”，也常是系统调用路径上的可观察/可拦截点。

#### 1.5.1 Category（分类）
- 是什么：在不修改原类源码的前提下，给类“追加方法”。不能新增 ivar（实例变量）。
- 何时生效：应用加载时由 runtime 合并方法列表；同名方法冲突时，最后加载的定义覆盖前者（易埋隐患）。
- 常见用途：打补丁、拆分实现、调试埋点、Method Swizzling（交换实现）。
- 逆向要点：留意 `+load` 时机与 swizzle 影响范围；定位 category 方法可通过运行时枚举或符号搜索。

#### 1.5.2 Protocol（协议）
- 是什么：仅声明方法列表的“接口”，不包含实现，类似 C++ 的纯虚接口。
- 用法：`@interface Foo : NSObject <UITableViewDataSource, UITableViewDelegate>`；对象可在运行时用 `conformsToProtocol:` 判断。
- 必选/可选方法：协议内可用 `@required`/`@optional` 标注；实现类只需满足必选项即可宣称“遵循”。
- 价值：定义回调/委托的契约，降低耦合；对二进制兼容有帮助（添加可选方法不破坏旧实现）。

#### 1.5.3 KVC（Key-Value Coding，键值编码）
- 是什么：通过字符串键访问属性或 ivar 的机制，绕过常规 getter/setter。
- 读取顺序（简化）：
  1) 查找 `-valueForKey:` 自定义覆盖；
  2) 查找 `-get<Key>`/`-<key>`/`-is<Key>` 等命名；
  3) 查找直接可访问的 ivar（`_key`、`key` 等模式，是否可访问取决于 KVC 合规设置）。
- 写入顺序（简化）：优先找 `-set<Key>:`，否则尝试直接写 ivar，失败则触发 `-setValue:forUndefinedKey:`（默认抛异常）。
- 典型用途：
  - 反射式访问/设置（如快速配置对象）；
  - 访问嵌套键路径 `setValue:forKeyPath:`（如 `person.address.city`）。
- 风险：
  - 误拼写键导致 `-setValue:forUndefinedKey:` 崩溃；
  - 可能越过访问控制（如直接改 ivar），调试期方便，发布期要谨慎；
  - 触发 KVO（见下）导致额外副作用。

（示例略，实践时建议在沙箱工程中尝试，避免污染生产代码路径。）

#### 1.5.4 KVO（Key-Value Observing，键值观察）
- 是什么：对某个键（通常是属性）建立观察，属性变化时收到回调。
- 原理（现代实现，精简）：runtime 可能为被观察对象动态生成子类，重写 setter，并“isa 替换”为该子类，从而插入通知逻辑。
- 基本用法：在对象上 `addObserver:forKeyPath:` 注册观察并在 `observeValueForKeyPath:` 接收回调，配对调用 `removeObserver:`；示例代码可在实验工程中自测。
- 常见坑：
  - 漏掉 `removeObserver:` 造成崩溃（对象销毁后仍回调）。
  - 被观察对象在多线程频繁变更，回调时机与一致性问题。
  - 因为 isa 被替换，调试时你看到的类名可能是 `_NSKVONotifying_Foo`。

#### 1.5.5 常见 Hook 触发点（实践向）
- 生命周期与内存管理：`-dealloc`、`-viewDidAppear:`、`retain/release`（MRC/底层）。
- 描述与日志：`-description`、`-debugDescription`（日志、错误路径常调用）。
- 复制与编码：`-copy`/`-mutableCopy`、`NSCoding` 的 `-encodeWithCoder:`/`-initWithCoder:`。
- 集合与枚举：集合遍历、排序、比较回调。
- 属性变更：KVO 回调；或者 setter 本身（swizzle setter）。
- 转发链：`+resolveInstanceMethod:`、`-forwardingTargetForSelector:`、`-forwardInvocation:`。

选用建议：
- 只读观测：优先 KVO（轻侵入），注意移除；
- 替换实现：Category + Swizzling，但要记录并可恢复；
- 仅加方法：Category 即可，避免覆盖同名；
- 动态路由：利用“转发链”集中处理未知选择子。

与 C/C++ 的思维差异小结：
- “绑定到类型的虚表”变成“运行时查缓存/方法列表/转发”；
- “编译期确定”变成“运行时可增删改（Category/Swizzling/KVC/KVO/Forwarding）”；
- 以上机制既强大又易误用，需配合单元测试与最小化影响面。

### 1.6 arm64 调用约定与 objc_msgSend 参数
- 寄存器惯例（AArch64）：
  - x0~x7 传递前 8 个整型/指针参数；x0 也用于返回值；
  - 在 ObjC 方法里：x0 = self，x1 = _cmd（SEL），后续参数从 x2 开始；
  - 浮点参数走 v0~v7；超过寄存器数量的参数走栈。
- `objc_msgSendSuper` 额外使用 `struct objc_super` 来指定“从父类开始查找”。
- 结构体返回（stret）：在 arm64 上大多走寄存器，不再像 armv7 那样用特殊调用约定（历史包袱减少）。
- 逆向意义：
  - 碰到 `objc_msgSend` 调用点，x0/x1 往往能帮助你在调试时识别“接收者对象”和“方法选择子”；
  - 这对定位动态分发链条非常关键。

### 1.7 运行时核心数据结构（逆向视角，简化）
以下为常见 runtime 结构的“伪代码”形态，字段命名取主流 objc4 源码思路，便于你在二进制里对号入座：

```c
// 方法条目（实例/类方法都用）
struct method_t {
    SEL name;          // 选择子
    const char *types; // 编码字符串，描述参数/返回类型
    IMP imp;           // 实现函数指针
};

// 成员变量条目
struct ivar_t {
    uintptr_t offset;   // 相对对象起始的偏移（注意对齐）
    const char *name;   // 名称
    const char *type;   // 编码
    uint32_t size;      // 大小
};

// 只读的类元数据（编译期生成）
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;  // 实例起始偏移
    uint32_t instanceSize;   // 实例大小（含对齐）
    const uint8_t *ivarLayout;
    const char *name;        // 类名
    method_list_t *baseMethods;  // 方法列表
    protocol_list_t *baseProtocols;
    const ivar_list_t *ivars;    // ivar 列表
    const uint8_t *weakIvarLayout;
    property_list_t *baseProperties;
};

// 运行期可写部分（类加载后构建）
struct class_rw_t {
    uint32_t flags;
    uint32_t version;
    const class_ro_t *ro;  // 指向 ro
    method_array_t methods; // 可能会合并 Category 方法
    property_array_t properties;
    protocol_array_t protocols;
};

// 方法缓存（哈希表）
struct bucket_t { SEL key; IMP imp; };
struct cache_t {
    bucket_t *buckets; // 指针（可能经过压缩/变体）
    mask_t mask;       // 桶掩码（大小 = mask+1）
    mask_t occupied;   // 已占用数量（便于扩容）
};
```

识别技巧：
- `class_ro_t.name` 往往是可读的 C 字符串（类名），可作为扫描锚点；
- `method_t.types` 是一串编码（如 `v@:@` 等），常出现在可读区；
- `cache_t` 的 buckets 列表里交替出现看似函数指针与“选择子地址”的模式；
- `nonpointer isa` 需先与掩码 AND 后得到真正类指针，注意 arm64e 的签名影响。

### 1.8 arm64e PAC（指针认证）对 ObjC 的影响（提要）
- 概念：在 arm64e 上，函数指针、返回地址等可能带签名（PAC），以防止被随意篡改；
- 对 ObjC 的两个关键点：
  1) IMP 函数指针可能被签名；
  2) isa/类指针在某些路径也会涉及签名/校验；
- 逆向与利用的影响：
  - 直接伪造指针可能失败（签名不匹配）；
  - 需要借助“在正确上下文中生成/搬运”或“剥离签名”的 API/原语；
  - 细节依平台与版本而异，实践中需结合设备与系统版本验证。

---

## 2. 内存管理与所有权

### 2.1 ARC/MRC 与引用计数
- MRC：手动 `retain/release/autorelease`；ARC：编译器插桩管理。
- 引用计数存放：
  - 小对象可能在对象头位域（nonpointer isa）里记录；
  - 否则落在 SideTable 哈希结构（包含 `RefcountMap` 与 `WeakTable`）。
- 关键词语义：
  - `strong` 持有；`weak` 非持有且自动置空（需要 WeakTable）；
  - `assign` 仅赋值（对对象不安全）；`unsafe_unretained` 类似 assign；
  - `copy` 拷贝值（对 `NSString`/`block` 很关键）。

### 2.2 AutoreleasePool
- 池以栈式嵌套存在，线程局部；RunLoop 边界行为会影响释放时机。内核利用侧只需知道其影响对象生命周期与内存峰值，避免误判悬挂对象。

### 2.3 弱引用与 SideTable
- `WeakTable` 记录对象地址 -> 弱引用集合；对象销毁时遍历清零。
- 引申风险：伪对象若未正确参与 SideTable 生命周期，`weak` 操作可能崩溃或泄露。

---

## 3. 对象内存布局与对齐

### 3.1 nonpointer isa 与标志位（arm64）
- 64 位下常见“非纯指针” isa：高低位混入标志，如 `has_assoc`、`has_cxx_dtor`、引用计数压缩位等。
- 通过掩码 `ISA_MASK` 还原真正类指针；arm64e 设备上 isa/函数指针可能经 PAC 加签。

### 3.2 ivar 布局、对齐与偏移
- 实例大小 `class_getInstanceSize(Class)`；ivar 偏移可由 `ivar_getOffset(Ivar)` 获取。
- 对齐规则：以最大成员对齐；在 64 位下指针 8 字节对齐。
- 属性到 ivar：`@synthesize` 生成访问方法；也可能使用“隐藏”下划线 ivar。

### 3.3 Tagged Pointer
- 小对象直接把值编码在指针里（如小的 `NSNumber/NSDate/NSString`）。
- 判定：高位 tag 模式，不可对其做普通内存读写。

---

## 4. 指针基础与常见陷阱

- 指针大小：LP64 下指针 8 字节；注意结构体/数组中的对齐填充。
- 悬挂指针（dangling）与双重释放：与 `weak`/`SideTable` 交互产生的崩溃模式。
- 指针算术：仅在同一对象/数组内是定义良好的；跨界是 UB。
- 函数指针与 IMP：签名必须匹配；调用约定错误会导致崩溃。
- 桥接：CoreFoundation/Objective-C 的 toll-free bridging：`__bridge`/`__bridge_transfer`/`__bridge_retained`。

---

## 5. Block 内存布局

Block 本质是结构体 + 函数指针：
```c
struct Block_literal {
    void *isa;           // _NSConcreteStackBlock / _NSConcreteMallocBlock 等
    int flags;           // 是否已拷贝到堆、是否有 copy/dispose helper
    int reserved;
    void (*invoke)(void *, ...); // 执行入口
    struct Block_descriptor *descriptor; // 大小、辅助函数指针
    // 捕获的变量按布局继续排布
};
```
- 栈 Block 需 `copy` 才会变堆 Block；捕获的 `__block` 变量用 byref 结构包装。
- 关键风险：把已释放的栈 Block 当作对象持有；或伪造 Block 以劫持 `invoke`。

---

## 6. 逆向与分析实务

本节给出从“二进制/运行时”两条路径入手的实践要点。

### 6.1 二进制视角（离线）
- 目标与环境：iOS 系统库通常打包在 dyld shared cache 中；分析时可借助解包工具获得单独的 dylib（环境与合法性请自行合规）。
- 符号与方法名：
  - ObjC 方法符号常见形式：`-[Class method:]`（实例方法）/ `+[Class method:]`（类方法）。
  - 没有符号时，可借助字符串（类名/选择子名）与交叉引用定位。
- 结构识别：
  - `class_ro_t.name` 的可读字符串可作为起点；
  - method 列表附近常伴随类型编码串；
  - `cache_t` 桶内常见“SEL 样地址 + 函数指针”的交替模式。
- 工具建议：IDA/Ghidra/Hopper（配合 AArch64 支持）；`strings`/`nm`/`otool`/`dyldinfo` 做基本索引。

### 6.2 运行时视角（在线）
- 枚举与查询：
  - `objc_getClassList`、`objc_copyClassList` 列出类；
  - `class_copyMethodList`/`class_copyIvarList`/`class_copyPropertyList` 做结构探查；
  - `class_getMethodImplementation` 可拿到 IMP 并打印地址；
  - `dladdr` 可反解某函数地址属于哪个镜像/符号（若存在）。
- LLDB 技巧：
  - `po` 打印对象；`expr -l objc --` 执行小段 ObjC/Runtime 代码；
  - 在 `objc_msgSend` 设断点，观察 x0（self）与 x1（_cmd）；
  - `image lookup -n '-[Class method:]'` 定位符号；无符号时配合内存读与调用栈。
- 动态验证：用一段小代码打印某类的 ro/rw、方法与 ivar 偏移，和反汇编/反编译结果相互校验。

### 6.3 C++ vtable vs ObjC 调度的逆向要点
- C++：调用点通常是“读 vptr -> 索引到 vtable -> 跳转到函数”；表项顺序编译期固定。
- ObjC：调用点是“调用 objc_msgSend -> 查 cache -> 沿 superclass 查找/转发”；高度动态。
- 因此：
  - C++ 更偏静态表扫描；ObjC 更依赖运行时状态（缓存、Category 合并、Swizzling）。
  - 想确认某次调用到底跳到哪：最佳方式是在目标进程里对 `objc_msgSend` 附加断点/日志，直接看 x0/x1 与最终 IMP。
