# Superagent 演示指南

> 本指南用这次演示，说明 dsh-superagent 如何把一条 GitHub 信号变成可评审的代码改动。演示的输入是合成的提交状态，不是真实的 CI 测试失败。

## 它能做什么

dsh-superagent 是运行在 DeepSeek Harness 里的主动型编码智能体，面向一个 GitHub 仓库。它监听 GitHub（也可以接入飞书），在收到明确请求，或命中团队事先写好的规则时，创建一个工单，把工单交给独立的执行器 Session 和目录完成，再通过原对话或 Pull Request 交付结果。

下面会反复出现几个词：Source（来源）是监听某个平台事件的部分；Trigger（触发）是一条来源事件；Thread（线程）是同一处按时间排列的触发；Eval（评估）判断要不要接单；Policy（策略）是团队写下的接单规则；Ticket（工单）是一次被授权的工作；Worker（执行器）是实际完成工作的独立 Session；Pull Request（PR）是把改动交给评审的请求。

这次演示里，智能体只做一件事：新增 `docs/DEMO.zh-CN.md`，在 `agent/<工单短名>` 分支上提交，推送到配置的 fork，并向该 fork 的 `main` 开一个 PR。它不修改其他文件，也不向 `deepseek-ai` 仓库开 PR。

演示的起点是一条**合成的 GitHub 提交状态**。演示者用本次智能体所用的同一个 GitHub 账号，在 `main` 的最新提交上写入 `context=superagent-demo/chinese-guide`、`state=failure` 的提交状态；CI 触发没有人的发送者，因此同一个账号也能发布它。仓库里没有真实的测试失败，也没有代码损坏。对 GitHub Source 来说，check run（检查运行）和 commit status（提交状态）都算 CI；它把“默认分支的失败集合发生变化”转成一条 CI Trigger。演示规则只接受这个精确的失败上下文，通过、恢复和其他事件都会被忽略。

## 一条任务如何完成

### 1. 触发（Trigger）

演示者用同一个账号写入一条提交状态，大致如下：

```sh
gh api --method POST \
  repos/Fangjj0621/dsh-superagent-prototype/statuses/$(git rev-parse HEAD) \
  -f state=failure \
  -f context=superagent-demo/chinese-guide \
  -f description='superagent demo, not a real test failure'
```

GitHub Source 读取默认分支的 check run 和 commit status。第一次观察只建立失败集合的基线；之后这个集合发生变化时，Source 才产生 Trigger，其线程为 `github:ci:main`。这条 Trigger 的文本类似 `CI failed 1/1 on main at <sha>: superagent-demo/chinese-guide`，并附带一个读取全部检查的链接。同理，如果演示环境刚刚启动，需要先让它完成一次基线观察，再写入失败状态。

### 2. 评估（Eval）

Trigger 先过 Gate（准入：确定发起者是否有资格、以多强的依据发起），再做 Assessment（研判：模型判断是否命中规则、是否接单）。CI 结果没有人类发送者，因此是否成为候选由 Policy 中“默认分支的 CI”这一接收行决定。随后，评估逐条检查已启用的规则。

本次命中的规则是“演示：中文使用指南”（`demo-chinese-guide`）：默认分支 CI 失败，且失败的上下文恰好是 `superagent-demo/chinese-guide`。通过、恢复和其他 CI 上下文都不匹配。规则命中后，Eval 接受这次 Trigger，并创建一个带有 Mandate（授权：这次工作被允许的范围和依据）的工单；它记录策略版本、范围、强度、命中的规则和请求来源。没有匹配规则时不会主动接单；在默认的 `observe` 模式下，这类候选只会被记录为“本会接单”。

### 3. 工单（Ticket）

一次接受（accept）会创建一个 Ticket。这个 Ticket 的 Mandate 把工作限制在“只新增 `docs/DEMO.zh-CN.md`，不修改其他文件”的范围内。它还订阅触发它的线程 `github:ci:main`，新的输入会交给执行器。工单自行维护状态：排队、执行中、等待人工评审、结束；面板的 Execution 页可以查看它和它的 Assignment。

### 4. 执行器（Worker）

派工（Assignment）启动一个独立的 Executor Session，名为 `worker-<ticket id>`，并给它一个以工单短名命名的目录；目录默认位于执行根目录（默认 `<stateDir>/work`）下。prepare 命令在这个目录里创建 git worktree 和分支 `agent/<工单短名>`，所以不同工单的 Session 和文件互不干扰。

