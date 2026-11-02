# vLLM-Ascend Developer Skill 九轮改进计划

> 本文档是仅保存在当前开发 workspace 的调优记录，不属于 Plugin 内容，也不得提交或上传。开始每一轮前先读取本文档；完成代码、测试或真实环境验收后同步更新状态和决策记录。

## 1. 状态说明

- `已完成`：代码与约定已经落地，并完成该轮要求的验证。
- `待远程验收`：本地单元测试和静态验证通过，但还没有在真实 SSH、Docker、NPU 环境完成验证。
- `修复中`：真实环境验收已发现问题，必须修复并重新验收。
- `部分完成`：后续轮次的一部分能力因前置需求已经提前实现，但该轮的核心目标尚未完成。
- `待开始`：尚未进入实现。
- `待决策`：需要真实日志或用户确认后才能安全确定方案。

当前总进度：第 0、1、2、3 轮已完成；第 4 轮的 4A 已通过真实环境验收，4B 源码 PYTHONPATH 注入已完成本地实现与回归，等待真实环境验收。

| 轮次 | 主题 | 当前状态 |
| --- | --- | --- |
| 第 0 轮 | 冻结基线 | 已完成 |
| 第 1 轮 | Plugin 骨架与 Marketplace | 已完成 |
| 第 2 轮 | P0 文件与远程 workspace 安全 | 已完成 |
| 第 3 轮 | P0 服务停止与执行域 | 已完成（standalone 最小方案） |
| 第 4 轮 | P0 Python path 与环境 preflight | 4B 待远程验收 |
| 第 5 轮 | P1 状态机与 workflow runner | 待开始 |
| 第 6 轮 | P1 测试与 verifier 自动化 | 待开始 |
| 第 7 轮 | P1 安全修复循环 | 待开始 |
| 第 8 轮 | P2 Skill 精简与专业化 | 待开始 |
| 第 9 轮 | P2 并行分析与发布 | 待开始 |

## 2. 已确认的全局设计决策

以下决策覆盖原计划中与实际使用反馈冲突的内容：

1. 运行目录统一为项目内的 `.dev/`，不再使用 `.vllm-ascend/`，避免与真实 vLLM-Ascend 源码目录混淆。
2. 一个本地 workspace 共享一个 `.dev/run/`。不按 Claude 会话创建 run ID；`generated/`、`downloads/`、`logs/`、`records/` 跨会话连续使用。
3. `bootstrap` 只在 workspace 首次使用或目录缺失时初始化；后续调用只检查并补齐结构，不重建新的 run。
4. `.dev/runtime/` 只保存 SSH daemon 等机器运行状态；诊断结果属于 `.dev/run/`。
5. 架构保持“本地控制、远程执行”。真实源码检查、修改、启动和调试可以直接在远程宿主机或容器执行，不强制先下载源码到本地。
6. `ssh_utils.py exec` 是明确的宿主机操作；`docker-exec` 是明确的容器操作；`eval` 是节点引用，不再为此新增 `host_exec`、`container_exec`、`eval_exec` 三套重复入口。
7. `exec` 默认使用节点 `work_dir`；`docker-exec` 默认使用 `docker.work_dir`；相对远程传输路径以宿主机 `work_dir` 为基准。绝对路径和显式 `cd` 保持可用。
8. 配置中的 `env_vars` 由 `docker-exec` 注入。启动脚本内部再次 `export` 同名变量时，以脚本内设置为准。
9. 真实证据已确认：单独在宿主机按端口停服务会漏掉不监听端口的 EngineCore。解决时优先复用已经远程测试通过的 `setsid + pid file + 容器清理 + 宿主机端口复查` 命令，不先建设通用生命周期框架。
10. 优先遵循 Keep Simple and Available：不增加尚无真实需求支撑的命令层、会话绑定层或本地源码中转层。

## 3. 分轮计划

### 第 0 轮：冻结基线

状态：`已完成`

目标：建立可比较基线，不改变既有运行行为。

已落地：

- 保存文件结构和典型旧运行轨迹。
- 建立 mock SSH、服务日志、HTTP 响应和 aisbench fixtures。
- 固化 standalone、PD-separated、aisbench 必须保留的能力。
- 为 workspace 越界、Python path 不一致、kill 执行域错误建立 known-gap fixtures。
- 清理并忽略 `__pycache__`、旧 curl 脚本和本地运行产物。

验收基线：

- fixtures 可重复解析。
- 三类已知问题可由测试稳定表达。
- 后续修改不会静默删除三种部署/评测能力。

