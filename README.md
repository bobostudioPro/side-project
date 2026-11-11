# Superagent 安装过程

[Superagent](https://github.com/deepseek-ai/dsh-superagent-prototype)（`dsh-superagent`）是运行在 [DeepSeek Harness](https://www.deepseek.com/en/harness/)（`dsh`）上的主动式编码 Agent。它监听 GitHub（可选飞书），把获授权的请求变成工单，在独立会话和目录中执行，并把结果（通常是 Pull Request）回复到原对话。

本文记录从零安装、启动到完成首次配置的完整过程，适用于 Linux、macOS 和 WSL。更完整的运维说明见官方 [SETUP.md](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/SETUP.md) 与 [docs/006-deployment.md](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/006-deployment.md)。

## 1. 安装前准备

| 依赖 | 要求 | 用途 |
| --- | --- | --- |
| Node.js | 22.19+ 或 24+ | 运行 DSH 与 Superagent |
| pnpm | 最新稳定版 | 安装和构建依赖 |
| git | 2.32+ | 拉取代码、管理工单目录 |
| Python | 3.8+ | 部分工具链依赖 |
| tmux | 任意较新版本 | 后台运行 Superagent |
| GitHub CLI（`gh`） | 2.48+ | Agent 账号的 GitHub 操作 |
| DeepSeek Harness（`dsh`） | 满足 `package.json` 中 `engines.dsh` 的版本 | Superagent 的运行宿主 |
| 模型 Provider | 已在 DSH 中配置 | 驱动 Agent 会话 |

另外需要：

- 一个 GitHub 仓库，作为 Superagent 的工作对象。
- 一个**独立的 GitHub 账号**作为 Agent 账号，并拥有该仓库的写权限。不要让 Agent 使用你的个人账号。
- 可选：飞书自建应用（机器人）与 `lark-cli`，用于从飞书接收需求。

先检查本机环境：

```sh
node -v
pnpm -v
git --version
python3 --version
tmux -V
gh --version | head -1
dsh --version
```

## 2. 方式一：让编码 Agent 协助安装（推荐）

官方推荐的安装方式是由编码 Agent 按 `SETUP.md` 与操作者逐步完成。把下面这句话发给你的编码 Agent：

> Read SETUP.md in this repository and set up dsh-superagent with me.

Agent 会先做一次环境探测，然后逐个询问来源（GitHub / 飞书）、仓库、Agent 账号、允许名单、工单目录和提交身份等问题；给出完整计划并确认后，再执行安装、配置和验证。

## 3. 方式二：手动安装

### 3.1 安装 DeepSeek Harness

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
dsh --version
```

版本需满足 Superagent `package.json` 中 `engines.dsh` 的范围。安装完成后，先在 DSH 中配置好模型 Provider；否则 Superagent 启动后没有可用模型。

### 3.2 获取并构建 Superagent

```sh
git clone https://github.com/deepseek-ai/dsh-superagent-prototype.git dsh-superagent
cd dsh-superagent
pnpm install
pnpm build
```

### 3.3 创建独立 Profile 并安装 Bundle

Superagent 使用独立于日常工作环境的 DSH Profile，默认名为 `superagent`，这样它的重启和升级不会影响你正在使用的会话。

```sh
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

- 第一条命令从 DSH 的 Web Profile 模板创建 `~/.dsh/profiles/superagent`。
- 第二条命令把当前目录的 Superagent Bundle 以 `link:` 方式安装进该 Profile。
- 如需把 DSH 数据放在别处，先设置 `DSH_HOME` 环境变量。
- `--from-default-profile` 只用于首次安装。已有 Profile 时再次执行会覆盖并删除其 `cordis.patch.yml`，升级时不要运行。

### 3.4 写入机器配置

复制示例配置并替换所有 `<placeholder>`：

```sh
cp examples/profile.cordis.patch.yml ~/.dsh/profiles/superagent/cordis.patch.yml
```

最小配置示例：

```yaml
# ~/.dsh/profiles/superagent/cordis.patch.yml
- id: webserver
  config:
    host: 127.0.0.1
    port: 3082
- id: dsh-superagent
  config:
    # 必须是绝对路径；数据库、锁、命令 shim、工单目录和 Agent 的 git 配置都在这里
    stateDir: /home/<user>/.dsh/state/superagent
# 可选：启用飞书 Source
# - id: dsh-superagent-feishu
#   disabled: false
```

同时在该文件中保留或写入你的模型 Provider 配置（`llm-*` 与 `agent-default-model`）。`stateDir` 之后不能在面板中修改，升级前记得备份其中的 `orchestrator.sqlite`。

### 3.5 准备 Agent 账号与 CLI

#### GitHub Agent 账号

创建独立的 GitHub 账号，并给它目标仓库的写权限。然后创建一个只使用该账号凭据的 `gh` 包装脚本：

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

最后一条命令应输出 Agent 账号的登录名。Superagent 也会用同样的方式读取该账号。

#### 飞书机器人（可选）

创建带机器人能力的自建应用，授予接收消息事件、读取和发送群消息、读取群成员的权限，并发布应用版本。然后为机器人准备独立的 `lark-cli` 配置和包装脚本：

```sh
LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" \
  lark-cli config init --app-id <cli_…> --app-secret-stdin
LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" \
  lark-cli config strict-mode bot --global

cat > ~/.local/bin/lark-bot <<'EOF'
#!/bin/sh
export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent"
exec lark-cli "$@"
EOF
chmod +x ~/.local/bin/lark-bot
```

第一条命令从标准输入读取 App Secret。请使用独立配置目录，避免与你个人的 `lark-cli` 登录混用；Superagent 会拒绝仍能以用户身份操作的 CLI。

### 3.6 启动与检查

在 Superagent 代码目录中执行：

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

`start` 会在 tmux 会话 `dsh-superagent` 中启动 `dsh --profile superagent`，并最多等待一分钟确认运行状态。`status` 的退出码含义：

| 退出码 | 含义 |
| --- | --- |
| 0 | 正常运行 |
| 1 | 未运行、重复运行、无响应或运行时错误，重启可能修复 |
| 2 | 运行中，但某个 Source 或 Policy 同步出错，重启不能修复 |
| 3 | 运行中，但仍在等待配置完成，重启不能修复 |

启动后打开 `http://127.0.0.1:3082`，在侧边栏选择 **Superagent**。

### 3.7 在面板中完成首次配置

1. **Sources**
   - GitHub：填写仓库 `owner/name` 和 Agent CLI 的绝对路径，例如 `/home/<user>/.local/bin/gh-agent`。
   - 飞书（可选）：填写机器人 CLI、要监听的群、监听模式（仅 @ 或全部消息），以及是否接收私聊。
2. **Policy**
   - 在 GitHub 下加入允许提需求的账号；列表为空时任何请求都不会变成工单。
   - 使用飞书时，可指定一个群自动同步成员名单。
3. **Execution**
   - 设置工单目录根路径。
   - 设置 prepare 命令，为每个工单创建独立 checkout。示例：

     ```sh
     git -C /home/<user>/src/<repo> fetch --no-tags \
       "https://github.com/$DSH_SUPERAGENT_GITHUB_REPOSITORY.git" \
       "${DSH_SUPERAGENT_GITHUB_PR_HEAD:-HEAD}:refs/dsh-superagent/$DSH_SUPERAGENT_TICKET" &&
     git -C /home/<user>/src/<repo> worktree add \
       -b "agent/$DSH_SUPERAGENT_NAME" \
       "$DSH_SUPERAGENT_DIR" \
       "refs/dsh-superagent/$DSH_SUPERAGENT_TICKET"
     ```

   - 把你个人的 CLI（例如登录了你本人账号的 `gh`）加入 blocked commands，避免 Agent 误用。
4. **Prompts**
   - 设置团队规则：提交作者身份、是否允许自动合并、GitHub 披露文案和需要关注的 review bot 等。

设置保存后，Superagent 会自动做一次 setup 检查；检查通过后才开始轮询和工作。

## 4. 验证安装

1. `scripts/dsh-superagent status` 退出码为 `0`，且没有 setup 问题。
2. 用一个在允许名单中的 GitHub 账号，在目标仓库的 Issue 或评论中写：

   ```text
   @<agent-login> <你的需求>
   ```

3. 大约一个轮询周期（默认一分钟）后，面板的 Overview 中应出现对应工单；Agent 会先确认，再在独立会话中执行并回复。

注意：通过编辑 Issue 正文新增的 @ 不会被接收，请发一条新评论。

## 5. 日常操作与升级

```sh
scripts/dsh-superagent start
scripts/dsh-superagent stop
scripts/dsh-superagent restart
scripts/dsh-superagent status
```

- 日志和控制文件位于 `.artifacts/`，启动日志保留最近 20 份。
- 建议按官方 [Hourly check](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/006-deployment.md#hourly-check) 一节，在**日常使用的 DSH Profile** 中创建每小时检查任务；不要把它建在 `superagent` Profile 内。

升级步骤：

```sh
# 1. 确认没有正在执行的工单
# 2. 备份 stateDir/orchestrator.sqlite
git pull
pnpm install
pnpm build
scripts/dsh-superagent restart
scripts/dsh-superagent status
```

新版本迁移过的数据库不能被旧版本打开，回滚必须使用升级前的备份。升级 DSH 本身后，也要重启 Superagent，让面板加载匹配的前端资源。

## 6. 常见问题

| 现象 | 处理 |
| --- | --- |
| `The agent GitHub CLI is not found.` | 在 GitHub Source 中填写 CLI 的绝对路径，或确保命令在 `PATH` 中 |
| `The agent GitHub CLI cannot read its account` | 对包装脚本执行 `gh-agent auth login` |
| `@… cannot push to …` | 给 Agent 账号目标仓库的写权限 |
| `status` 退出码 2 | 查看输出中的 `errors`，按具体 Source / 同步项修复；重启无效 |
| `status` 退出码 3 | 查看 `setup` 给出的缺失项，到面板对应位置补全 |
| 面板提示插件加载失败 | 升级 DSH 后重启正在运行的 Superagent 实例 |
| @ 了 Agent 但没有反应 | 检查发送者是否在允许名单、Source 是否开启、处理是否被暂停；Issue 正文中的 @ 不会被接收 |

更完整的排错表见官方 [docs/006-deployment.md](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/006-deployment.md#troubleshooting)。

## 7. 参考

- 项目主页：<https://github.com/deepseek-ai/dsh-superagent-prototype>
- 安装向导：[SETUP.md](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/SETUP.md)
- 部署、配置与升级：[docs/006-deployment.md](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/006-deployment.md)
- 设计文档：[docs/000-philosophy.md](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/000-philosophy.md) 起按编号阅读
- DeepSeek Harness：<https://www.deepseek.com/en/harness/>
