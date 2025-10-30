# “A survey of recent iOS kernel exploits” 学习打卡路线图

- 原作者：Brandon Azad（Project Zero）

---

## 学习目标
- 掌握 C/ObjC 基础，理解指针与内存布局思维。
- 熟悉 Mach IPC 基本概念（端口、消息、rights）。
- 理解 XNU 内核基础与内存分配机制（zone/kalloc）。
- 能够分析 iOS 10.x 至 iOS 12.x 的典型内核利用案例，绘制技术链路图。
- 总结常见技术族谱（如伪造 Mach 端口、dangling port 重占、KASLR 破除路径等）。
- 掌握缓解机制（如 PAN、KTRR、PAC、PPL 等）的映射与应对策略。

---

## 任务清单
- [ ] 完成先修知识（Week 1）
- [ ] iOS 10.x 代表案 5 个流程图/摘要（Week 2）
- [ ] iOS 11 初期 4 个案例摘要（Week 3）
- [ ] multipath 三件套对比与族谱图（Week 4）
- [ ] LightSpeed 线 3 个案例摘要（Week 5）
- [ ] voucher 线 3 个案例摘要（Week 6）

---

## 总体安排（6 周）

### 第 1 周：先修建议

| 日期 | 每日安排 | 链接 |
| :--- | :--- | :--- |
| **Day 1** | C/ObjC 基础，指针与内存布局思维。 | |
| **Day 2** | Mach IPC 基本概念（端口、消息、rights）。 | |
| **Day 3** | XNU 基础与内存分配（zone/kalloc）。 | |

### 第 2 周：iOS 10.x 代表案例

| 日期 | 每日安排 | 链接 |
| :--- | :--- | :--- |
| **Day 1** | In-the-wild 1 (IOAccel/IOSurface 路线)。 | |
| **Day 2** | extra_recipe (异常消息叠加 240B OOB RW)。 | |
| **Day 3** | Yalu102 (userspace fake port → clock brute force → `pid_for_task()`)。 | |
| **Day 4** | ziVA (三参任意内核函数调用 → `copyin/out` 任意读写)。 | |
| **Day 5 (总结)** | 画“iOS10 时代通用利用流”大图，为缓解机制占位。 | |

### 第 3 周：iOS 11 初期（dangling port 与 GC 技巧）

| 日期 | 每日安排 | 链接 |
| :--- | :--- | :--- |
| **Day 1** | async_wake (信息泄露 + 悬挂端口 + 自实现 GC)。 | |
| **Day 2** | In-the-wild 2 (分段 OOL + 4MB 粒度定位 + pipe fake port)。 | |
| **Day 3** | v0rtex (`OSString` 图案 → 4B 任意读 → 七参调用)。 | |
| **Day 4** | CVE-2018-4150 POC (BPF 竞态思路、PAN 影响)。 | |
| **Day 5 (总结)** | 整理“如何在 iOS11 无 `mach_zone_force_gc()` 时强制/诱导 GC”的三种思路。 | |

### 第 4 周：iOS 11.3.1 深入（multipath 三件套）

| 日期 | 每日安排 | 链接 |
| :--- | :--- | :--- |
| **Day 1** | multi_path (16MB 边界、两次 kfree、pipe dangling)。 | |
| **Day 2** | multipath_kfree (预分配 `ipc_kmsg` 链、AGX vtable 派生、三参调用)。 | |
| **Day 3** | empty_list (kalloc.16 写 0 → 端口首 8B 破坏 → 悬挂端口)。 | |
| **Day 4** | 对比学习三者的共同“骨架”与各自独特的“起手式”。 | |
| **Day 5 (总结)** | 输出“ipc_kmsg/pipe 技术族谱”图。 | |

### 第 5 周：iOS 11.4.x（LightSpeed 线）

| 日期 | 每日安排 | 链接 |
| :--- | :--- | :--- |
| **Day 1** | In-the-wild 3 (VXD393/D5500 repeated IOFree)。 | |
| **Day 2** | Spice (双线程竞速 + `IOSurface` 小块喷射)。 | |
| **Day 3** | treadm1ll (固定批次 OOL + 逐条接收 + 喷射命中)。 | |
| **Day 4** | 形成“从伪 OOL 到七参内核调用”的统一抽象。 | |
| **Day 5 (总结)** | 列举“为何这些在 PAN 开启时失效”的技术原因。 | |

### 第 6 周：iOS 12.x（voucher 时代与 PAC 初现）

| 日期 | 每日安排 | 链接 |
| :--- | :--- | :--- |
| **Day 1** | Chaos (`task_swap_mach_voucher` 引用语义 → voucher UaF)。 | |
| **Day 2** | voucher_swap (MIG 引用操控 → 伪 OOL/伪端口 → 读原语)。 | |
| **Day 3** | machswap2 (PAN 版本思路)。 | |
| **Day 4** | 缓解映射复盘（`task_conversion_eval`, PAC/KTRR/PPL, `ipc_port_finalize()` 变化）。 | |
| **Day 5 (总结)** | 产出“从 iOS10→12 的通用利用流演进图”和“见招拆招”表。 | |
