# 006 部署

用这份指南安装、配置和运维 dsh-superagent。它覆盖账号、启动检查、升级与故障排查，建立在 [001 领域模型](001-domain.md) 至 [005 面板](005-panel.md) 之上。

## 运行着什么

- **一个专用的 DeepSeek-Harness profile**，本指南中叫 `superagent`。从 Web profile 模板创建它。它加载 bundle，并在 `127.0.0.1` 上提供面板。把它和你日常使用的 profile 分开：插件会改变进程级的 `PATH` 和 git 设置，它的进程也需要独立的重启和升级。
- **一个智能体 GitHub 账号。** 智能体通过一个只以该账号登录的 GitHub CLI，用这个账号发表评论、推分支、开 pull request。
- **可选：** 一个用于会话请求的飞书机器人。

插件配置只保存状态目录和机器上限。其他都在面板里做：来源里设仓库和账号，策略里设允许名单，执行里设工作节点目录和命令。

### 权限与隔离

bundle 为执行器定义了 `dsh-superagent-worker`，为主 Session 定义了 `dsh-superagent-router`。它们出现在 DeepSeek-Harness 的预置列表里，但只应供 dsh-superagent 使用。不要给个人 Session 选它们，也不要禁用它们的配置行。本应用需要它们来创建和恢复自己的 Session。

在 Web UI 里打开一个空白托管 Session 可能会选中 profile 的默认预置。dsh-superagent 会在第一轮之前恢复它按角色需要的预置。

两个预置都以 `danger-full-access`、以操作者的操作系统用户身份运行，并读取已订阅线程里的消息。账号包装脚本默认选择智能体的凭据，但它们不隔离进程。工作节点可以按路径执行程序，也能读操作者的主目录。需要隔离的话，用一个只含智能体凭据的专用操作系统账号。

## 前置条件

- Node.js 22.19+ 或 24+、pnpm、git 2.32+、Python 3.8+ 和 tmux
- GitHub CLI（`gh`）2.48 或更高。来源里的 CLI 字段必须指向 `gh` 或 `lark-cli` 的包装脚本；宿主依赖它们的子命令和输出格式。
- github.com 上的一个仓库；不支持 GitHub Enterprise Server。要为每张工单创建检出，需要一个本地克隆，供 prepare 命令在上面添加 worktree。仓库用了 Git LFS 的话，安装 `git-lfs`。
- 为 DeepSeek-Harness 配置好的模型提供方。除非评估或执行设置里指定了别的，dsh-superagent 使用该 profile 的默认模型。
- 用飞书时：`lark-cli`（在 1.0.90 上验证过），以及一个带机器人的飞书或 Lark 应用

## 账号

### 智能体 GitHub 账号

1. 为智能体创建一个 GitHub 用户账号，并给它仓库的写权限。GitHub App 不能充当智能体，因为 dsh-superagent 读的是这个账号的通知。
2. 给它一个只用它自己凭据的 GitHub CLI：

   ```sh
   mkdir -p ~/.local/bin
   cat > ~/.local/bin/gh-agent <<'EOF'
   #!/bin/sh
   GH_CONFIG_DIR="$HOME/.config/gh-agent" exec gh "$@"
   EOF
   chmod +x ~/.local/bin/gh-agent
   gh-agent auth login
   gh-agent api user --jq .login
   ```

   以智能体账号登录；最后一条命令会打印它的登录名。dsh-superagent 用同样的方式读取登录名。

### 飞书机器人（可选）

