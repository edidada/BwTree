# Bw-Tree 概念入门（新手向）

这篇文档面向第一次接触 Bw-Tree 的读者，解释它的**结构（Structure）**、**优势（Strengths）**以及本仓库里常见的术语。读完后应该能看懂 [分支差异说明](branches-overview.md) 里各个分支在改什么。

## 1. 一句话概括

Bw-Tree 是微软研究院在 2013 年 ICDE 论文 *"The Bw-Tree: A B-tree for New Hardware Platforms"* 中提出的一种**无锁（latch-free / lock-free）B+ 树索引**，为多核 CPU 和闪存/大内存设计。官方实现用在 SQL Server Hekaton、Azure DocumentDB、Bing 等产品里。

本仓库（Open BwTree）是该论文的一个开源 C++ 实现，与 CMU 的 Peloton 数据库原型一起开发。

**重要边界**：Bw-Tree 只保证单次操作的**原子性**，不提供事务的**隔离性**。若需要完整事务语义，必须在它之上再加一层并发控制（MVCC / 锁管理器）。

## 2. 为什么需要它：传统 B+ 树的瓶颈

传统 B+ 树在修改页面时要用 latch（读写锁/自旋锁）保护树节点。高并发下，哪怕线程访问的是不同叶子，也可能在**根节点和内部节点**上排队，形成伸缩性瓶颈；在闪存上，"就地更新整页"还会带来写放大。

Bw-Tree 的两个核心思路：

1. **不就地改页**：所有修改都以"增量记录"追加，靠原子指针切换生效；
2. **间接寻址**：树节点之间引用的是逻辑 ID，通过一个可 CAS 的映射表找到物理地址。

于是线程之间不需要 latch，冲突只发生在一次 CAS 上。

## 3. 结构（Structure）：六个基本零件

### 3.1 逻辑页 ID（PID）与映射表（Mapping Table）

树里每个节点都有一个逻辑编号 **PID**。父子节点里存的是子节点的 PID，而不是内存地址。

**映射表**是一个数组：`PID -> 物理地址`。它是整个结构的"枢纽"：

- 查找时：由 PID 查表得到当前页面地址；
- 修改时：造好新页面/新增量，然后用一次 **CAS（Compare-And-Swap）** 把映射表里该 PID 的指针从旧值换成新值；
- CAS 成功 = 这次修改原子地生效；失败 = 有别的线程抢先了，重读再重试。

> 本仓库的已知限制：映射表大小是固定的，溢出后不能动态扩容（见 README "Known Problem"）。

### 3.2 增量链（Delta Chain）

页面不是被"改写"，而是被"叠加"：每次插入/删除/更新都生成一个 **Delta Record**（增量记录），记录"在原来状态之上做了什么"，并把它挂到该 PID 当前版本的头部。多个增量串成一条**链**：

```
Mapping Table[PID]
        |
        v
  [Delta: insert 42] -> [Delta: delete 7] -> [Base Page (排序好的键值数组)]
```

读取时从链头往下走，把增量应用/合并出当前视图。这样写操作只需分配一小块内存 + 一次 CAS，代价极低。

常见 delta 类型（本仓库 `src/bwtree.h` 中）：

- `LeafInsertNode` / `LeafDeleteNode` / `LeafUpdateNode`：叶子上的增删改；
- `InnerInsertNode` / `InnerDeleteNode`：内部节点上增删分隔键与子指针；
- `LeafSplitNode` / `LeafMergeNode` 等：结构变更的中间状态；
- `ABORT` delta：本实现的特有机制，见第 6 节。

### 3.3 页合并（Consolidation）

增量链越来越长会让读变慢。当链长超过阈值时，某个线程会把 Base Page 和链上所有增量**合并**成一棵新的、紧凑的 Base Page，再通过 CAS 替换映射表指针。这一步叫 consolidation（页合并/压实）。

因为旧版本页面不能直接释放（可能还有线程在读），需要配合下面的 epoch 机制做延迟回收。

### 3.4 结构修改操作（SMO：Split / Merge）

页面满了要**分裂**、太空了要**合并**，这类改动树形状的操作叫 **SMO（Structure Modification Operation）**。难点在于一次 SMO 要同时改子节点和父节点，无法用一次 CAS 完成。

Bw-Tree 的做法是把 SMO 拆成多步：

1. 先在子节点上挂一个 "正在分裂/合并" 的 delta（对外宣告中间状态）；
2. 后续访问这个子节点的线程发现中间状态，会**帮忙把剩下的步骤做完**（help-along 协议）；
3. 最后在父节点的增量链上挂 `InnerInsertNode` / `InnerDeleteNode`，完成索引项的更新。

### 3.5 B-link 侧链与 fence key

Bw-Tree 借用了 **B-link tree** 的思想：每个节点有一条指向**右兄弟**的指针，并带有分隔键范围（low key / high key，也叫 fence key）。

这样在分裂过程中，即使父节点还没更新，查找也能顺着右兄弟链走到正确的新节点；同时也是**正向/反向迭代器**（`forward / backward iterator`）能遍历全树的基础。本仓库的 `reverse-iterator` 分支就是在做这件事。

### 3.6 Epoch 与垃圾回收（GC）

