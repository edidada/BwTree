# 分支差异说明

本仓库 `edidada/BwTree` fork 自 [wangziqi2013/BwTree](https://github.com/wangziqi2013/BwTree)（Open BwTree，CMU Peloton 项目配套的 Bw-Tree 实现）。远端 `origin` 上保留了上游的若干历史分支，本文说明它们各自做了什么、以及和 `master` 的关系。

> 对比基于本地 `origin/*` 快照（2026-11-11）。文中 "ahead/behind" 指 `master..分支` / `分支..master` 的提交数。

## 一览

| 分支 | 顶端提交 | 日期 | ahead / behind | 与 master 的关系 | 主题 |
| --- | --- | --- | --- | --- | --- |
| `master` | `09b7354` | 2018-11-14 | — | 主干 | 已合并全部特性分支，另有文档、License、SIGMOD 幻灯片 |
| `epoch-manager` | `f855225` | 2016-11-26 | 0 / 10 | 已合并（是 master 的祖先） | Epoch 垃圾回收器、线程池、线程注册/注销 |
| `preallocation` | `29a5f80` | 2016-10-29 | 0 / 224 | 已合并（是 master 的祖先） | 叶子/内部节点预分配、利用率统计 |
| `reverse-iterator` | `3c5b506` | 2016-11-23 | 0 / 80 | 已合并（是 master 的祖先） | 反向迭代器、迭代器测试 |
| `primary-key` | `a72d8ac` | 2017-05-28 | 87 / 4 | **仍然分叉，未合并** | 主键/变长键支持、GC 重写、update node |

注意：前三个分支的 `ahead` 都是 0，且 `git merge-base --is-ancestor origin/<分支> master` 为真 —— 它们是 master 的**祖先**，也就是说这些特性已经通过 PR 合进主干，分支只是停留在合并时的旧快照上。所以"分支与 master 的差异"主要来自 master 后来新增的提交，而不是分支上有独有代码。

## 各分支细节

### master（主干）

当前顶端 `09b7354 Fixing BF test`（2018-11-14）。主干上除了合并各特性外，还有：

- `README.md` 的补充说明、Apache License 信息（`445df87`）；
- `SIGMOD 2018.pptx` 幻灯片（`2e3b605`、`a01fc0e`、`772c7af`）；
- `src/bwtree.h` 与 `stl_test/bloom_filter.cpp` 的小修补（`09b7354`、`16c4ab4`）。

相对 `preallocation` 分支，master 多出约 4000 行改动，主要是把各分支的测试代码（`test/test_suite.h`、`benchmark_art_full.cpp` 等）补齐；相对 `reverse-iterator` 分支，master 在 `src/bwtree.h` 上又多了约 600 行；相对 `epoch-manager` 分支只差 4 个文件、约 30 行。

### epoch-manager

顶端 `f855225 Fixed a bug that incorrect thread local counter is used`。该分支围绕 **epoch-based 垃圾回收**：

- 用线程池替换临时线程（`4c290c5 Switch to thread pool`）；
- 加入线程注销（`309eb81 Adding unregister thread`）；
- 清理旧 epoch 数据、修正 thread-local 计数器使用错误（`bee7aaa`、`f855225`）；
- 通过 PR #59 合并进 master（`b673369`）。

对应概念见 [Bw-Tree 概念入门](bwtree-concepts.md) 的 "Epoch / GC" 一节。

### preallocation

顶端 `29a5f80 Revert "Adding class MicroIndex"`。该分支探索 **节点预分配**：

- `BWTREE_PREALLOCATION` 编译开关与预分配大小常量（`2fea99f`、`fa2472d`）；
- 统计计数器与叶子/内部节点的空间利用率统计（`b110dc8`、`005c131`）；
- 重写垃圾回收相关代码（`789c787 Finish changing the garbage collector`）；
- 最后回退了试验性的 `MicroIndex` 类。

它是四个分支中最早的快照（落后 master 224 个提交）。

### reverse-iterator

顶端 `3c5b506 Removing unused code`。该分支实现 **反向迭代器**：

- `src/bwtree.h` 新增约 600 行（`BackwardIterator` 相关逻辑）；
- `test/test_suite.h` 与 `iterator_test.cpp` 增加正向/反向遍历校验；
- 顺带扩充了 benchmark 代码。

对应概念见 "迭代器 / B-link 侧链" 一节。

### primary-key（唯一未合并的分支）

顶端 `a72d8ac Adding support for update node`。这是唯一一个**仍有 87 个独有提交**的分支，属于一次较大的重构：

- `src/bwtree.h` 改动约 2800 行，`src/thread_local.h` 新增 196 行；
- 新增 `src/thread_local.h`；
- 引入 `BWTREE_PREALLOCATION` 常量、搜索捷径开关、新的垃圾回收器；
- 支持 **update node**（原地更新记录），面向主键索引场景；
- 中间有若干 "does not compile yet" / "Making it compile" 的过渡提交，说明它是一条进行中的开发线。

`git diff --stat origin/primary-key master` 显示方向相反：master 相对该分支删掉了约 2300 行、加回约 730 行。也就是说主干没有采纳这条重构，两个方向的代码已经明显不同。想看它独有的内容，用 `git log master..origin/primary-key`，不要用 `git diff master...origin/primary-key`（三点 diff 在分叉时容易误读）。

## 本分支 `edidada/master`

从 `master` 的 `09b7354` 切出，只包含文档与 CI 配置，不改任何 C++ 源码：

- `docs/branches-overview.md`（本文）
- `docs/bwtree-concepts.md`（新手概念入门）
- `.github/workflows/build-and-test.yml`（GitHub Actions 编译 + 运行测试）

## 自己复核这些结论

```bash
git fetch origin --prune

# 分支是否为 master 的祖先
git merge-base --is-ancestor origin/epoch-manager master; echo $?   # 0 表示已合并

# 各有独有提交 / 落后多少
git rev-list --count master..origin/epoch-manager
git rev-list --count origin/epoch-manager..master

# 分支独有提交列表
git log --oneline master..origin/primary-key

# 代码差异（两点 diff 更直观）
git diff --stat origin/primary-key master
git diff --stat origin/preallocation master
```
