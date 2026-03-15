# OpenClaw (Clawdbot) 教程

## 简介

OpenClaw（原名 Clawdbot，过渡名 Moltbot）是开源个人 AI 助手项目，由 Peter Steinberger 开发。

OpenClaw 是一个可执行任务的智能体，我们给指令，它不仅回答，还能主动操作系统、访问网页、处理邮件、整理文件、发起提醒甚至自动编写代码。

OpenClaw 是一个把 **本地算力 + 大模型 Agent 自动化** 玩到极致的开发者效率工具。

OpenClaw 目标是让 AI 不只是给建议，而是直接完成完整工程任务。

![img](https://www.runoob.com/wp-content/uploads/2026/01/fe0e8572-f35b-4e20-8b00-7723e48ec498.png)

关于更名背景：2026-01-27 项目从 Clawdbot 更名为 Moltbot（商标顾虑），随后又更名为 OpenClaw；旧命令 clawdbot 作为兼容入口保留（详见报道与官方更新说明）。[来源 1](https://mashable.com/article/clawdbot-changes-name-to-moltbot)

Moltbot 是项目组为了应对侵权风险想出的过渡名字，OpenClaw 这是目前的最终官方名称。

- OpenClaw 官网: https://openclaw.ai/
- 中文文档： https://docs.openclaw.ai/zh-CN
- Github 地址：https://github.com/openclaw/openclaw
- OpenClaw 技能合集: https://github.com/VoltAgent/awesome-openclaw-skills

Clawbot、Moltbot 和 OpenClaw 其实是同一个开源项目，名字演进顺序为：

```
Clawdbot → Moltbot → OpenClaw
```

| 名称                   | 时间线                     | 背景/原因                                           | 本质关系                                   |
| :--------------------- | :------------------------- | :-------------------------------------------------- | :----------------------------------------- |
| **Clawdbot / Clawbot** | 2025 年末至 2026 年 1 月初 | 最初项目名；灵感来自 *Claude* 和 *claw*（龙虾爪）梗 | 原始名称，是最早出现在 GitHub 的身份       |
| **Moltbot**            | 2026 年 1 月 27 日         | 因 Anthropic 商标顾虑被要求更名                     | 中间过渡名字；功能、代码与 Clawdbot 一致   |
| **OpenClaw**           | 2026 年 1 月 30 日之后     | 抛弃版权冲突、强调开源性/长线品牌                   | 当前官方名称，也是今后文档、仓库等统一标识 |

## 安装方法

OpenClaw 的安装被设计得极为友好，即使是非开发者也能快速上手。

系统要求（不一定 Mac mini）：

- 环境：macOS / Linux / Windows
- Node：推荐 Node 24；兼容 Node 22 LTS（22.16+）。安装脚本会自动处理 Node 检测与安装。[来源 2](https://docs.openclaw.ai/install)
- pnpm：仅在你从源码构建时需要（常规安装不需要）

### 1、推荐安装方式（一键脚本）：

直接通过终端，执行以下命令。

macOS/Linux 系统:

```
curl -fsSL https://openclaw.ai/install.sh | bash
```

Windows 系统：

```
# PowerShell
iwr -useb https://openclaw.ai/install.ps1 | iex

# CMD
curl -fsSL https://openclaw.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

这会自动安装/检测 Node（推荐 Node 24；兼容 Node 22 LTS）并完成基本配置。[来源 2](https://docs.openclaw.ai/install)

### 2、手动安装

需要先准备 Node（推荐 Node 24；兼容 Node 22 LTS）。[来源 2](https://docs.openclaw.ai/install)

**使用 npm：**

```
npm i -g openclaw

# 如果速度慢可以指定国内镜像
npm i -g openclaw --registry=https://registry.npmmirror.com
```

**或使用 pnpm：**

```
pnpm add -g openclaw
```

安装完成后，初始化并安装后台服务（launchd / systemd 用户服务）：

```
openclaw onboard --install-daemon
```

### 3、从源码安装（开发模式）

```
git clone https://github.com/openclaw/openclaw.git
cd openclaw

pnpm install

pnpm ui:build   # 首次运行会自动安装 UI 相关依赖并构建前端界面
pnpm build      # 构建整个项目（包含后端与相关模块）

# 初始化 OpenClaw 并安装为系统后台服务（开机自动运行）
pnpm openclaw onboard --install-daemon

# 开发模式：监听 TypeScript 代码变更并自动重载网关服务
pnpm gateway:watch
```



后期要重新进入设置向导，可以执行以下命令：

```
openclaw onboard --install-daemon
```

启动后，我们可以使用 **openclaw status** 命令查看状态：

```
openclaw status
```

如果你在新手引导期间安装了服务，Gateway 网关应该已经在运行：

```
openclaw gateway status
```

手动运行（前台）：

```
openclaw gateway --port 18789 --verbose
```

安装后可以执行的命令：

- 运行新手引导：`openclaw onboard --install-daemon`（安装/修复后台服务）
- 快速检查：`openclaw doctor`
- 检查 Gateway 网关健康状态：`openclaw status` + `openclaw health`
- 打开仪表板：`openclaw dashboard`（或按 install/onboard 输出的 URL 打开）[来源 2](https://docs.openclaw.ai/install)

### 更新与升级

OpenClaw 迭代很快，推荐把升级当作“发版流程”：更新 → `openclaw doctor` → 重启 → `openclaw health` 验证。[来源 5](https://docs.openclaw.ai/install/updating)

推荐更新方式（覆盖大多数安装方式）：

```
curl -fsSL https://openclaw.ai/install.sh | bash
```

全局安装（npm/pnpm）也可以直接：

```
npm i -g openclaw@latest
# 或
pnpm add -g openclaw@latest
```

更新后建议执行：

```
openclaw doctor
openclaw gateway restart
openclaw health
```

### 4.  卸载

使用内置卸载程序：

```
openclaw uninstall
```

非交互式（自动化 / npx）：

```
openclaw uninstall --all --yes --non-interactive
npx -y openclaw uninstall --all --yes --non-interactive
```

## 使用技能

技能是包含 SKILL.md 文件的独立文件夹，想要为 OpenClaw 代理拓展新功能，ClawHub 是查找、安装技能的最优选择。

ClawHub 技能仓库地址：https://clawhub.ai/

安装 ClawHub 工具：

```
# 推荐方式（npx 无需全局安装）
npx clawhub@latest --version

# 或全局安装（方便以后直接用 clawhub 命令）
npm install -g clawhub
```

登录：

```
clawhub login
```

搜索技能：

```
clawhub search "postgres backups"
```

下载安装新技能：

```
clawhub install my-skill-pack
```

常用技巧：

- 指定安装到某个目录（避免写入权限问题）：

```
clawhub install <slug> --workdir <你的工作区> --dir skills
```

- 遇到 `Rate limit exceeded`（HTTP 429）时，通常等待几秒重试即可；频繁操作建议先 `clawhub login`。

批量更新已安装技能：

```
clawhub update --all
```

### 插件管理

插件是轻量代码模块，可通过新增命令、工具及 Gateway RPC 能力，为 OpenClaw 拓展额外功能。

查看已加载插件：

```
openclaw plugins list
```

安装官方插件（示例：语音通话插件）：

```
openclaw plugins install @openclaw/voice-call
```

插件安装后重启网关生效：

```
openclaw gateway restart
```

注意事项（非常重要）：

- 插件与 Gateway 同进程运行，不是沙箱隔离；只安装你信任的插件。[来源 3](https://docs.openclaw.ai/tools/plugin)
- 插件发现有优先级（配置路径 / workspace 扩展 / ~/.openclaw/extensions / 内置扩展）；同一个 id 只会激活最高优先级的那一份，低优先级会被忽略。[来源 3](https://docs.openclaw.ai/tools/plugin)
- 建议设置 `plugins.allow` 为可信插件 id，避免自动加载未追踪的本地代码。[来源 3](https://docs.openclaw.ai/tools/plugin)

### 常用命令

**OpenClaw 常用命令一览表：**

| 分类         | 命令                                 | 描述                                       |
| :----------- | :----------------------------------- | :----------------------------------------- |
| 初始化与安装 | `openclaw onboard`                   | 交互式向导（配置模型、通道、网关、工作区） |
| 初始化与安装 | `openclaw setup`                     | 初始化配置 + 工作区（非交互版）            |
| 初始化与安装 | `openclaw configure`                 | 交互式配置向导（模型、通道、技能）         |
| 网关管理     | `openclaw gateway status`            | 查看网关服务状态 + RPC 探活                |
| 网关管理     | `openclaw gateway start`             | 启动网关服务                               |
| 网关管理     | `openclaw gateway stop`              | 停止网关服务                               |
| 网关管理     | `openclaw gateway restart`           | 重启网关服务                               |
| 网关管理     | `openclaw gateway --port 18789`      | 直接在前台运行网关（调试用）               |
| 网关管理     | `openclaw health`                    | 从运行中的网关拉取健康数据                 |
| 配置管理     | `openclaw config file`               | 显示当前配置文件完整路径                   |
| 配置管理     | `openclaw config get <path>`         | 读取配置项                                 |
| 配置管理     | `openclaw config set <path> <value>` | 修改配置项                                 |
| 配置管理     | `openclaw config validate`           | 校验配置文件是否合法                       |
| 诊断与状态   | `openclaw doctor`                    | 一键健康检查 + 自动修复                    |
| 诊断与状态   | `openclaw status`                    | 显示会话健康状态和最近联系人               |
| 诊断与状态   | `openclaw logs --follow`             | 实时查看网关日志                           |
| 其他高频操作 | `openclaw dashboard`                 | 打开网页控制面板                           |
| 其他高频操作 | `openclaw channels status`           | 查看已连接的聊天通道                       |
| 其他高频操作 | `openclaw agent run`                 | 手动触发一次代理运行                       |

改完配置后记得运行 `openclaw gateway restart` 让改动生效。

### 通道管理

- **WhatsApp**：`openclaw channels login`（或扫描 QR）
- **Telegram**：`openclaw channels add --channel telegram`（需 Bot Token）
- **Discord**：`openclaw channels add --channel discord`（需 Bot Token）
- **iMessage**：macOS 原生桥接
- **Slack**：`openclaw channels add --channel slack`（需 Bot Token）

### 工作区结构（Workspace Anatomy）

- `AGENTS.md`：指令说明
- `USER.md`：偏好设置
- `MEMORY.md`：长期记忆（如存在；部分版本使用 `workspace/memory/`）
- `HEARTBEAT.md`：检查清单
- `SOUL.md`：人格/语气
- `IDENTITY.md`：名称/主题
- `BOOT.md` / `BOOTSTRAP.md`：启动配置（不同版本文件名可能不同）
- 根目录：`~/.openclaw/workspace`

### 聊天内斜杠命令

- `/status`：健康 + 上下文
- `/context list`：上下文贡献者
- `/model <m>`：切换模型
- `/compact`：释放窗口空间
- `/new`：全新会话
- `/stop`：中止当前运行
- `/tts on|off`：切换语音
- `/think`：切换推理模式

### 关键路径映射（Essential Path Map）

- 主配置：`~/.openclaw/openclaw.json`
- 默认工作区：`~/.openclaw/workspace/`
- 智能体状态目录：`~/.openclaw/agents/<agentId>/`（常见：`main/`）
- OAuth & API 密钥：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（旧版可能使用 `~/.openclaw/credentials/`）
- 向量索引存储：`~/.openclaw/memory/*.sqlite`（例如 `main.sqlite`）
- Skills（常见）：`~/.openclaw/workspace/skills/` 与 `~/.openclaw/extensions/<pluginId>/skills/`
- 日志：优先用 `openclaw logs --follow`；文件日志位置因安装/运行方式不同可能在 `~/.openclaw/logs/` 或 `/tmp/openclaw/`。[来源 4](https://docs.openclaw.ai/help/faq)

### 语音与 TTS

- 付费：OpenAI / ElevenLabs
- 免费：Edge TTS（无需 API Key）
- 自动 TTS：`messages.tts.auto: "always"`

### 内存与模型

- 向量搜索：`openclaw memory search "X"`
- 模型鉴权设置：`openclaw models auth setup`
- 日志：优先用 `openclaw logs --follow`；记忆/会话文件位于 `~/.openclaw/` 下

### Hooks 与技能

- ClawHub：`clawhub install <slug>`
- Hook 列表：`openclaw hooks list`

### 故障排除

- 无 DM 回复 → 配对列表 → 批准
- 群组中静音 → 检查提及模式配置
- 认证过期 → `models auth setup-token`
- 网关关闭 → `doctor --deep`
- 内存 Bug → 重建内存索引

### 自动化与研究

- 浏览器：`openclaw browser start` / `openclaw browser screenshot`
- 子代理：`/subagents list/info`
- 定时任务：`openclaw cron list` / `openclaw cron run <id>`
- 心跳：`heartbeat.every: "30m"`



## 通过第三方云直接安装配置

现在各大平台都已经支持这个智能体，如果不想安装在本机，可以一键部署云上OpenClaw：

- 阿里云：[https://www.aliyun.com/activity/ecs/clawdbot](https://www.aliyun.com/activity/ecs/clawdbot?userCode=i5mn5r7m)
- 腾讯云：[https://cloud.tencent.com/developer/article/2624973](https://cloud.tencent.com/act/cps/redirect?redirect=37925&cps_key=4537fb0f9e70f157220dafdec0f9c750&from=console)



## 常用命令

OpenClaw 常用命令如下：

| 命令                                               | 作用                      | 备注 / 参数                      |
| :------------------------------------------------- | :------------------------ | :------------------------------- |
| `openclaw status`                                  | 查看 Gateway 当前运行状态 | 包含健康度与上下文信息           |
| `openclaw health`                                  | 健康检查                  | 检测 core、依赖与运行环境        |
| `openclaw doctor`                                  | 综合诊断与修复建议        | 支持 `--deep` 深度检查           |
| `openclaw onboard`                                 | 交互式初始化向导          | 首次使用推荐                     |
| `openclaw onboard --install-daemon`                | 安装系统守护进程          | 后台常驻运行 Gateway             |
| `openclaw onboard --uninstall-daemon`              | 卸载守护进程              | 不删除数据                       |
| `openclaw configure`                               | 交互式配置向导            | 模型、通道、凭据等               |
| `openclaw config get <path>`                       | 获取配置值                | JSON Path                        |
| `openclaw config set <path> <value>`               | 设置配置项                | 支持 JSON5 / raw 文本            |
| `openclaw config unset <path>`                     | 清除配置项                | 移除单个键值                     |
| `openclaw channels list`                           | 列出已登录通道            | WhatsApp / Telegram / Discord 等 |
| `openclaw channels login`                          | 登录新的通道账号          | 扫码或授权流程                   |
| `openclaw channels add`                            | 添加通道                  | Telegram / Discord / Slack       |
| `openclaw channels status --probe`                 | 通道健康检查              | 检测连接可达性                   |
| `openclaw skills list`                             | 列出技能                  | 已安装 / 可用技能                |
| `openclaw skills info <skill>`                     | 技能详情                  | 参数、版本信息                   |
| `clawhub install <slug>`                           | 从 ClawHub 安装技能       | 官方技能市场                     |
| `openclaw hooks list`                              | 列出 Hook 列表            | 事件钩子机制                     |
| `openclaw plugins list`                            | 列出插件                  | 查看已安装插件                   |
| `openclaw plugins install <id>`                    | 安装插件                  | 例如 `@openclaw/voice-call`      |
| `openclaw plugins enable <id>`                     | 启用插件                  | 通常需要重启 Gateway             |
| `openclaw models list`                             | 列出可用模型              | 包含鉴权状态                     |
| `openclaw models status`                           | 模型状态                  | 当前可用性                       |
| `openclaw models auth setup-token`                 | 模型鉴权配置              | Cheatsheet 推荐方式              |
| `openclaw memory search "X"`                       | 搜索长期记忆              | 向量搜索                         |
| `openclaw memory index`                            | 重建记忆索引              | 修复 memory 异常                 |
| `openclaw logs`                                    | 查看日志                  | 默认聚合输出                     |
| `openclaw logs --follow`                           | 实时日志                  | `--json / --plain / --limit`     |
| `openclaw gateway install`                         | 安装 Gateway 系统服务     | 注册为系统守护进程               |
| `openclaw gateway start`                           | 启动 Gateway 服务         | system service 模式              |
| `openclaw gateway stop`                            | 停止 Gateway 服务         |                                  |
| `openclaw gateway restart`                         | 重启 Gateway 服务         | 配置变更后使用                   |
| `openclaw gateway status`                          | Gateway 系统服务状态      | 不同于 `openclaw status`         |
| `openclaw browser start`                           | 启动浏览器代理            | Automation 能力                  |
| `openclaw browser screenshot`                      | 网页截图                  |                                  |
| `openclaw subagents list`                          | 列出子代理                |                                  |
| `openclaw cron list`                               | 列出定时任务              |                                  |
| `openclaw cron run <id>`                           | 执行定时任务              |                                  |
| `openclaw uninstall`                               | 卸载 Gateway 服务及数据   | 官方推荐                         |
| `openclaw uninstall --all --yes --non-interactive` | 全自动卸载                | 状态 / workspace / 插件          |
| `openclaw uninstall --state`                       | 删除状态文件              | 不删除 workspace                 |
| `openclaw uninstall --workspace`                   | 删除工作区                | agent / workspace 数据           |
| `openclaw uninstall --service`                     | 仅卸载系统服务            | 不删除数据                       |
| `openclaw uninstall --dry-run`                     | 模拟卸载                  | 仅展示结果                       |

------

## 为什么最近这么火？

- 真正做到了"像JARVIS一样"：能读写文件、跑终端命令、操作浏览器、收发邮件、日历、写代码、订机票、清空收件箱……
- 本地优先 + 长期记忆：所有对话跨平台共享上下文，USER.md 和 memory/ 目录会越用越聪明
- 支持几乎所有大模型：Claude、Gemini、OpenAI、Ollama 本地模型、Pi 等
- 社区技能生态爆炸：ClawdHub 上已有 500+ 社区技能（Slack、Discord、GitHub、浏览器控制、macOS UI 自动化……）
- 安装简单像 npm install，实际能力却很 spicy （开发者原话）



其核心能力包括：

- 将自然语言目标拆解为可执行步骤
- 自动调用终端命令
- 创建与修改项目文件
- 运行代码并检测结果
- 根据报错自动修复

相比 Claude Code/OpenCode 这种代码补全工具，OpenClaw 更接近一个具备执行权限的工程型智能体。

- Claude Code 与 OpenCode等 强在代码质量与理解
- OpenClaw 强在自动完成整个工程流程

| 能力维度   | OpenClaw | Claude Code | OpenCode |
| :--------- | :------- | :---------- | :------- |
| 任务规划   | 强       | 中          | 中       |
| 自动执行   | 完整     | 部分        | 部分     |
| 自我修复   | 有       | 无          | 无       |
| 工程级操作 | 强       | 强          | 中       |
| 本地自动化 | 原生支持 | 较弱        | 较弱     |
