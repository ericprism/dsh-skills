---
name: github-project-lookup-install
description: 当用户给出 GitHub 项目名称或链接，需要联网查找项目、通读 README 并生成项目概述，再由用户选择 git clone、下载 zip 或按 README 安装时使用。含 GitHub 连通性闸门、安装过程 5 分钟异常监控与系统级操作汇报；每个新项目独立重新检索，不沿用会话上下文。
---

# GitHub 项目查找与选择性安装

三条硬闸门，未通过就停，不得先做后面的步骤再补：

1. 网络不通 → 暂停工作流；
2. 用户未明确选择安装方式 → 不 clone、不下载、不执行 README 命令；
3. 系统级操作未获确认 → 不执行。

## 流程清单

0 解析输入 → 1 网络闸门 → 2 定位项目 → 3 通读 README → 4 生成概述文档 → 5 安装决策 → 6 执行与监控 → 7 系统级汇报 → 8 收尾与优先事项

新项目输入先按下一节重置，再进入第 0 步。

## 输入隔离：每个新项目都从零开始

用户给出新的项目名称或新的链接时，视为一次全新执行：不读取、不引用会话上下文里的项目信息，只把本轮消息中的 `{input}` 和本轮实时请求到的 GitHub 数据作为事实来源。

| 本轮消息 | 处理 |
|---|---|
| 出现项目名/链接，且与当前项目不同 | 清空上一项目的全部状态，从第 0 步重跑 |
| 同一项目再次输入 | 默认也重新核验（README 可能已更新），除非用户明确说“沿用刚才的结果” |
| 安装方式选择、参数补充、“继续”等短回复 | 属于当前项目的后续交互，不重置 |

硬规则：

1. 不复用上一项目的 README 摘录、概述文档、类型判断、安装步骤、失败原因；需要就重新获取。
2. 不继承上一项目的安装许可和系统级操作许可，新项目重新征求确认。
3. 不凭记忆或上下文回答项目事实（功能、依赖、启动方式），一律以本轮请求到的 README 为准。
4. 每轮重新过网络闸门、重新解析 `{input}`，不沿用上一轮的连通性结论。
5. 工作区里同名的旧 `<repo>-项目概述.md` 和旧日志只作历史文件，不作为本轮结论依据；本轮重新生成，不覆盖旧文件。
6. 上下文与实时数据冲突时，以实时数据为准，并在报告中说明差异。

“不读取上下文”不等于失忆：技能规则、DSH 工具用法、用户在本轮消息中写明的偏好仍然生效；被隔离的是其他项目的具体事实与决定。

## 0. 解析输入

`{input}` 指用户原始输入，全程保留原文，不要自行改写项目名。若本轮是新项目（见“输入隔离”），先丢弃上一项目的仓库、分支、README、概述路径、安装计划与已获批准事项，再按下表重新解析。

| 输入形态 | 规范化处理 | 注意 |
|---|---|---|
| `owner/repo` | 拼成 `https://github.com/owner/repo` | 先用 API 校验存在 |
| 完整 URL（含 `.git`、`/tree/<branch>/<subdir>`、查询串） | 去掉 `.git` 和查询串；记录分支、子目录 | 子目录才是项目根时，安装与阅读都以子目录为准 |
| `git@github.com:owner/repo.git` | 查询转 HTTPS | clone 可沿用 SSH 地址 |
| 裸名称（如 `langflow`） | 进入搜索，不猜测 | 同名项目多时让用户选 |
| 名称 + 描述（如 `xxx 的 webui`） | 组合关键词搜索 | 记录匹配依据 |

先做本地预检：`git --version`、`curl.exe --version`。缺 git 时仍可走 zip 路线，在报告里注明缺失项。默认安装目录为 `<workspace>/<repo>`，未与用户确认不要改。

## 1. 网络闸门（硬闸门）

以 HTTPS 探测为准；ICMP 被禁是常态，`Test-Connection github.com -Count 2` 只作 DNS/IP 层参考，不作为失败依据。