### 第 1 轮：Plugin 骨架与 Marketplace

状态：`已完成`

目标：规范 Plugin 包装，不改变远程业务逻辑，并支持远程直接安装。

已落地：

- 增加 `.claude-plugin/plugin.json`。
- 主入口迁移到 `skills/diagnose/SKILL.md`。
- 建立 `hooks/`、`scripts/`、`tests/` 结构；`schemas/` 等到状态或事件 schema 真正落地时再创建，不保留无内容目录。
- 根 `SKILL.md` 暂时作为兼容入口。
- 增加 `.claude-plugin/marketplace.json`，支持 `/plugin marketplace add` 安装方式。
- 完成真实远程安装测试。

持续验收：

- `claude plugin validate` 通过。
- `claude --plugin-dir ./` 与 `/reload-plugins` 可用于本地开发。
- Marketplace 添加、安装和命令发现正常。

### 第 2 轮：P0 文件与远程 workspace 安全

状态：`已完成`

目标：同时解决本地 workspace 外写入和远程工作目录漂移，不限制正常远程开发。

#### P0-A：本地文件写入安全

状态：`已完成，远程测试通过`

已落地：

- 项目运行目录改为共享的 `.dev/run/`。
- `path_policy.py` 负责 bootstrap、配置模板复制和本地写入边界。
- Skill-scoped `PreToolUse` Hook 检查 Write、Edit 和 Bash 的本地写入目标；没有重复实现 Claude Code Hook 系统本身。
- Plugin 安装/缓存目录保持只读。
- 本地下载、生成文件、日志和记录分别进入固定子目录。
- `.dev/config/`、`.dev/runtime/`、`.dev/run/` 全部由项目 `.dev/.gitignore` 隔离。
- 配置初始化改为脚本完成，避免让模型手工判断和复制模板。
- upload/download 成功时返回 `exit_code: 0`。

验收结果：

- workspace 外 Write/Edit 被阻止。
- Bash 本地重定向绕过被阻止。
- 下载到 workspace 外被拒绝。
- Plugin 源目录不会接收运行结果。
- 多会话共享同一 `.dev/run/`，不因 run ID 丢失过程材料。

#### P0-B：远程 workspace 安全

状态：`已完成，真实环境验收通过`

已落地：

- `exec` 自动从节点宿主机 `work_dir` 执行。
- `docker-exec` 自动从 `docker.work_dir` 执行，并注入节点 `env_vars`。
- upload/download 的相对远程路径相对于宿主机 `work_dir` 解析。
- `wait` 支持显式 `--scope host|container`；默认 host 保持兼容，容器日志在文档中显式指定 container。
- 每个相关响应携带 `node_ref`、`scope`、`cwd`。
- 保留绝对路径和显式 `cd`；允许直接操作远程 `vllm_ascend_source`。

真实环境发现的问题：

- SFTP 将 curl 脚本上传到节点 `work_dir`，该目录按现有部署约定也是 `docker.startup_script` 所在的宿主机/容器共享脚本目录。
- 测试流程却从不同的 `docker.work_dir` 执行 curl 脚本。日志只证明 `docker.work_dir/curl_test.sh` 不存在，不能据此判断共享脚本目录没有挂载。
- 修复应优先在容器内执行上传后的节点 `work_dir/curl_test.sh`，并在执行前检查该绝对路径；不应默认引入 `docker cp` 或让模型手写 curl。
- 已同步修正 test-runner、verifier 和 precision workflow，使上传与执行都使用节点 `work_dir/curl_test.sh`。
- 远程命令退出码非 0 时仍返回 `success: true`，会放大 Agent 对失败结果的误判。
- 修复必须保持 `test.yaml` 为 prompt 唯一来源，禁止回退到模型手工拼接 curl。

真实环境验收：

- standalone、PD 节点和 eval 节点的宿主机 `pwd` 与配置一致。
- 容器 `pwd` 与各节点 `docker.work_dir` 一致。
- 相对 upload/download 落在节点宿主机 `work_dir`。
- host/container 两种 `wait` 都能读取正确作用域的日志。
- 绝对源码路径和显式 `cd` 不被误拦截。
- 响应上下文与实际节点、作用域、cwd 一致。

### 第 3 轮：P0 服务停止与执行域

状态：`已完成（standalone 最小方案已通过真实环境验收）`

目标：解决“停止 API Server 后 NPU 上仍残留 EngineCore”等宿主机/容器生命周期混淆，同时避免误杀其他服务。