1. 创建一个带机器人能力的自建应用。授予它接收消息事件、读取并发送会话消息、读取会话成员的权限，然后发布应用版本。
2. 给机器人一份自己的 `lark-cli` 配置，限定为机器人身份，并写一个包装脚本：

   ```sh
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config init --app-id <cli_…> --app-secret-stdin
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config strict-mode bot --global
   cat > ~/.local/bin/lark-bot <<'EOF'
   #!/bin/sh
   export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent"
   exec lark-cli "$@"
   EOF
   chmod +x ~/.local/bin/lark-bot
   ```

   第一条命令从标准输入读应用 secret。要用一个独立的配置目录：`lark-cli` 每个 app id 只允许一个 profile，而你的个人配置里可能已经有该应用的用户登录。

   宿主会拒绝能以用户身份行动的 CLI。它从机器人配置里读取 app id、飞书/Lark 品牌和机器人 `open_id`。

   两份配置共用钥匙串里同一条应用 secret。只想删掉机器人配置时，删掉它的目录即可；不要运行 `lark-cli config remove`，那会影响到共用的 secret。

## 安装

在没有 `superagent` profile 的机器上，把下面这段跑一次：

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
git clone <this repository> dsh-superagent && cd dsh-superagent
pnpm install && pnpm build
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

上面固定的版本号是公开的；其他版本必须满足 `package.json` 里的 `engines.dsh`。

第四条命令从 DeepSeek-Harness Web 模板创建 `~/.dsh/profiles/superagent`。第五条安装本 bundle。设置 `DSH_HOME` 可以改变默认的 `~/.dsh` 目录。

创建 profile 的那条命令只用于全新安装。profile 已存在时，该命令会替换它并删掉它的 `cordis.patch.yml`（含机器配置）。升级时不要用它。

## 机器配置

把 [examples/profile.cordis.patch.yml](../../examples/profile.cordis.patch.yml) 复制为 `~/.dsh/profiles/superagent/cordis.patch.yml`，替换掉该文件，并把每个 `<placeholder>` 换掉。用飞书时，也在那里启用 `dsh-superagent-feishu` 条目。`webserver` 条目设置面板端口。

| 键 | 必需 | 含义 |
| --- | --- | --- |
| `stateDir` | 是 | 绝对路径：数据库、锁、`worker-bin` 里的命令 shim、智能体的 git 配置，默认还有 `work` 里的工单目录。升级前先备份 |
| `promptMaxChars`、`admissionContextMaxPages`、`admissionContextTimeoutMs` | 否 | 保存的提示词配置的最大长度，以及一次研判读多少页线程、在多少时间内读完 |

首次启动之后，在 **Settings → Deployment**（设置 → 部署）或 DeepSeek-Harness 插件管理器里修改机器上限。改动无需重启即可生效。`stateDir` 不能在那里修改。无效值或未知的配置键会阻止启动；日志会指出是哪个键。

## 启动与停止

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

`start` 在 tmux 会话 `dsh-superagent`（其他 profile 则为 `dsh-<profile>`）里运行 `dsh --profile superagent`。它最多等一分钟，让运行时就绪或报告设置要求。首次运行时它会打印缺失的设置信息。它会拒绝重复实例，包括在本脚本之外启动的实例。

`stop` 和 `restart` 请求优雅关闭，最多为执行器等五分钟。它们从不强杀进程，也不碰其他 tmux 会话。

`status` 打印进程 id、健康状态和设置问题。换 profile 或端口时用 `--profile` 和 `--url`，或设置 `DSH_SUPERAGENT_PROFILE` 和 `DSH_SUPERAGENT_URL`。

`status`、`start`、`restart` 用四种退出码之一结束：

| 退出码 | 含义 |
| --- | --- |
| 0 | 服务正常运行 |
| 1 | 没有运行、运行了两份、没有响应，或运行时报告错误；重启可能修复 |
| 2 | 在运行，但运行时以外的某部分（例如某个来源，或策略同步 `sync:<source id>`）报告错误；重启修不好 |
| 3 | 在运行，但在等设置；重启修不好 |

仍在启动中的来源算作正常运行。设置没能检查的飞书机器人也算。

`status` 在 `errors` 下返回失败的部分及其细节，在 `setup` 下返回设置要求。`start` 和 `restart` 把这些打印成行；服务始终起不来时它们以 1 退出。只有刻意的 `stop` 才会暂停自动恢复。

每个 profile 把日志和控制文件存在 `.artifacts/` 下：