```powershell
foreach ($u in 'https://api.github.com/rate_limit','https://github.com','https://raw.githubusercontent.com') {
  $code = curl.exe -sS -o NUL -w '%{http_code}' --max-time 10 $u 2>$null
  if (-not $code) { $code = '000' }
  "$u`t$code"
}
```

POSIX 把 `NUL` 换成 `/dev/null`。

- 通过：api 返回 200；github 返回 2xx/3xx；raw 2xx/3xx。raw 失败但前两项通过时可继续，等到下载阶段再验证。
- 若 curl 因本机 TLS 凭据问题（如 Windows Schannel `SEC_E_NO_CREDENTIALS`）全部返回 `000`，但 `git ls-remote https://github.com/git/git.git HEAD` 退出码为 0，判定网络通过，并在报告里注明 curl 不可用、后续优先用 git。
- 失败：超时、`000`、5xx、证书/握手错误。单次失败隔 3 秒重试一次，总预算不超过 30 秒。
- 限流（403 且 `rate_limit.remaining` 为 0）不算断网，报告并建议配置 `GITHUB_TOKEN`（只引用变量，不打印值）。

失败时只做只读诊断：

- DNS：`Resolve-DnsName github.com`（或 `nslookup github.com`）
- 代理：`$env:HTTPS_PROXY`、`git config --global --get http.proxy`
- 端口：`Test-NetConnection github.com -Port 443`
- TLS：证书或握手错误，多为代理、杀毒软件或系统时间问题

**不通过 → 暂停当前会话工作流**：列出每个端点的结果、最可能的原因、2–3 条可操作的修复建议（开/关代理、换网络、配置 token），然后停住等用户。禁止改用镜像站、代理站或本地缓存绕过检查继续安装。

## 2. 定位项目

URL 直接解析；裸名称走搜索：

```powershell
$q = [uri]::EscapeDataString('<关键词>')
curl.exe -sS --max-time 20 -H 'Accept: application/vnd.github+json' `
  "https://api.github.com/search/repositories?q=$q&sort=stars&order=desc&per_page=5"
```

设置了 `GITHUB_TOKEN` 时追加 `-H "Authorization: Bearer $env:GITHUB_TOKEN"` 提高限额，不要把 token 写进日志或回复。

再用 `/repos/{owner}/{repo}` 核对：`full_name`、`default_branch`、`archived`、`license.spdx_id`、`stargazers_count`、`pushed_at`、`size`（KB）、`language`、`description`。

- 重命名或转移：API 会 301，记录规范 `full_name` 并告知用户。
- 404：私有、已删除或拼写错误，停下请用户确认，不要拿同名项目顶替。
- 有歧义：列出前 3 个候选（全名、Star、一句话描述），用户选定后再继续。

## 3. 获取并通读 README

优先用 API，能自动处理文件名大小写和扩展名：

```powershell
curl.exe -sSL --max-time 30 -H 'Accept: application/vnd.github.raw' `
  'https://api.github.com/repos/{owner}/{repo}/readme'
```

失败再试 `https://raw.githubusercontent.com/{owner}/{repo}/{branch}/README.md`（也试 `readme.md`、`README.rst`）。

用 `read` 工具完整读取；超过 2000 行分段读完，确实读不全时明确记录“未读部分”，不要用摘要冒充全文。逐项记下原文依据：

- 一句话定位与核心功能
- 前置依赖（运行时版本、系统库、GPU/内存/磁盘）
- 安装步骤（包管理器、脚本、Docker、二进制包）
- 启动/使用命令、端口、配置文件、环境变量与密钥
- 模型/数据下载、首次初始化
- 许可证、平台限制、已知坑
- 是否包含 skill 或插件清单（下一步核对）

## 4. 生成《项目概述文档》

默认写入 `<workspace>/<repo>-项目概述.md`，回复里再给要点。文件名冲突时加日期后缀，不覆盖已有文件。

固定结构，顺序不变：

### 基本信息

表格：仓库、规范链接、默认分支、Star、最近提交、License、语言、体积、归档状态。

### 1. 项目能做什么 / 功能 / 作用

2–5 条，来自 README 的实际描述，区分核心能力与附属功能。

### 2. 使用方式

- 环境要求
- 安装（按 README 原步骤，标注适用平台）
- 配置（配置文件/环境变量；密钥只写占位符）
- 启动与验证（命令 + 预期输出或 URL）
- 最小可用示例（README 有就引用，没有就写“README 未提供”）

### 3. 是否存在 skill

先拉文件树，避免逐个猜：

