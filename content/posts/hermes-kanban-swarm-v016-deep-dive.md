# Hermes V0.16 Kanban Swarm 深度解析：当 SQLite 变成多智能体协作的中枢

> 一个有趣的问题——如果你让 5 个 AI 智能体同时处理一个复杂任务，它们之间如何共享进度、如何处理依赖、如何在某个节点失败时不至于整体崩溃？Hermes V0.16 给出的答案，是把整个协作过程写进一张 SQLite 表里。

这听起来平平无奇，但它的工程含义值得深究。

---

## 背景：为什么需要 Swarm？

V0.16 代号 **"The Surface Release"**，规模不小：874 commits、542 PRs、1,962 个文件、+205,216 行代码。这版发布的表面亮点是 Electron 原生应用、Web 管理面板、OAuth 远程网关、简体中文 UI、后悔机制（`/undo`）——都是让 Hermes「触达更多地方」的工作。

但真正影响架构未来的改动，藏在多智能体协作层。

此前的 `delegate_task` 是一个典型的 RPC 模式：父任务 fork 一个子智能体，等待返回，结果通过上下文压缩传回。问题在于：

- 子任务是**匿名**的——没有持久化身份
- 任务是**一次性的**——失败即失败，无法从断点恢复
- 人类是**旁观者**——无法在执行过程中插入干预
- 协作是**层级**的——只有调用方和被调用方两个角色

Kanban Swarm 解决的是同一类问题，但换了一套世界观：**把任务协作变成一张任何人都能看见和编辑的持久化看板**。

---

## 核心：一张表定义了整个协作拓扑

Kanban Board 的核心是 `tasks` 表，其中几列定义了协作语义：

```sql
status     -- triage | todo | ready | running | blocked | done | archived
assignee   -- 具名 Profile，不是匿名进程
parent_id  -- 依赖关系，通过 task_links 表实现
body       -- 任务描述，worker 被重新生成时完整读取
comments   -- 结构化数据传递通道，blackboard 协议载体
```

这意味着：

- 任何一个 Profile（或者人类）随时可以 `SELECT * FROM tasks`，看到整个舰队的实时状态
- 任何一个 Profile 可以 `UPDATE tasks SET status='blocked' WHERE id=?`，中断某个节点
- 任何一个 Profile 可以往任务里插 comment，传递结构化数据给下游

**没有共享内存，没有进程间通信，只有 SQLite 行。**

这个设计的精妙之处在于：它把「协调」这件事从代码层搬到了数据层。协调逻辑变成了一组对表的操作，而不是一堆锁和队列。

---

## Swarm 拓扑：279 行代码的任务图

`hermes kanban swarm` 命令生成如下结构：

```bash
hermes kanban swarm "Design a multi-region failover plan" \
  --workers researcher,architect,sre \
  --verifier reviewer --synthesizer writer
```

生成以下拓扑：

```
Swarm Root (共享 blackboard，已完成)
    ├── Worker 1 (parallel, researcher)
    ├── Worker 2 (parallel, architect)
    └── Worker 3 (parallel, sre)
            ↓ 全部完成后
        Verifier (gated: 等待所有 worker 完成)
            ↓ gate: pass
        Synthesizer (写入最终输出)
            ↓ gate: fail
        Root (blocked with reason，等待人工介入)
```

**Root** 是共享上下文中枢，实际上就是一张带有特殊 comment 的任务卡——结构化 JSON 写入 comment 列，所有 worker 通过读取这张卡的评论线程来获取共享上下文。这就是所谓的 **Blackboard 协议**，整个 Swarm 模块只有 **279 行 Python**。

文档原文特别强调了这一点：

> "This module intentionally does not introduce a second scheduler. It writes a small task graph into the existing Kanban kernel."

**没有第二个调度器**。Swarm 把任务图写入 Kanban 内核，Gateway 内的 dispatcher 天然就能处理这个拓扑——因为它本来就能扫描 `ready` 任务、检测依赖晋升、处理 block/unblock。

**N 个 Worker** 并行启动，各自独立工作，结果写入 blackboard（往 Root 的 comment 里 append JSON）。

**Verifier** 是一个特殊的 gated 节点——它等待所有 worker 完成后才启动，执行质量/一致性检查。通过则放行 synthesizer，失败则 block root 并附上失败原因。

**Synthesizer** 汇总所有产物，写入最终交付物，通过 MCP 工具发布。

---

## 调度器：每 60 秒一轮的长寿循环

Dispatcher 运行在 Gateway 内部（`kanban.dispatch_in_gateway: true`），每 60 秒醒来一次，执行：

```python
# 伪代码，描述 dispatcher tick 的核心逻辑
def dispatch_tick():
    # 1. 扫描 ready 任务，原子认领
    for task in db.select("status='ready'"):
        if task.claim():  # UPDATE ... SET status='running'
            spawn_worker(task)

    # 2. 检测崩溃：PID 消失但 TTL 未到期
    for task in db.select("status='running'"):
        if not process_alive(task.pid):
            db.update(task.id, status='ready')  # 回收重跑

    # 3. 晋升依赖：所有 parent done → child ready
    for child in db.select("status='todo'"):
        if all(p.status == 'done' for p in child.parents):
            db.update(child.id, status='ready')
```

配置文件中的关键参数：

