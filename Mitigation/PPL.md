## PPL 是什么
PPL（Page Protection Layer）是 XNU 里的“内核中的内核”，依靠 APRR 把页表与相关元数据放在更高权限域。普通内核代码看到的页表是只读的，能改页表的代码默认不可执行，必须通过有限的 PPL 入口函数（类似从 XNU 做一次系统调用进 PPL）来修改。这把可改页表的入口收窄为受控的小面。

其实现基于以下几个层次：

1. **硬件隔离（APRR）**：ARM 处理器上的 APRR 机制允许将内存页和代码段分组标记，赋予不同的权限属性。PPL 利用这一机制，把页表页、PPL 数据结构、PPL 代码段等分别标记到 PPL 权限域，而其他内核代码和数据在普通权限域。

2. **权限域切换**：正常 XNU 内核运行在低权限域，对页表的访问受限（只读）；修改页表的代码被标记为不可执行。当需要进行页表修改操作时，XNU 调用 PPL 入口例程（通过特殊指令或异常陷入），处理器切换到高权限域（PPL mode），此时 PPL 代码变为可执行，页表页变为可写。

3. **入口控制**：所有页表修改都必须通过有限的 PPL 入口例程，如 `pmap_remove_options_internal()`、`pmap_enter_options_internal()` 等。XNU 侧通过 `pmap_remove_options()`（普通权限）调用，先做基础参数校验，再陷入 PPL。PPL 侧再度验证，确保只有合法的修改才被执行。

4. **执行流程示例**：
   - 用户代码请求释放虚拟页：`munmap()`
   - XNU syscall 处理：调用 `pmap_remove_options(addr, size)`
   - pmap 验证参数合法性（如范围对齐、进程权限等）
   - 调用 `pmap_remove_options_internal()` —— 陷入 PPL
   - 处理器切换到 PPL mode，页表页变为可写
   - PPL 代码再度验证参数，找到对应 L3 页表页，清零要删的 TTE
   - 调用 TLB 失效指令确保 CPU TLB 同步
   - 恢复到普通权限域，返回结果

5. **防护特点**：
   - 纵深防御：即便攻击者在普通内核获得 R/W/X，也无法直接读写页表页（硬件级只读）或执行 PPL 代码（硬件级不可执行）。
   - 最小入口：PPL 只暴露必要的例程接口，内部实现细节对 XNU 隐藏，减少可被攻击的代码面。
   - 参数校验：每次 PPL 调用都会验证参数，防止越界或非法修改。

6. **风险集中化**：虽然 PPL 增加了安全，但安全保证完全依赖于：(a) APRR 硬件正确实现，(b) PPL 入口例程代码无漏洞。一旦入口中发现漏洞（如 pmap_remove_options_internal 的越界），就可能被利用跨越整个 PPL 边界。

> **APRR（ARM Protected Page Regions）**：ARM 处理器的附加权限隔离机制，可以为物理内存页和代码段单独设定访问属性。PPL 的核心就是靠 APRR：在平时，页表页被标记为在普通内核态下只读，修改页表的代码被标记为不可执行；只有进入 PPL 入口例程时，处理器临时切换权限域，让这些代码可执行、页表可写；完成修改后恢复，再次锁定。这相当于在 CPU 硬件层面加了一道门闩。
> 
> **pmap 的作用**：XNU 内核的物理映射抽象层，负责：VA→PA 映射的维护；页表页的动态分配与回收；TLB 的同步与失效。PPL 引入后，凡是涉及页表修改的操作都从普通内核路径转发给 pmap，由 pmap 再转交给 PPL 入口例程在受保护域执行。可以把 pmap 理解为"页表管理的网关"。

## The core of Apple is PPL

原文：Brandon Azad（Project Zero），2020-07-31，《The core of Apple is PPL: Breaking the XNU kernel's kernel》。以下先做逐段直译（含原文代码与图片），末尾附个人笔记与要点。

### 原文翻译

在为 one-byte exploit technique 做研究时，作者考虑了几种仅用物理地址映射原语（即在拿到内核读/写或绕过 PAC 之前）绕过 Apple Page Protection Layer（PPL）的方法。鉴于 PPL 比 XNU 内核更高一层，能在“进入”XNU 之前攻破它很诱人。但最终作者没能想到只靠物理映射就能打破 PPL 的办法。