旧页面、被替换掉的增量链不能直接 `delete`，因为可能还有并发读者持有旧指针。本仓库用 **epoch-based reclamation**：

- 每个线程进入/离开操作时登记自己所在的 epoch；
- 被淘汰的内存先挂到对应 epoch 的"垃圾链表"上；
- 当所有线程都越过某个 epoch 后，该 epoch 的垃圾才被真正释放。

`epoch-manager` 分支就是围绕这块做的：线程池、线程注销、修正 thread-local 计数器。相关代码在 `src/bwtree.h` 的 `EpochManager` 与 `src/thread_local.h`。

## 4. 一次插入发生了什么（把零件串起来）

1. 从根节点 PID 开始，查映射表拿页面地址；
2. 沿增量链 + Base Page 判断该走哪个子节点（必要时顺 B-link 右链修正）；
3. 到达叶子 PID：构造 `LeafInsertNode` 增量；
4. **CAS** 把映射表中该 PID 的指针指向新链头 —— 成功即插入完成；
5. 若链过长或节点过满，触发 consolidation 或 split SMO，并回到父节点补索引项；
6. 被换下来的旧内存交给 epoch GC 延后释放。

查找（search）流程相同，只是不产生增量、不做 CAS，因此读者完全无锁、不会被写者阻塞。

## 5. 优势（Strengths）：为什么快

| 优势 | 原因 |
| --- | --- |
| **无锁 / latch-free** | 修改只靠映射表上的一次 CAS，没有全局或节点级 latch，多核下不会在根节点排队 |
| **写放大低、对闪存友好** | 不就地重写整页，只追加小增量；旧页异步回收 |
| **读不被写阻塞** | 读者只看快照，不需要等写者释放锁 |
| **高伸缩性** | 线程间冲突点收敛到单个 CAS，冲突后重试而非睡眠 |
| **缓存/内存局部性好** | 热数据集中在增量链头， consolidation 后恢复紧凑布局 |
| **天然支持 MVCC 类系统** | 旧版本页面按 epoch 保留，便于做多版本 |

代价也要清楚：链太长会拖慢读、CAS 失败需要重试、SMO 协议复杂、实现难度远高于普通 B+ 树。

## 6. 本实现的特有设计与已知局限

README 的 "Improvements / Known Problem" 值得配合阅读：

- **ABORT Delta**：论文里删除节点时，若父节点正好在分裂且分裂键指向被删子节点，会破坏边界键。本实现先在父节点挂一个 `ABORT` delta 挡住后续访问，等 remove 完成再撤掉。副作用是它相当于短暂**锁住父节点**，使结构不再是 wait-free；若挂 ABORT 的线程崩溃，其他线程可能一直自旋。
- **InnerDeleteNode 的检查**：split 完成后补父节点时不必严格校验父节点快照，但 **merge** 的 help-along 必须校验，否则会误判 "merge 已被别人完成"。
- **映射表固定大小**：不能动态扩容，只能预先开"足够大"的表。
- **随机数生成器**：C++11 默认随机数在众核下曾是瓶颈（约 1.00 M op/s），已修正但仍在优化。
- **只保证原子性**：不提供隔离性（见第 1 节）。

## 7. 术语速查表

| 中文 | English | 含义 |
| --- | --- | --- |
| 映射表 | Mapping Table | PID 到物理地址的 CAS 数组 |
| 逻辑页 ID | PID / NodeID | 节点在树中的逻辑编号 |
| 增量记录 | Delta Record / Delta Chain | 追加式修改及其链 |
| 页合并 / 压实 | Consolidation | 把增量链压回新的 Base Page |
| 结构修改操作 | SMO | Split / Merge 等改树形状的操作 |
| 帮忙协议 | Help-along | 发现半成品 SMO 的线程顺手做完 |
| 右兄弟链 | B-link / side link | 分裂期间的容错遍历路径 |
| 边界键 | Fence key / low key / high key | 节点负责的键范围 |
| 纪元回收 | Epoch-based reclamation | 延迟释放旧页面的 GC |
| 比较并交换 | CAS | 无锁生效的关键原子指令 |
| 闩锁 | Latch | 传统 B+ 树用的短期锁，Bw-Tree 不需要 |

## 8. 与各分支的对应关系

| 分支 | 相关概念 |
| --- | --- |
| `epoch-manager` | Epoch / GC、线程池、thread-local |
| `preallocation` | 预分配、空间利用率、`BWTREE_PREALLOCATION` |
| `reverse-iterator` | B-link 侧链、正向/反向迭代器 |
| `primary-key` | 主键索引、`update node`、GC 重写 |
| `master` | 上述全部 + Bloom Filter、ABORT delta 修正 |

本仓库另外两个辅助容器也常一起出现：`src/bloom_filter.h`（加速判断某个键是否可能在本页）和 `src/sorted_small_set.h`（节点内部的小型有序集合）。

## 9. 参考

- Levandoski, Lomet, Sengupta. *The Bw-Tree: A B-tree for new hardware platforms.* ICDE 2013.
- 本仓库 `README.md` 的 Improvements / Known Problem / Compile and Run 章节。
- `docs/branches-overview.md`：各分支改动说明。