```yaml
kanban:
  dispatch_in_gateway: true        # 默认，调度器在 gateway 内
  dispatch_interval_seconds: 60    # 轮询间隔
  max_in_progress: 2               # 全局并发上限
  max_in_progress_per_profile: 1   # 单 Profile 并发上限
  auto_promote_children: true      # 分解产生的子任务自动晋升
  failure_limit: 2                 # 连续失败次数上限，超限自动 block
```

这个配置意味着：如果一个 writer 任务连续失败两次，任务自动被 block 并附上失败原因，等人类介入。这解决了匿名子智能体时代「失败即丢失」的问题。

---

## Workspace 的三种形态

每个任务有一个 workspace，定义了工作文件的生命周期：

| 类型 | 路径 | 保留？ | 典型用途 |
|---|---|---|---|
| `scratch`（默认） | `~/.hermes/kanban/workspaces/<id>/` | **否**（任务结束删除） | 一次性探索任务 |
| `dir:<path>` | 绝对路径 | 是 | 共享项目目录 |
| `worktree` | `.worktrees/<id>/` | 是 | 需要 git 分支的工程任务 |

注意 `dir:<path>` 要求**绝对路径**——相对路径在 CLI 层就被拒绝，防止 confused-deputy 攻击。Dispatch 首次创建 scratch workspace 时会发出警告并产生 `tip_scratch_workspace` 事件，提醒开发者 ephemeral by design。

工作空间保留的文件在任务完成后仍然存在，可以被后续任务 pickup。这对于需要跨阶段共享状态的流水线特别有用。

---

## 人类在环：随时可以介入

这是 Kanban Swarm 相比 `delegate_task` 最重要的体验差异：

```bash
# 任何时刻插入上下文
/hermes kanban comment t_12345 "第三阶段的方向调整：优先处理 X"

# 人工决策点
/hermes kanban block t_12345 "需要人工决策：选方案 A 还是 B"

# 绕过执行锁直接操作 board
/hermes kanban unblock t_12345
```

- 任意时刻可对任务添加 comment，worker 重新生成时完整读取
- 可手动 block/unblock 任务
- `/kanban` 命令在 agent 运行期间**绕过执行锁**，直接操作 board
- 可通过 Dashboard 拖拽、查看实时状态

这意味着人类不是旁观者，而是协作网络的节点之一。Agent 执行到一半，人类可以插入 context，改变执行路径。

---

## 崩溃恢复：上下文丢失的问题

Worker 崩溃后的上下文丢失是当前已知的主要局限。替代者只收到任务描述（body），之前的工作成果——如果没来得及写入 blackboard——就丢失了。

缓解措施：Reviewer（Verifier）可以配置为强制验证产物存在性，但这不能恢复已丢失的中间状态。文档建议将**关键中间结果及时写入 blackboard**，减少单点崩溃的损失。

成本方面也需要注意：每个 Worker 是完整 Hermes 进程，5 workers = 5x token 消耗。这是架构上的取舍，没有银弹。

---

## 对比：`delegate_task` vs Kanban Swarm

| | `delegate_task` | Kanban Swarm |
|---|---|---|
| 协调形状 | RPC 调用（fork → join） | 持久化消息队列 + 状态机 |
| 父任务行为 | 阻塞直到子任务返回 | `create` 后即发出即忘 |
| 子身份 | 匿名子智能体 | 具名 Profile + 持久化内存 |
| 可恢复性 | 无——失败即失败 | Block → Unblock → 重跑 |
| 人类介入 | 不支持 | 随时 comment / unblock |
| 审计追踪 | 上下文压缩后丢失 | SQLite 行持久化，永远可查 |
| 调度器 | 调用方自身 | 独立 60s 循环 |
| 节点数量 | 1 个父 + 1 个子 | N workers + verifier + synthesizer |

**一句话区分**：`delegate_task` 是函数调用；Kanban 是工作队列，每个交接都是一行任何 Profile（或人类）都能看到和编辑的记录。

---

## 使用场景

Kanban Swarm 适合以下场景：

- **研究流水线**：并行 researcher + analyst + writer，Verifier 把关，人类在环
- **定时运维**：每日简报构建数周日志的循环任务，持久化状态跨次运行
- **数字孪生**：持久化具名助手（`inbox-triage`、`ops-review`）累积记忆
- **工程流水线**：分解 → 并行 worktree 实现 → review → 迭代 → PR
- **舰队管理**：一个专家 Profile 管理 N 个独立对象（50 社交账号、12 监控服务）

---

## 小结

Kanban Swarm 的核心洞察是：**多智能体协作不需要新的基础设施，只需要把「协调」变成数据库的一行记录**。279 行 Python 把任务图写入了 Kanban 内核，调度器天然就能处理拓扑、依赖、恢复和人类干预。

这不是在重新发明轮子，而是在现有基础设施上，用最少的代码实现了最有价值的协作语义。

V0.16 把这个能力放到了台面上。接下来的问题是：你能用它构建什么？

---

*参考：[Kanban 功能文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban) · [Kanban 教程](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial) · [V0.16.0 发布说明](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.6.5) · [Swarm 架构深度分析](https://magnus919.com/2026/05/the-smartest-agent-orchestration-framework-doesnt-have-a-scheduler/)*
