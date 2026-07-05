# Hermes Agent v0.18 Kanban Swarm 功能深度解析：面向 AI 开发者的多智能体协作实践

> 本文基于对 Hermes Agent v0.18 Kanban Swarm 机制的梳理与实践记录撰写，面向正准备进入 AI 应用开发领域的工程师。

---

## 1. 为什么多智能体协作需要 Kanban Swarm

在构建 AI 应用时，单个模型调用通常只解决子问题。真实工程中需要的是：
- 研究、编码、评审、集成等不同角色并发推进
- 失败重试与超时保护
- 人类可检索的任务生命周期与产物衔接收纳

Hermes Agent 的 Kanban Swarm 将这些问题封装成一套可编程、可审计、可中断的协作协议，而不是让开发者手工拼装子进程与消息队列。

---

## 2. 核心模型

### 2.1 任务卡片（Task Card）作为一等公民

每个工作单元都是一个 Kanban 卡，拥有：
- **分配对象（assignee）**：决定哪个 profile/profile 执行
- **生命周期状态**：ready → running → blocked/done
- **父母/子任务依赖图**：用 `parents=[...]` 显式表达
- **工作区语义**：scratch / dir / worktree
- **超时与重试控制**：`max_runtime_seconds` + dispatcher 自动重入

这很像 Jira + GitHub Actions 的混合体，但它是 agent-native 的：卡片的 carry-on 信息直接进入后继 agent 的上下文。

### 2.2 显式依赖替代隐式编排

最反人类直觉、也最容易出错的多 agent 设计是“隐式配合”——agent A 写文件，agent B 靠约定去读。

Kanban Swarm 用 `parents` 硬保证执行顺序：
- 父任务全部达成 `done`，子任务才进入 `ready`
- 不满足不执行，避免"抢跑读写"

用人类例子类比：这相当于你在 CI 里写 `needs: [frontend, backend]`，而不是靠口头约定。

---

## 3. 关键运行机制

### 3.1 Heartbeat 与存活保护

Dispatcher 维护任务锁；worker 通过定期 `kanban_heartbeat` 声明存活。如果长时间无心跳，任务被强制 requeue，标记为 `timed_out` 而非永久丢失。

对开发者来说这意味着：
- 长跑任务（训练、编码、爬虫）**必须**周期性 heartbeat
- 不要用"运行完再汇报"的心态写 agent loop

### 3.2 Block 与 Resume

当任务卡在外部依赖上（缺凭证、需要人类决策、第三方服务不可用），应调用 `kanban_block(reason=...)`：
- `dependency`：等待上游，自动恢复
- `needs_input`：人工介入，之后 dispatcher 重新派发
- `capability` / `transient`：硬墙 / 偶发故障

这比直接 crash 更可观测，也更容易做 SLA 告警。

### 3.3 产物交付约定

任务完成时使用 `kanban_complete`：
- `summary`：人类可读的交接语
- `metadata`：变更文件、测试次数、决定
- `artifacts`：可附件的文件路径（图表、PDF、脚本）
- `created_cards`：本次派生出的子任务 id，kernel 会校验存在性

注意：
- 产物路径在 `artifacts` 字段；`metadata.changed_files` 只是参考，不触发上传
- 创造子任务后**不要**自己在 `complete` 里继续执行下一代工作，那是下一名 worker 的职责

---

## 4. 工程取舍与实践建议

### 4.1 Orchestrator 不是万能胶水

Orchestrator 更适合做任务分发、状态聚合、中止/恢复决策，而不是在单个 orchestrator 里塞进所有业务逻辑。把重活分派给 leaf agent；orchestrator 只做路由。

### 4.2 避免子任务爆炸

每个新增子任务都会消耗调度资源、profile 锁、上下文窗口。建议：
- 3~5 个并行 worker 是合理起点
- 子任务粒度定向解决单一职责
- 使用 `parents` 表达 fan-in，不要用或隐式或隐式顺序

### 4.3 状态机要保守

任务状态只有少数字面值；不要私自扩展。遭遇未知状态时：
1. 写 `kanban_comment` 备注当前状态
2. 调用 `kanban_block` 等待人工判断

### 4.4 审计优先于优化

Kanban 的 comment 线程和 events 是天然审计日志。遇到 agent 表现不稳定时：
- 优先读取 `kanban_show` 中的 events 看崩溃模式
- 再决定是加 retry、改 timeout 还是拆任务

---

## 5. 实际故障模式速查

| 现象 | 根因 | 建议修复 |
|------|------|---------|
| 任务反复 crash，无 log | worker 崩溃在初始化 | 增加 heartbeat 间隔；确认 script 路径存在 |
| ready 任务一直不被消费 | 无可分配 profile | 检查 assignee 是否为真实 profile |
| 子任务永远不会 ready | parents 有未完成的 | 显式查看父任务状态 |
| artifacts 权限被拒 | artifact 路径不在允许目录 | 文件写入依赖工作区监管，配置 workdir 或使用允许路径 |
| 完成时 created_cards 被内核拒绝 | id 虚构或不是本轮创建 | 仅填入 `kanban_create` 返回的真实 id |

---

## 6. 与手写多线程脚本的定性区别

| 维度 | 手写 threads/processes | Kanban Swarm |
|------|----------------------|--------------|
| 任务生命周期 | 自己写锁/信号量/IPC | Kanban 状态机 + 锁 |
| 失败恢复 | try/except + 手写重试 | dispatcher requeue + heartbeat |
| 产物传递 | 约定文件路径/变量 | artifacts + metadata 结构化 |
| 人类中断 | 自己实现 handler | block/unblock 原语 |
| 审计 | 自己打日志 | comment/event 内建 |

---

## 7. 总结

Kanban Swarm 不是一个 UI 仪表盘。它是一个 **agent 协作协议 + 持久化状态机 + 调度器** 的三合一设计。

对 AI 应用工程师而言，掌握这套机制的价值在于：
- 可以把复杂工作流拆成可靠、可观测、可重试的子任务
- prompt、状态、文件三种交接媒介并存，各司其职
- 不依赖外部消息队列或数据库即可获得结构化协作能力

如果你已经在用 Hermes Agent 做日常辅助或项目集成，花时间理解 `kanban_*` 工具是你从单 agent 助手迈向多 agent 工程的最小可行路径。

---

> 作者注：本文基于 Hermes Agent 官方文档与实战调试整理；没有使用第三方 MCP 做内容检索，力求不跑题、不掺水分。如需具体命令与配置，可直接看 Hermes agent profile 目录下的 skill 实现。