- 启动日志：`dsh-superagent-<profile>-<date>-<time>-<pid>.log`，其中 pid 属于脚本。保留最新的 20 个。
- 控制文件：`control/profiles/<profile>/`，含停止标记、锁和进程 id。

打开日志里打印的 URL，然后在侧边栏选择 **Superagent**。

### 每小时检查

在你日常使用的 DeepSeek-Harness profile 里创建这个每小时检查，不要放在 `superagent` 里。它在智能体服务停着的时候也必须继续运行。在那个 profile 里让智能体用下面这段提示词创建每小时排程，把检出路径填好：

```text
Check dsh-superagent: run `scripts/dsh-superagent --profile superagent status` in <checkout>. Exit 0: do nothing and say nothing. Exit 2: do not restart; tell me each part under `errors` with its detail, unless you already told me about the same ones. Exit 3: do not restart; tell me what `setup` says setup needs, unless you already told me. If it reports manually_stopped: true, leave it stopped. Otherwise (exit 1) run `scripts/dsh-superagent --profile superagent --respect-stop restart`, which exits the same way: 0 means it recovered, so say nothing; 2 and 3 are reported as above; if it exits 1 or fails otherwise, read the newest .artifacts/dsh-superagent-superagent-*.log and the Troubleshooting section of docs/006-deployment.md, fix what you can, and tell me what was wrong and what you did.
```

`--respect-stop` 会保留刻意的停止。只有 `stop` 写标记；`start` 清除它。`restart` 不写标记，所以检查可以救回一次关闭超时的重启。

## 设置

开始轮询或工作之前，设置必须成功。概览会列出缺失的要求，并链接到对应设置。设置改动会重新跑一次 setup。如果你在应用之外修了东西（例如 CLI 登录），请点 **Check again**（重新检查）。

1. **Sources 来源。** 设置 GitHub 仓库（`owner/name`）和智能体 CLI，例如 `/home/me/.local/bin/gh-agent`。请求通常通过智能体账号的通知到达。提到该账号的新 Issue 或新评论会在下次轮询时到达；靠编辑已有 Issue 正文加进去的 @ 不会。改为发一条新评论。用飞书时，设置机器人 CLI、会话、每个会话是"收全部"还是"仅 @"，以及是否接收私聊。
2. **Policy 策略。** 两条示例规则可以改也可以删。在 **On each Source**（每个来源上）下，添加可以请求工作的 GitHub 账号和飞书用户；名单上没有人之前，任何请求都无法变成工单。用飞书时，可以选一个会话来同步名单，在启动时和点 **Sync now**（立即同步）时同步。
3. **Execution 执行。** 目录根和一个 prepare 命令。要让每张工单得到仓库的一份检出：

   ```sh
   git -C /home/me/src/widgets fetch --no-tags "https://github.com/$DSH_SUPERAGENT_GITHUB_REPOSITORY.git" "${DSH_SUPERAGENT_GITHUB_PR_HEAD:-HEAD}:refs/dsh-superagent/$DSH_SUPERAGENT_TICKET" &&
   git -C /home/me/src/widgets worktree add -b "agent/$DSH_SUPERAGENT_NAME" "$DSH_SUPERAGENT_DIR" "refs/dsh-superagent/$DSH_SUPERAGENT_TICKET"
   ```

   把个人的 CLI（例如登录到你本人账号的那个）列为被禁命令。
4. **Prompts 提示词。** 设置团队小节：GitHub 披露行、评审机器人、提交作者和自动合并条件。

### 管理 API

面板使用一个本地 HTTP API，智能体也可以调用它。`SETUP.md` 用它保存设置，服务脚本读 `GET state`。

把请求发到 `http://127.0.0.1:<port>/orchestrator/api/`，带上 `x-orchestrator-request: 1`。Host 必须是 `127.0.0.1:<port>` 或 `localhost:<port>`。缺少请求头、或 `Origin` 来自别的站点，都会收到 403。

通过 API 保存具有面板的权限。与通过主 Session 的工具保存不同，它们可以修改执行环境和设置里的账号字段。