> **物理地址映射原语**：指能将指定物理内存页映射到可控虚拟地址空间的能力。攻击者通常通过 IOMMU/DMA 配置漏洞、内核驱动的字符设备接口，或在已有内核原语基础上获得此能力。文中强调：虽然物理映射能力强大，但单靠它不足以绕过 PPL——因为 PPL 限制了哪些代码能在 PPL 模式下运行、修改什么。物理映射只是前置的一个环节。

PPL 的目标是阻止攻击者修改进程的可执行代码或页表，即使已经取得内核读/写/执行权限。它利用 APRR 构建了“内核里的内核”来保护页表：正常内核执行时，页表与元数据是只读的，修改页表的代码不可执行；内核只能通过调用“PPL 例程”（类似从 XNU 进入 PPL 的一次系统调用）进入 PPL 修改页表，将可修改页表的入口收缩到这些例程。

作者尝试用 one-byte 技术的物理映射原语绕过 PPL，例如直接映射页表、映射 DART 让协处理器改物理内存、映射控制时钟门控的 I/O。结果都行不通。

> **DART（DMA Address Remapping Table）**：SoC 上的 IOMMU 组件，负责给外设和协处理器（如 GPU、神经网络引擎等）的 DMA 请求做地址重映射。作者尝试过利用协处理器通过 DART 绕过 CPU 侧的 PPL 限制，但因为 PPL 也保护了 DART 的配置页，所以该思路行不通。

但 Project Zero 不会让缓解措施不被攻破。作者转而寻找内存破坏，反编译几个 PPL 函数就找到了漏洞。