```powershell
curl.exe -sS --max-time 20 -H 'Accept: application/vnd.github+json' `
  'https://api.github.com/repos/{owner}/{repo}/git/trees/{branch}?recursive=1'
```

在返回的 `tree[].path` 中匹配：

| 证据 | 结论 |
|---|---|
| `SKILL.md`（根或子目录）、`skills/*/SKILL.md`、`.dsh/skills/`、`.agents/skills/`、`.claude/skills/` | 有原生 skill |
| `cordis.patch.yml`、`package.json` 含 `dsh.profile.bundles`、`.claude-plugin/marketplace.json`、`plugin.json` | 插件形态，需要宿主 |
| `.mcp.json` 或 MCP server 配置 | 提供 MCP 工具，不是 skill |
| 以上都没有 | 无 skill |

结论必须写成 **有（路径 + 安装方式）/ 无 / 仅插件形态**。有 skill 时给出可直接复制到 skills 目录的路径。

### 4. 项目类型：完整程序 / 插件 / 综合性项目

| 类型 | 判定证据 |
|---|---|
| 完整程序 | 有独立入口（CLI/GUI/server/main），README 给出装完即运行的启动命令，不依赖特定宿主 |
| 插件 | 无独立运行入口，由宿主加载（DSH/Claude/Cordis/VSCode/Obsidian 等），含 manifest 或注册文件 |
| 综合性项目 | monorepo / 多 package，同时含可运行应用、库、插件或服务；根目录有 workspaces、`packages/`、`apps/` |
| 库/SDK（补充） | 只被其他代码引用，不直接运行 |

写“结论 + 证据文件”，不要只给标签。

### 附：风险与注意（建议写）

License 限制、停更/归档、安装脚本会做什么、是否要管理员权限、是否联网下载模型、磁盘占用。

写完自检：四个必填小节齐全；每条结论都能指回 README 或仓库文件；没有把推测写成事实；全部内容来自本轮实时请求，没有复用其他项目的文字。

## 5. 安装决策（硬闸门）

用 `ask_user_question` 给三个选项，并写清代价：

- **git clone（推荐）**：`--depth 1`，可后续 `git pull`；需要 git 和网络。
- **下载 zip**：没有 git 也能用；无版本历史，更新要重新下载。
- **暂不安装**：只交付概述文档。

同时说明目标路径、预计体积（API 的 `size`）、将执行的命令、是否需要管理员权限或系统级依赖。

用户选择前不 clone、不下载、不执行 README 里的任何命令。选“暂不安装”时，给出“稍后如何安装”的命令后结束。

## 6. 执行安装并监控

### 6.1 准备

- 目标目录已存在且非空：停下问用户（覆盖/换名/取消），禁止直接删除。
- 记录开始时间、命令全文、日志路径：`<workspace>/.install-logs/<repo>-<步骤>.log`。

### 6.2 clone

```powershell
git clone --depth 1 https://github.com/{owner}/{repo}.git '<target>'
```

README 指定 tag/分支时加 `--branch <ref>`；确实需要 submodule 时再 `git submodule update --init --recursive`。

### 6.3 zip

```powershell
curl.exe -L --fail --max-time 300 -o '<repo>.zip' `
  'https://codeload.github.com/{owner}/{repo}/zip/refs/heads/{branch}'
Expand-Archive -LiteralPath '<repo>.zip' -DestinationPath '<target-parent>' -Force
```

校验 zip 大于 0 字节、解压出非空目录；成功后删除 zip，失败则保留供排查。解压出的目录通常带分支后缀，确认后重命名为 `<repo>`，重命名前检查目标不存在。curl 报 TLS 错误时改用 `git clone`，或让用户用浏览器下载 zip 后手动解压。

### 6.4 按 README 安装依赖

- 严格按 README 的顺序和包管理器；Windows 上把 `sudo apt` 一类步骤替换为对应做法，无法替换就跳过并告知用户。
- 远程脚本（`curl ... | sh`、`iwr ... | iex`）必须先展示完整 URL 和脚本作用，经用户确认后再执行，默认不执行。
- 不装全局包、不改系统配置、不申请管理员权限；这些进入第 7 节先汇报。

### 6.5 监控（关键）

长任务（大仓库 clone、装依赖、编译、下载模型）放后台运行，日志落盘：

- 每 30–60 秒读一次输出或日志尾部，确认仍在推进（字节数或时间戳在变化）。
- 异常判定，满足任一即异常：
  1. 单条命令非 0 退出；
  2. 同一任务持续超过 5 分钟仍未完成；
  3. 日志连续 2 分钟无增长，且没有 CPU/网络活动（疑似卡死）。
- 异常处理：
  1. 取退出码、日志最后 50 行、当前目录与磁盘余量，先判断是失败还是仍在跑；
  2. 分类：网络中断/超时、认证（私有库、token）、磁盘满、权限/占用、缺运行时、依赖冲突、编译 OOM 或超时；
  3. 仅对瞬时故障（网络超时、registry 5xx）自动重试 1 次，间隔递增；重试前终止残留进程；
  4. 仍失败 → 停止安装，报告失败命令、退出码、日志路径、原因判断、已完成步骤、可选下一步（换网络/装运行时/改用 Docker/手动命令），不要连续盲试。
- 超过 5 分钟但确实在正常推进（如大模型下载）：向用户汇报进度与预计剩余时间后继续监控，不要静默等待。

## 7. 系统级操作汇报

执行前必须汇报并取得确认（`ask_user_question`），说明：做什么、为什么、体积/耗时、影响范围、回滚方式。

| 类别 | 例子 |
|---|---|
| 运行时 | Node/Python/Rust/.NET/JDK/Docker/WSL/CUDA 驱动 |
| 全局包 | `npm i -g`、`pnpm add -g`、`pipx install`、`cargo install`、winget/choco/brew |
| 系统配置 | PATH、环境变量、服务/开机启动、防火墙、端口、shell profile |
| 大体积 | 超过 500MB 的下载、模型权重、构建缓存 |
| 权限 | 管理员/sudo/UAC、注册表 |

汇报模板：`将安装 <组件> <版本>，约 <大小>，写入 <位置>，会修改 <PATH/服务>；回滚：<命令>。是否继续？`

安装后验证版本或服务状态，并把“改了什么、怎么撤销”记入最终报告。

## 8. 收尾

### 8.1 验证

按 README 的最小验证方式：`--version`、启动命令、健康检查 URL、示例脚本。失败就回到 6.5 分析，不要宣布成功。

### 8.2 报告

- 结果：成功 / 部分成功 / 失败
- 位置、版本、分支或 commit
- 执行过的关键命令与耗时
- 系统级改动清单与回滚方式
- 日志路径、遗留问题

### 8.3 现在最优先要做的几件事

给 3–5 条，按依赖顺序，每条带确切命令或路径，例如：

1. 复制 `.env.example` 为 `.env` 并填 `<必填项>`（不打印密钥）；
2. 初始化数据库或下载模型：`<命令>`；
3. 启动：`<命令>`，然后打开 `http://localhost:<port>`；
4. 跑通示例：`<命令>`；
5. 按 README 配置可选集成。

只 clone 未安装时，优先项改为：装依赖 → 配置 → 启动 → 验证 → 读文档。

## 停止条件

遇到即停，说明原因并等指令：

| 条件 | 动作 |
|---|---|
| 网络闸门失败 | 暂停，给诊断和修复建议 |
| 项目找不到或有歧义 | 暂停，请用户确认 |
| 用户未选安装方式或选择“暂不安装” | 不执行安装 |
| 5 分钟异常且无法定位、重试失败 | 停止安装，保留现场和日志 |
| 系统级操作被拒绝 | 跳过该项，报告影响 |
| 目标目录冲突 | 暂停，等用户决定 |

“暂停工作流”指在回复里明确写“工作流已暂停”及恢复所需条件；不启动后台重试，不继续后续步骤。

## 禁止

- 未过网络闸门就搜索或下载；用镜像绕过失败检查。
- 把上一个项目的 README 内容、概述、类型判定或安装步骤套到新项目上。
- 凭记忆或会话上下文回答新项目；新项目必须重新联网核实。
- 未获用户确认就 clone、下载或执行 README 命令。
- 静默执行 `curl | sh`、`sudo`、管理员安装、全局包安装。
- 覆盖或删除已存在的目录/文件；把密钥写进日志或报告。
- 把 README 没写的内容编造成功能，把推测当结论。
- 单一任务卡住超过 5 分钟仍不汇报、不处理。
