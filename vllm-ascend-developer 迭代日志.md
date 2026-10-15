# 迭代日志：

## 0轮

冻结了基线，保证后续修改不会导致能力丧失

## 1轮

配置了 Plugin 骨架结构，可以在Claude code中通过/plugin获取

## 2轮

- P0-A增加(1)skill运行时目录，控制每次任务的脚本、日志、下载和修复记录放到`.vllm-ascend/runs/run-id`避免再次生成类似 flashcomm-mtp-debug.md 的无关结果文件；(2)初始化在本地工作目录下创建`.vllm-ascend/config`，用于真实环境配置的设置、保存服务地址、密码和源码路径，避免修改plugin缓存目和把真实凭据提交到plugin仓库；（3）`.vllm-ascend/runtime/`保存daemon等运行状态。
- 增加路径检查：阻止向 Plugin 源码目录、本地 workspace 外和其他 run 写文件。保证Plugin 安装目录保持只读、干净和避免本地workspace乱跳。
- 增加 Write/Edit/Bash Hook：在 Claude 执行工具前检查路径，拦截重定向、tee、动态路径等绕过方式。
- 调整工具脚本：
  - curl_test.sh 不再生成到 Plugin 的 scripts/。
  - 下载文件只进入当前 run。
  - SSH daemon 状态不再写 /tmp/vllm-ssh-daemon。
- 增加 Git 忽略规则，避免运行结果和凭据被提交。
- 错误提示改为中文，并测试了中文和空格路径。
- 修复配置文件中env_vars读取
- 脚本增加远程默认workspace、禁止修改
- 修复脚本调用路径硬解码问题 

## 3轮

- 在skill.md修复杀进程流程命令，目前仅作了最小化修改，目前单任务测试没问题

# 迭代中遇到的问题ToDo：

- [x] **Skill 本身不具备注入 env_vars 的能力。 原因如下：**
  
  1. ssh_utils.py 的 _extract_node()（第 169 行）只提取 5 个字段：host、port、username、password、work_dir。env_vars 和 docker
     配置完全未被读取。
  2. service.md 的启动模板也没有 env_vars 占位符，启动命令就是原样 docker exec {name} bash -c '...'。
  3. 整个 ssh_utils.py 没有任何代码将 env_vars 注入到 docker exec -e 或命令前缀中。
  
  结论：service.yaml 中的 env_vars 字段目前只是文档性质的配置声明，实际并未被任何脚本消费。 要让 ASCEND_RT_VISIBLE_DEVICES=15
  生效，只能修改启动脚本 test.sh（取消注释并改为 export ASCEND_RT_VISIBLE_DEVICES=15），或手动在 docker exec 命令中加 -e 参数。
- [x] **停止服务的时候会有许多vllm残留进程**
  
  问题出在 fuser -k {port}/tcp 这个 kill 方式本身。 vLLM 是多进程架构：
  
  APIServer (持有端口 8088)
  └─ EngineCore
  └─ VLLM::Worker × N (不持有端口)
  
  fuser -k 8088/tcp 只杀持有端口 8088 的进程（APIServer），EngineCore 和 Worker 进程没有监听任何端口，所以 fuser
  根本不会碰它们。APIServer 被杀后，子进程变成孤儿被 init 收养，继续占用 NPU 显存。
  
  所以 service.md 中用 fuser -k 的方法对单进程服务有效，但对 vLLM 这种多进程服务不够用。
  
  回顾我们的清理过程也验证了这一点：
  
  - 第一次 fuser -k 8088/tcp → 只杀了宿主机的 PID 3552130，容器内 VLLM::EngineCore（PID 62605）还在跑
  - 宿主机上 3 个 VLLM::Worker（3561033, 3564009, 3565712）完全没被触及
  
  建议修复方向： service.md 的停止步骤中，fuser -k 之后应增加一步 —— 进容器内 pkill -f "vllm\|VLLM" 或直接 kill 掉 EngineCore/Worker
  进程族，确保子进程也被清理。需要我修改 service.md 吗？
- [x] **出现检查状态调用命令未约束问题** 现在进入 第 2 步：启动服务。先检查当前服务状态和 vllm-ascend 安装情况。
  
  ● Bash(PLUGIN_ROOT="/root/.claude/plugins/cache/vllm-ascend-tools/vllm-ascend-developer/3d834330c6b6"
  python3 "${PLUGIN_ROOT}/scripts/ssh_utils.py" docker-exec standal…)
  ⎿  Error: 拒绝无法审计本地写入的解释器命令: python3；请使用 Plugin 可信脚本
  
  ● Bash(PLUGIN_ROOT="/root/.claude/plugins/cache/vllm-ascend-tools/vllm-ascend-developer/3d834330c6b6"
  python3 "${PLUGIN_ROOT}/scripts/ssh_utils.py" docker-exec standal…)
  ⎿  Error: 拒绝无法审计本地写入的解释器命令: python3；请使用 Plugin 可信脚本
  
  Thought for 3s (ctrl+o to expand)
  
  ● Hook 拒绝了直接 python3 调用。让我检查项目设置和可用的调用方式。
  
  Thought for 11s, searched for 3 patterns, read 2 files (ctrl+o to expand)
  
  ● Bash(python3 /root/.claude/plugins/cache/vllm-ascend-tools/vllm-ascend-developer/3d834330c6b6/scripts/ssh_utils.py docker-exec standalone "fuser 8088/tcp 2>&1 || ech…)