![pmap_remove_options_internal 反编译截图](https://projectzero.google/images/oldblog/blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgfLPh_s1200_image1%2810%29.png "pmap_remove_options_internal 调用 pmap_remove_range_options")

PPL 例程 pmap_remove_options_internal() 是从 XNU 到更高权限 PPL 的“PPL 系统调用”之一。XNU 通过 pmap_remove_options() 触发它，先校验参数再进入 PPL。其作用是把给定 VA 范围从某个进程的 pmap 中取消映射。

```c
MARK_AS_PMAP_TEXT static int
pmap_remove_options_internal(
        pmap_t pmap,
        vm_map_address_t start,
        vm_map_address_t end,
        int options)
```

实际移除该虚拟地址范围 TTE 的工作由 pmap_remove_range_options() 完成，入参是要从第 3 级（叶子）翻译表移除的 TTE 范围的起止指针。

```c
static int
pmap_remove_range_options(
        pmap_t pmap,
        pt_entry_t *bpte,   // The first L3 TTE to remove
        pt_entry_t *epte,   // The end of the TTEs
        uint32_t *rmv_cnt,
        int options)
```

![跨 L2 TTE 边界导致 L3 表页越界](https://projectzero.google/images/oldblog/blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgW8K5_s1200_image2%287%29.png "地址范围跨 L2 边界，L3 TTE 处理越界")

不幸的是，pmap_remove_options_internal() 在调用 pmap_remove_range_options() 时，似乎假设传入的虚拟地址范围不会跨越 L3 翻译表边界；如果跨界，计算出的 TTE 范围会越过 L3 表页边界。

```c
remove_count = pmap_remove_range_options(
                pmap,
                &l3_table[(va_start >> 14) & 0x7FF],
                (u64 *)((char *)&l3_table[(va_start >> 14) & 0x7FF]
                        + ((size >> 11) & 0x1FFFFFFFFFFFF8LL)),
                &rmv_spte,
                options);
```

这意味着如果有任意内核函数调用原语，可以直接调用进入 PPL 的包装函数，让 pmap_remove_options_internal() 带着不当的 VA 范围在 PPL 模式下运行，使 pmap_remove_range_options() 读取越界“ TTE ”并将其清零，从而破坏 PPL 保护的内存。

不过，单靠把越界 TTE 清零很难利用：想破坏的数据多半离页表很远，PPL 代码面也不大，清零未必有用，还可能被 PPL 的记账检测到。作者转而关注副作用：**错误的 TLB 失效**。

![错误 TLB 失效可导致陈旧 TLB](https://projectzero.google/images/oldblog/blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhV0OB_s1200_image3%285%29.png "TLB 刷新集合与被删 TTE 集合不一致")

在 pmap_remove_options_internal() 之后，TTE 删除完需要失效 TLB，防止进程通过陈旧 TLB 访问已取消映射的页。TLB 刷新按照传入的 VA 范围，而不是被移除的 TTE。若越界 TTE 属于地址空间的其他区域，失效的 TLB 集合与被移除的 TTE 集合不一致，会留下越界 TTE 的陈旧 TLB。

陈旧 TLB 允许进程在页被取消映射并可能被重用于页表后继续访问该物理页。若能为某个 L3 表留下陈旧 TLB，就能往其中写入 L3 TTE，把任意 PPL 保护页映射为可写。

作者的 PPL 绕过流程：
1. cpm_allocate() 分配连续物理页 A、B。
2. pmap_mark_page_as_ppl_page() 把 A、B 插到 ppl_page_list 头部以便复用为页表。
3. 对 VA P、Q 触发缺页，让 A、B 分别当作映射 P、Q 的 L3 表；P、Q 在 VA 上不连续，但 TTE 在表内连续。
4. 在 Q 上自旋读取，保持 Q 的 TLB 存活。
5. 调用 pmap_remove_options() 删除从 P 开始的两页。漏洞导致 P、Q 的 TTE 都被移除，但只失效了 P 的 TLB。
6. pmap_mark_page_as_ppl_page() 再次把 Q 插到 ppl_page_list 头部，供复用为页表。
7. 为 VA R 触发缺页，让 Q 被分配为 R 的 L3 表，同时仍持有 Q 的陈旧 TLB。
8. 利用陈旧 TLB 写 Q，插入映射让 Q 自身可写。

![陈旧 TLB 驱动的表页重用动画](https://projectzero.google/images/oldblog/blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhVnrT_s1200_image4.gif "利用陈旧 TLB 重写 L3 表")

此绕过报告为 Project Zero issue 2035，修复于 iOS 13.6，并附有把任意物理地址映射到 EL0 的 POC。想看更详细的错误 TLB 失效利用，可参阅 Jann Horn 的博文。

这个漏洞说明了在本无安全边界处新建边界的常见问题：代码会有对安全模型的假设（参数校验位置、暴露与私有功能），而这些假设在新模型下不再成立。作者不惊讶在 PPL 会看到更多类似漏洞。总体而言，PPL 设计让作者印象深刻，边界清晰且未引入更多攻击面。唯一疑问是：PPL 缓解了哪些真实攻击，或只是为更强的缓解铺路？无论如何，作者宁愿有它，并向 Apple 致敬。

### 学习笔记

#### 设计目标与攻击面
目标：就算攻击者拿到内核 R/W/X，也要阻止其直接改进程代码页或页表。作者试过靠物理映射原语绕过：直接映射页表、用 DART 走协处理器、改时钟门控 IO 等，都失败，说明单纯物理访问难以越权。

#### 核心漏洞：pmap_remove_options_internal 越界
PPL 入口 pmap_remove_options_internal 会调用 pmap_remove_range_options 去删除一段 L3 TTE。代码假定 VA 范围不会跨 L3 表页边界，若跨界就按连续数组处理，把越界内存当作 TTE 清零。更糟的是后续 TLB 刷新按传入的 VA 范围，而不是按真实被清除的 TTE 范围，导致“清除的 TTE 集合”与“被刷新 TLB 的集合”不一致，可能留下陈旧 TLB。

#### 利用思路（简化版）
1) cpm_allocate 拿两页连续物理页 A、B。
2) pmap_mark_page_as_ppl_page 把 A、B 放入 ppl_page_list 供页表复用。
3) 触发缺页让 A、B 分别当作 L3 表，映射两个 VA P、Q，它们在 VA 空间不连续，但 TTE 在表中连续。
4) 开一个线程在 Q 上自旋读，保证 Q 的 TLB 不被回收。
5) 调用 pmap_remove_options 让漏洞清除 P、Q 的 TTE，但只刷新 P 的 TLB，留下 Q 的陈旧 TLB。
6) 再次标记 Q 可供 PPL 复用，然后用它去承载新的 L3 表。
7) 借助仍然存活的 Q 的陈旧 TLB，从用户态写回这张 L3 表，插入可写映射，进而改 PPL 受保护的页表页。

效果是把陈旧 TLB 变成对 PPL 受保护页的写原语，从而突破“页表只能由 PPL 路径修改”的边界。

#### 修复与影响
漏洞报告为 Project Zero issue 2035，修复于 iOS 13.6，并附有可把任意物理地址映射进 EL0 的 POC。作者认为 PPL 边界清晰且总体设计稳健，但创建新安全域时容易遗漏旧假设（如参数验证位置、跨界处理），未来类似错误仍需警惕。