回滚与结论：

- 提交 `0c866f7` 在真实环境中因巨大内联 Shell 命令匹配自身，导致 `service-start/status/stop` 都误报 `untracked-vllm-process`。
- 启动在 `setsid` 和 pid file 写入前已退出，因此“无 pid file”是启动误判的下游结果。
- 错误结果隐藏匹配证据，引导 Agent 反复读取 `ssh_utils.py`，严重污染上下文。
- 该提交已完整反向应用，15 个相关文件与父提交 `3d83433` 完全一致；回滚后 54 项测试通过，1 项跳过，3 项预期失败。
- 下次只围绕用户已真实验证通过的三条命令做最小封装/文档替换，不新增通用状态探测、所有权匹配、daemon 改名或大范围 Hook 规则。

当前最小实现：

- 不修改 `ssh_utils.py`，不新增 lifecycle action。
- standalone 使用 `setsid` 启动，将 `$!` 写入 `{docker.work_dir}/vllm.pid`。
- 停止时先在容器内用负 PID 杀进程组并删除 PID 文件，再在宿主机执行已有端口清理与验证。
- PD 分离暂不套用未经真实环境验证的 PID 规则。

### 第 4 轮：P0 Python path 与环境 preflight

状态：`4A 已通过真实环境验收；4B 已通过本地回归，待真实环境验收`

目标：在启动前发现错误源码、错误包版本和 host/container 路径不一致，避免服务启动后才暴露导入错误。

已提前完成：

- 节点 `env_vars` 会被 `docker-exec` 实际消费，而不再只是文档字段。
- 启动脚本内同名 `export` 可以覆盖配置注入值。
- host 与 container 已有独立 cwd 配置。

4A 当前实现：

- 新增只读 `scripts/preflight.py`，复用现有 `docker_exec_command`，不新增 SSH 执行通道。
- 每个节点一次性输出容器 Python、版本、`PYTHONPATH`、`sys.path`、两个包的 `__file__`、版本与导入错误。
- 导入失败时阻止启动；源码路径不匹配在 4A 只保留证据，不自动改环境。
- 移除工作流中容易误跑在 Claude 本地的 `pip show` 和 `ls` 检查。
- 不安装依赖、不写远程文件、不修改现有服务启动命令。

4B 当前实现：

- `docker-exec` 新增显式 `--source-pythonpath` 开关；默认关闭，普通容器命令行为不变。
- 开关开启时，把容器内 vLLM、vLLM-Ascend 源码目录置于原有 `PYTHONPATH` 前方。
- 容器源码路径可选独立配置，未填写时复用现有 `vllm_source` / `vllm_ascend_source`。
- preflight 与所有服务启动模板显式开启该开关；停止、日志、健康检查不启用。
- 注入发生在执行启动脚本之前，启动脚本内部同名设置仍然具有最终优先级。
- 不修改第三轮的 `setsid`、PID 文件或停止流程。

计划改动：

- 明确定义宿主机源码路径与容器内源码路径，支持挂载后路径不同。
- 在 preflight 中检查 vLLM、vLLM-Ascend 源码路径及预期 mount 映射。
- 使用容器实际 Python 输出 `vllm.__file__`、`vllm_ascend.__file__` 和必要版本信息。
- 根据实际目录构造服务启动所需 `PYTHONPATH`，并记录配置值、脚本覆盖值和最终有效值。
- 在启动前阻止明显的源码/site-packages 冲突和不可导入状态。
- 不自动 `pip install` 或重装 vLLM/vLLM-Ascend。

验收：

- 源码与 site-packages 冲突时能检测并给出证据。
- host/container 路径不同时仍能加载预期源码。
- 启动脚本覆盖 `ASCEND_RT_VISIBLE_DEVICES` 等变量时，最终值符合脚本优先规则。
- 可预判的导入错误在正式启动前被阻止。

### 第 5 轮：P1 状态机与 workflow runner

状态：`待开始`

目标：用最小 runner 替换脆弱的 Markdown 人工编排，并保持 workspace 级跨会话续作。

计划改动：

- 先定义最小状态集合、转换表和每个转换的守卫，不先设计通用工作流框架。
- 状态存放在共享 `.dev/run/state.json`；事件追加到 `.dev/run/records/events.jsonl`，不创建会话级 run ID。
- 原子写入状态，记录当前节点、host/container scope、cwd、配置摘要和最近一步证据。
- 实现 `plan`、`run`、`resume`、`pause`、`abort`、`status`。
- 加 workspace/node 粒度锁，防止同一节点同时运行两个相冲突的诊断步骤。
- 实现幂等重试和最大迭代限制。
- 将健康检查设为测试和评测前的强制状态守卫。
- `resume` 首先读取共享记录，跨 Claude 会话继续同一任务。