智能体可以依赖以下路由和字段。`src/app/api.ts` 里的其他路由服务于面板，会随面板变化。

| 请求 | 响应 |
| --- | --- |
| `GET state` | 每个页面都读的东西：`setup`，含 `problem` 与 `feishuProblem`，各自为 `null` 或一个带 `place`、`code`、`values`、`detail` 的设置问题，以及 setup 解析出的 `accounts`；`health`，每个部分一个条目（见下表），含 `state`（`starting`、`ready`、`paused` 或 `error`）、`detail` 和 `at`；`load`，含 `running` 的执行器 Session、`queued` 的工单和 `maxWorkers`；`paused`；以及 `masterId` |
| `GET settings` | `blocks`：`policy`、`eval`、`executor`，各自带 `value` 与 `revision`；`sources`：来源实例，各自带 `id`、`kind`、`installed`、所属块的 `value`（`{"enabled": …, "config": {…}}`）、`revision`、其种类的表单、以及有问题时的 `problem`；`policyVersion`；`environment`，执行环境的 `root` 与 `variables` |
| `GET sources` | 每个来源的状态：`state`（`missing`、`off`、`error`、`setup`、`starting`、`paused` 或 `ready`）、其各部分的 `health`、设置 `problem`、最近一天的 `stats`、该日 `filtered` 的触发、该日 `dropped` 的消息（最新丢弃在前，各自带 `channel`、`preview`、属某张工单时的 `ticket`（`id` 与 `title`）、存储里有的话的 `thread`、以及 `dropped.at` 与 `dropped.reason`、`channelGone` 或 `channelIgnored`） |
| `PUT settings/<block>`，body 为 `{"value": …, "revision": n}` | 保存一个块：`saved`、该块的新 `revision`、`policy_changed`、`restarting`。`value` 是整个块，形如 `GET settings` 里的样子；缺少的键取默认值 |
| `POST executor/pause`，body 为 `{"paused": true}` 或 `false` | 暂停或恢复处理：`paused` |
| `POST setup/check` | 通过重启 dsh-superagent 重新执行 setup：`restarting: true`。没有可修的东西时会被拒绝 |
| `POST policy-sync/<source id>` | 立即同步该来源的允许名单：`policy`，该来源已保存的那一部分 |
| `GET prompts`、`PUT prompts`（body 为 `{"config", "baseRevision"}`） | 读取提示词配置：`current`（含 `revision` 与 `config`）、`defaults`、`sections`（团队小节，按 Session 读取的顺序）、`skillName`（团队技能名的模式）、`builtin`（团队技能不能使用的名字）、`skillPrefix`（Session 加载技能名时用的前缀）。或者保存它：返回新的 `revision` |
| `POST tickets/<id>/retry`、`POST tickets/<id>/stop` | 像面板一样重试或停止某张工单 |

健康条目按职责分组：

| 条目 | 报告 |
| --- | --- |
| `runtime` | 应用本身 |
| `models` | 正在等模型预算重试的 Session |
| `evals` | 最近一小时内、之后没有新判断或忽略的 `unavailable` 评估 |
| `assessment`、`launch`、`delivery`、`relay`、`notice`、`reply` | [003 执行](003-execution.md) 里描述的存储工作；delivery 统计等待中的工作，reply 覆盖渠道消息 |
| `worker` | 执行器 Session 的事件处理；首次失败之后出现 |
| `github`、`feishu`、`history` | 来源的轮询、事件与历史回溯 |
| `sync:<source id>` | 允许名单同步 |

拒绝会带状态码和 `error`（英文文本）：

- 400：无效值（`issues`，各自带 `path`）、账号检查失败（`setup`）、固定拒绝（`code` 与 `values`），或 body 不是 JSON 或过大。
- 404：东西不存在。
- 409：版本号过期。
- 500：其他情况。

各个块：

