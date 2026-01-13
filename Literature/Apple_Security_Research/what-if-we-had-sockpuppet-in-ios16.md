# What if we had the SockPuppet vulnerability in iOS 16?

## 背景介绍

这篇文章用一个老漏洞 SockPuppet 来检验 iOS 16 里的内核堆分配加固（kalloc_type）到底能把利用难度拉高多少。作者的出发点很明确：只说缓解有效不够，要拿一个曾经非常好用、利用条件又理想的 UAF（use-after-free）来做对照，看看在新分配器规则下还能不能像以前那样稳定复现和利用。

## 漏洞成因

SockPuppet 是 2019 年 Google Project Zero 报告的 XNU 内核漏洞，影响 iOS 12 的多个版本，后来在 iOS 12.3 和 12.4.1 修掉。它之所以适合作为案例，是因为它几乎把攻击者需要的条件都给齐了：释放时机和复用时机都能被攻击者很好控制，被释放的结构体里有不少可用字段，而且自带信息泄露，堆风水非常稳定。文章提到的根因是 in6_pcbdetach() 里释放了 inp->in6p_outputopts（类型是struct ip6_pktopts），但没有把指针置空，导致 socket 里留下悬空指针，后续通过 getsockopt/setsockopt 就能对已经释放的 ip6_pktopts 做有限读写，还能通过其中的指针字段间接读写别的内存。补丁本质上就是 free 之后补一个 NULL 赋值。

## 原始利用链（iOS 12 时代）

原始利用链依赖几组不错的原语。比如用 getsockopt(IPV6_USE_MIN_MTU) 和 getsockopt(IPV6_PREFER_TEMPADDR) 各读 4 字节，能拼出指针；用 getsockopt(IPV6_PKTINFO) 可以从 ip6po_pktinfo 指向的地址拷 20 字节出来；用 setsockopt(IPV6_PKTINFO) 又能把 ip6po_pktinfo 当成指针交给内核去 free。于是旧 exploit 可以做到三件事：
1. 通过喷 Mach message 的 out-of-line ports 数组，把悬空的 ip6_pktopts 位置复用成"指针数组"，再用两个 4 字节读把 Mach port 的内核地址拆出来；
2. 通过喷 OSData，把悬空对象复用成"纯数据块"，在数据里布好目标地址和校验值，实现任意地址读 20 字节；
3. 把第二步的"读"换成"free"，实现任意地址 free。

## iOS 16 的防护机制

iOS 16 的 kalloc_type 直接卡住了这套思路的关键前提：被 UAF 的 ip6_pktopts 分配在 kalloc_type 的某个 type zone 里，但 out-of-line ports 数组在 kalloc_type_var 的 zone，OSData buffer 则在 data submap。也就是说，过去那种"我用 A 类型释放，再用 B 类型喷射来复用同一块内存"的手法，在 iOS 16 下经常根本复用不到同一个分配来源。并且就算你想把"任意 free"硬凑出来，kfree_type() 变成按 zone 释放（zfree(kt_view, ptr)），如果指针不属于那个 zone 会直接 panic，这等于把"把任意指针丢进 kfree()"这条路也堵死了。

## 替代方案探索

文章后半段主要在回答一个问题：就算原始 exploit 不行了，攻击者能不能换一种复用对象、换一种字段重组原语，把能力再拼回来。作者把 ip6_pktopts（固定 192 字节）在 iOS 16 beta1 里对应的 size class 拆开看：同尺寸的 regular type 一共有 194 个，按"signature group"粗分后落在 11 个 bucket（kalloc.type0.192 到 kalloc.type10.192）里，**开机时 bucket 分配是随机的**。ip6_pktopts 在这次启动里落在 bucket 8，而且它不和任何其他类型共享 signature group，意味着没有一个"必然同桶"的稳定复用对象可用，攻击者只能赌碰撞。

接着作者把 ip6_pktopts 里可通过 syscall 触达的字段按能力列出来：哪些是直接读/写的数据字段，哪些是通过指针间接读、free、free+replace 的字段。结论是：直接写那几项对构造指针帮助不大（要么约束太强，要么改出来的指针粒度很尴尬），真正有戏的是那 6 个"通过指针做事"的间接原语，比如能读某指针指向的数据、能把指针 free 掉、能在某些条件下 free 再换成可控的新分配等。作者还专门提到一个有意思的点：ip6po_hlim 这个 1 字节/4 字节范围内的字段可能撞到很多 C++ OSObject 的 retainCount，如果把它写坏，有机会把问题转化成另一个对象上的 UAF，从而把利用迁移到别的 bucket，但这属于更复杂的路线。

## 成功率分析

为了估算"现实里到底能有多稳定"，文章做了一套偏上界的统计：在所有可能同桶的类型里，有多少类型在某些偏移上能形成有意义的 pointer/data overlap，有多少类型在可达性上适合喷射，有多少需要 root 或 entitlement。然后把"可用的 signature group 数量"换算成"碰撞到同一个 bucket 的概率"。作者给出一个直观数字：如果只押一个复用类型，成功率大概只有 8%。想要 75% 碰撞概率，得准备 15 个不同候选；想要 95%，得准备 30 个。问题在于，实际可达、可喷、还能进一步走到写原语的候选并没有那么多。文章估计在更现实的条件下，能用于写原语的 signature group 大概 26 个，这意味着理论最优也就到 92% 成功率，仍然会有约 8% 的启动实例必失败。更糟的是如果攻击者处在非特权状态，受 sandbox 和权限限制，可用候选会继续缩水，成功率上界可能只有 59%，也就是失败至少 41%。

## 结论

最后的结论比较直接：在 kalloc_type 下，SockPuppet 这类 UAF 的利用会变得很不划算。要想把成功率拉上去，攻击者需要写很多套不同的利用分支，然后在运行时根据本次开机的 bucket 分配去选择策略，而且就算把能用的都写全，也很可能仍然达不到过去接近 100% 的稳定性。文章也强调了几个更普遍的点：signature 碰撞对 UAF 可利用性影响很大；pointer/pointer 的重叠仍然可能是实际可走的路线；在 size class 比较"稀疏"的情况下，能触达的替换类型数量会把成功率封顶；sandbox 和 kalloc_type 叠加后，对非特权攻击的限制非常明显。总体意思是，继续推广和完善 kalloc_type，会让大多数内核 UAF 的稳定利用越来越不划算。