验收：

- 进程或会话中断后可以从持久状态恢复。
- 服务未健康时不能进入测试状态。
- 重复 `resume` 不会重复启动、重复应用修改或覆盖记录。
- 非法状态转换被拒绝并留下事件。
- 不引入会话绑定和 run ID 管理负担。

### 第 6 轮：P1 测试与 verifier 自动化

状态：`待开始`

目标：让测试、落盘、验证和报告形成确定性的完整闭环。

计划改动：

- 由脚本遍历 `test.yaml` 中全部 tests 和 prompts。
- 支持当前配置中的全部请求参数。
- 使用代码直接发送 HTTP 请求并写入 JSONL；生成 curl 脚本暂留为兼容和人工复现入口，待真实验证后再决定删除。
- 实现 exact、contains、regex verifier。
- 实现乱码、异常字符和复读检测。
- 结构化解析 aisbench 结果。
- runner 消费结构化结果，Markdown 只保留选择规则和异常处理说明。

验收：

- 同一 prompt 在不同配置间保持一致。
- 每个请求、响应、耗时、错误和验证结果都可追踪。
- verifier 的成功与失败 fixtures 全部通过。
- aisbench 结果可进入统一报告和状态判断。

### 第 7 轮：P1 安全修复循环

状态：`待开始`

目标：规范远程源码修改、记录和回滚，但不强制本地具备源码或运行环境。

计划改动：

- 分离日志收集、问题分析、变更提案、应用、重启和验证状态。
- 允许直接检查和修改远程 `vllm_ascend_source`；不要求先把完整源码下载到 `.dev/run/generated/`。
- 应用前记录目标文件、原始 hash、修改理由、diff 或等价变更证据、备份/回滚方式。
- vLLM 上游路径保持强制只读。
- vLLM-Ascend 自动应用变更前保留明确批准边界；开始本轮前再次确认批准粒度，避免阻碍已经明确授权的连续调试。
- Python 修改与 `csrc/` 修改使用不同验证流程；只有 `csrc/` 变更触发编译/安装要求。
- 每轮修改→重启→复测生成 `.dev/run/records/fix_N.md`。

验收：

- 未满足批准策略的变更不能应用。
- vLLM 上游修改始终被拒绝。
- 远程变更可定位、可审计、可回滚。
- 不依赖本地 Python/NPU 环境完成远程修改。
- 应用后必须完成符合变更类型的重启和验证。

### 第 8 轮：P2 Skill 精简与专业化

状态：`待开始`

目标：降低中文使用场景的上下文成本，提高可维护性。

计划改动：

- 重构主 `skills/diagnose/SKILL.md`，只保留状态机入口、关键守卫、资源路由和必须遵守的安全边界。
- 将详细模块说明迁移到 `references/`，按需加载。
- 将 test-runner、verifier、service 中已确定的逻辑迁移到脚本/runner。
- 案例文档迁移到 `references/cases/`。
- 删除 `SKILL.md`、`CLAUDE.md`、`AGENTS.md` 之间的重复规则，建立单一事实来源。
- `CLAUDE.md` 只说明 Plugin 开发仓库约定，不承担安装后的运行规则。
- 保持用户交互、错误消息、操作提示以中文为主，命令和结构化字段保持稳定英文标识。

验收：

- 主 Skill 明显缩短，核心约束不丢失。
- 新会话能按需找到正确 reference，不必一次加载全部文档。
- 同一规则只维护一份。
- 中文用户可理解初始化、远程执行、失败原因和下一步操作。

### 第 9 轮：P2 并行分析与发布

状态：`待开始`

目标：在核心流程稳定后增加可选分析能力并正式发布。

计划改动：

- 评估是否需要 Claude Code 官方 Dynamic Workflow；只用于多日志只读并行分析，不进入核心执行链。
- 可选增加只读 `log-analyzer`、`patch-reviewer` agents。
- 在小规模真实 NPU 环境执行 canary。
- 使用全新会话 forward-test；只提供原始任务和工件，不泄露预期结论、已知缺陷或修复答案。
- 执行 `claude plugin validate`、`--plugin-dir` 测试、Marketplace 安装测试。
- 完成内部 Plugin 安装、版本号和发布说明。