- `source:github`、`source:feishu`：`{"enabled": true, "config": {…}}`。GitHub 的 config 有 `repository` 和 `cli`，以及开关 `observeIssues`、`observePulls`、`observeComments`、`observeCi`（默认关闭），开启后接收仓库里每个新 Issue、新 pull request、评论，以及默认分支的 CI；飞书的 config 有 `cli`、`chats`（形如 `[{"chatId": "oc_…", "name": "…", "listen": "mentions"}]`，或 `"all"`）、`directMessages`，以及 `otherChats`（`"none"` 或 `"mentions"`）。
- `policy`：`{"rules": […], "sources": {"github": {"allowed": [{"id": "<login>"}], "bots": [{"id": "dependabot[bot]"}]}, "feishu": {"allowed": [{"id": "ou_…", "name": "…"}], "syncChat": "oc_…", "admission": {"oc_…": {"unlistedAddressed": "explicit"}}}}}`。`bots` 列出其消息可以成为推断候选的机器人。`admission` 的一行以会话 id 为键；飞书还可用 `other` 和 `direct`，GitHub 用 `issues`、`pulls`、`comments`、`ci`；未设置的单元格取默认值（见 [002 授权](002-authorization.md)）。
- `eval`：`mode`、`timeoutSeconds` 和 `model`。
- `executor`：`model`、`maxWorkers`、`directory`、`prepare` 和 `blockedCommands`。处理暂停是一项独立设置 `paused`，由 `POST executor/pause` 设置。

修改仓库或 CLI 会先跑对应的 setup 检查。检查失败返回 400，细节在 `setup` 里。账号修改成功、或在 setup 待处理期间的任何保存，都返回 `restarting: true`。应用重启期间 API 暂时不可用。

```sh
api=http://127.0.0.1:3082/orchestrator/api
rev=$(curl -s -H 'x-orchestrator-request: 1' $api/settings | jq '.sources[] | select(.id == "github") | .revision')
curl -s -X PUT -H 'x-orchestrator-request: 1' -H 'content-type: application/json' $api/settings/source:github \
  -d "{\"value\": {\"enabled\": true, \"config\": {\"repository\": \"acme/widgets\", \"cli\": \"$HOME/.local/bin/gh-agent\"}}, \"revision\": $rev}"
```

## 设置检查

setup 使用 DeepSeek-Harness 的子进程服务，工作节点的 shell 也一样。它会移除名字里含 KEY、PASSWORD、SECRET 或 TOKEN 的环境变量。

setup 验证 GitHub 仓库和智能体的推送权限，写入工作节点环境，并检查 git 对 github.com 只用智能体凭据。飞书 setup 失败只会停用飞书来源。

setup 写入的进程环境：

- **命令 shim** 位于 `<stateDir>/worker-bin`，排在 `PATH` 最前：`gh` 运行智能体的 CLI；`lark-cli` 运行机器人的 CLI；被禁命令会失败。shim 在运行目标之前会把自己的目录从 `PATH` 里移除。
- **git 配置** 位于 `<stateDir>/worker-gitconfig`：替换操作者的全局与系统 git 配置。它把智能体 CLI 选为 github.com 的凭据助手，把 GitHub 的 SSH 地址改写成 HTTPS，并定义 Git LFS 过滤器。HTTP 代理之类的个人 git 设置不会生效。这里不设置提交身份，git 从不提示输入凭据，`GH_TOKEN` 和 `GITHUB_TOKEN` 会被移除。

`worker-bin` 或 `worker-gitconfig` 里出现不是由 dsh-superagent 写入的文件，会中止 setup。工单目录位于某个 git 仓库内部、而该仓库自己的配置里加了凭据时，该工单会失败。

## 试一下

用允许名单上的 GitHub 账号，开一个 Issue 或发一条包含 `@<agent login> <request>` 的评论。在一个轮询间隔内（默认一分钟），概览应该出现一张工单。智能体确认请求，在自己的 Session 和目录里干活，并在原来的会话里交付。用飞书时，在配置过的会话里 @ 机器人。

## 升级

1. 等到没有执行器正处在一轮中间。
2. 备份 `<stateDir>/orchestrator.sqlite`。
3. 运行 `git pull && pnpm install && pnpm build`。
4. 运行 `scripts/dsh-superagent restart`，并检查它的状态。