Worker 读取仓库文档、只修改 `docs/DEMO.zh-CN.md`、运行相关检查并自审，然后以智能体账号提交，推送到配置的 fork，并向该 fork 的 `main` 开一个 PR。主 Session 只负责路由和转交报告，不替 Worker 做技术决定；Worker 需要人决定时，通过结构化报告把一个问题交回原对话。

### 5. 拉取请求（Pull Request）

PR 分支的最后一段与工单短名一致，Ticket 因此把 PR 关联到自己，并订阅它的线程。Worker 报告 `ready` 后，工单进入“等待人工评审”。本演示的完成标准是：文档通过必要检查，PR 已创建，工单等待人工评审；不需要合并。默认情况下，智能体不会以智能体账号启用自动合并，是否合并由人决定，智能体不代替人合并。

### 订阅与人工评审

订阅是 Ticket 与 Thread 之间的多对多关系。Ticket 订阅触发它的线程，以及它打开的 PR 的线程；这些线程上的后续消息会作为上下文送到同一个 Ticket，不需要重新 @ 智能体。订阅本身不会为人的跟进新开一次 Eval：人的跟进由该 Ticket 的 Worker 判断，在范围内处理，超出范围交回主 Session 路由；订阅也不授予新权限，Worker 仍只能在原 Mandate 的范围内工作。机器人消息仍可独立进入 Eval，被当作新的候选评估。

人的评审意见正是通过 PR 线程到达 Worker 的：范围内的修改由 Worker 处理；超出工单范围的请求会交回主 Session 重新路由。如果 PR 后来合并或关闭，Ticket 结束但保留订阅；之后有人在该线程留言，可以按原 Mandate 恢复同一个 Ticket（被明确停止的除外），而 CI 和机器人消息不会唤醒已经结束的 Ticket。

## 当前原型的限制

- **这是研究原型，不是生产软件。** 代码可能仍有缺陷，工程实践也可能不一致；在用于自己的环境前需要自行评审和适配。
- **演示的 CI 失败是合成的。** 它验证的是“提交状态变化 → 工单 → 独立执行 → PR”的链路，不验证真实测试失败的检出或修复能力。原型不提供一键演示，这个提交状态需要演示者预先写入；这次改动也只是一份文档，不代表复杂代码修改能力。
- **人通过 Issue 或评论下任务时，任务来源身份和智能体身份必须不同。** Source 以智能体账号的凭据读取通知，并丢弃由该账号自己发起的 Issue、PR 和评论；如果同一个人既发起这类任务又充当智能体，请求会被当作智能体自己的消息而无法成为 Trigger。智能体应使用专用账号（不能是 GitHub App，因为 Source 需要读取该账号的通知），并通过只属于它的 `gh` 包装脚本行事。CI 触发没有人的发送者，不受这条身份过滤影响：本次 synthetic commit status 就是由 Fangjj0621 账号写入的，演示没有使用第二个 GitHub 账号。
- **隔离有限。** Worker 以运行者的操作系统用户身份、在 `danger-full-access` 权限下执行；`gh` 和 `lark-cli` 包装脚本只是选择默认凭据，不是安全边界，Worker 可以按绝对路径执行程序并读取运行者的 home 目录。需要真正隔离时，应使用只保存智能体凭据的专用操作系统账号。
- **平台和范围有限。** 一个 Source 只监听一个 github.com 仓库，不支持 GitHub Enterprise Server。默认分支 CI 通过轮询读取，轮询默认约一分钟一次，因此从写入状态到出现工单会有延迟；失败集合变化才会触发，第一次观察还只建立基线。演示规则本身很窄，只匹配 `superagent-demo/chinese-guide` 这个精确的失败上下文。
- **人工评审是必需的，合并不是。** 默认没有自动合并，智能体也不能以自己的账号启用自动合并；本演示以“文档通过必要检查、PR 已创建且工单等待人工评审”为完成标准，是否合并由人决定。
- **执行会受环境和故障影响。** 宿主重启、网络或模型故障可能中断当前 Assignment；报告为 `blocked` 或 `failed` 的工单会停止自动接收输入，需要人的消息、重试或恢复。Worker 的模型、并发和运行目录都由部署配置决定，不是由这次演示固定下来的。