验收：

- standalone、PD-separated、aisbench 三条主链路通过 canary。
- 新会话能够独立初始化、续作、诊断和产出记录。
- 可选 agent 只能读日志或评审 patch，不能绕过主 runner 修改远程环境。
- Plugin validator、本地加载和 Marketplace 安装均通过。

## 4. 每轮固定“集训”循环

每轮严格执行以下顺序：

1. 只选择一个主要行为问题。
2. 先补充能复现问题的失败 fixture。
3. 修改最少数量的文件；不跨轮同时重写大量提示词和执行代码。
4. 运行单元测试、静态验证和 Plugin 结构检查。
5. 在 mock 环境执行该轮完整场景。
6. 由维护者提供真实环境必测命令和预期结果。
7. 用户在真实 SSH、Docker、NPU 环境测试并返回日志；本地无真实环境不是跳过该步骤的理由。
8. 使用全新 Claude 会话执行真实用户式 prompt，检查工具调用、文件产物和状态转换。
9. 通过后更新本文档状态，再进入下一轮。

固定回归任务：

- Python 路径不一致。
- 端口被占用，且 API Server 与 EngineCore 生命周期不一致。
- Agent 尝试在本地 workspace 或预期远程 work_dir 外创建调试文件。
- 服务启动失败 → 分析 → 修改提案 → 批准 → 重启 → 验证。

## 5. 决策与验收记录

| 日期 | 轮次 | 记录 |
| --- | --- | --- |
| 2026-08-03 | 总计划 | 根据原九轮计划和已完成改动建立本地持续计划文档。 |
| 2026-08-03 | 第 2 轮 | `.vllm-ascend/runs/` 调整为 workspace 共享的 `.dev/run/`；取消会话级 run ID。 |
| 2026-08-03 | 第 2 轮 | P0-A 已通过真实环境测试；P0-B 已完成代码和本地测试，等待真实环境验收。 |
| 2026-08-03 | 第 2 轮 | P0-B 真实测试发现 curl 上传目标为节点 `work_dir`，执行目标却错误使用 `docker.work_dir`；应先按与 `startup_script` 相同的共享目录路径执行。同时发现非零退出码仍返回 success 的响应语义问题。 |
| 2026-08-03 | 第 2 轮 | 已修复 curl 上传/执行路径不一致和非零退出码仍返回 success 的问题，并增加回归测试；等待真实环境重新验收。 |
| 2026-08-03 | 第 3 轮 | 执行域基础提前在第 2 轮落地；kill/EngineCore 处理推迟到取得真实日志后，不再以“容器 kill 必须失败”为前提。 |
| 2026-08-03 | 第 2 轮 | P0-B 路径与退出码补丁已通过真实环境验收，第 2 轮完成。 |
| 2026-08-03 | 第 3 轮 | 真实环境确认宿主机 `fuser` 只能停止端口监听进程，EngineCore/worker 可能残留；开始评估容器内 PID/PGID 记录与 TERM→KILL 分级停止方案。 |
| 2026-08-03 | 第 4 轮 | `env_vars` 注入能力提前落地；完整 PYTHONPATH preflight 仍属于第 4 轮。 |
| 2026-08-05 | 第 3 轮 | 大规模生命周期实现真实验收失败：命令自匹配导致无法启动、pid file 未执行、Agent 循环读取源码。已完整回滚到 `3d83433`，后续改用已真实验证命令的最小修改方案。 |
| 2026-08-05 | 第 3 轮 | 按已验证命令完成 standalone 最小方案：`setsid` 启动、`vllm.pid`、容器内负 PID kill、宿主机 fuser fallback；未修改 ssh_utils，等待真实环境验收。 |
| 2026-08-07 | 第 3 轮 | standalone 最小 PID/进程组方案通过真实环境验收，第 3 轮完成；PD 分离不扩展未经验证的 PID 规则。 |
| 2026-08-07 | 第 4 轮 | 启动 4A 只读 preflight：新增固定脚本与工作流入口，本地 61 项测试通过（1 项跳过、3 项预期失败），等待真实环境验收后再进入 4B。 |
| 2026-08-07 | 第 4 轮 | 4A 真实结果成功识别两个源码目录存在但实际从 site-packages 导入，验证通过；开始 4B。 |
| 2026-08-07 | 第 4 轮 | 4B 新增显式源码 PYTHONPATH 注入开关并接入 preflight/启动模板，本地 65 项测试通过（1 项跳过、2 项预期失败），等待真实环境验收。 |