重启会结束正在运行的派工，并带着恢复说明启动替代者。团队文本和被覆盖的提示词部件会保留；没有被覆盖的部件跟随新默认值。升级之后要复查标为 **Default updated**（默认内容已更新）的部件。

被新版本迁移过的数据库，旧版本打不开。所以回滚需要升级前的备份。升级 DeepSeek-Harness 本身之后，重启正在运行的实例，让面板加载匹配的产物。

### 智能体预置

worker 和 router 预置在 5 秒后把 shell 命令转到后台。Session 没有它的预置就无法加载。这些 Session 在用的时候，保留 `cordis.patch.yml` 里的 `dsh-superagent-worker` 和 `dsh-superagent-router` 行；否则恢复会以 `agent-preset/not-found` 失败。

## 故障排查

设置问题用面板的语言显示在概览上，并链接到修复处；日志和 API 显示下表中的英文原文。

| 症状 | 原因与修复 |
| --- | --- |
| `The agent GitHub CLI is not set.` | 在 GitHub 卡片上设置智能体的 CLI |
| `The agent GitHub CLI was not found.` | 设置包装脚本的绝对路径，或 `PATH` 上的命令名 |
| `The agent GitHub CLI cannot read its account` | 用 `auth login` 登录该包装脚本 |
| `@… cannot read …` 或 `@… cannot push to …` | 给智能体账号仓库写权限 |
| `The GitHub Source names no repository.` | 在 GitHub 卡片上设置仓库，`owner/name` |
| 工单以 `The prepare command failed: …` 失败 | 输出里指明了原因，例如克隆路径不对；修好命令后重试该工单 |
| `git does not read the github.com credential configuration dsh-superagent writes` | 安装 git 2.32 或更高版本 |
| 工单以 `The git configuration of the repository in … sets …` 失败 | 从克隆的 `.git/config` 里移除被点名的凭据设置或 `http.*.extraheader` 设置 |
| `… was not written by dsh-superagent` | 把被点名的文件移出 `<stateDir>/worker-bin` 或 `<stateDir>/worker-gitconfig`，或者换一个 `stateDir` |
| `The bot lark-cli allows user identities` | 让飞书来源指向一份仅机器人的配置，参见上面的飞书机器人 |
| `The bot lark-cli does not work.` 或 `The bot identity of app … is …` | 检查配置里的 app id 和 secret |
| `The Feishu bot is not a member of the chat.` | 在会话设置里把机器人加入该会话；下次尝试就会成功 |
| `status` 以 2 退出 | 某部分报错，重启修不好：`errors` 会点名每个部分（见管理 API 一节）及其细节；每部分恢复后会自动清除自己的条目 |
| `status` 以 3 退出 | setup 在等待：`setup` 说明缺什么，概览链接到能修好它的设置 |
| `status` 以 1 退出 | 服务没在运行、运行了两份、没有响应，或运行时报告错误；输出和最新日志会说明是哪种 |
| `… timed out after 90 s`、`… was stopped by …`、`… exited with code …` 或 `… could not run: …` | 某个 CLI 失败却没有打印原因；以操作者用户手动运行被点名的程序，看它自己的消息 |
| 面板说 dsh-superagent 没有响应 | 它正在重启，或因配置没能启动；DeepSeek-Harness 日志会点出原因 |
| 日志里出现 `dsh-superagent configuration has keys it does not use` | 删除被点名的键；消息会说明每个值现在放在哪里 |
| 日志说插件与正在运行的 dsh 不兼容 | 装一个 `engines.dsh` 范围内的 DeepSeek-Harness 版本 |
| DeepSeek-Harness 升级后面板显示 "Failed to load plugins" | 重启正在运行的实例 |
| @ 了之后没有任何反应 | 检查作者是否在该来源的允许名单上、来源是否开启、处理是否被暂停。在 GitHub 上，靠编辑 Issue 正文添加的 @ 不会被接收；在评论里 @ 智能体 |
