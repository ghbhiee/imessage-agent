# imessage-agent

[English](README.md) | 中文

**在 iMessage 里指挥你的 AI Agent——用你自己的 Apple ID、在你自己的 Mac 上。**

## 这是什么

在 iPhone 上给一个「Agent 联系人」发 iMessage，Mac 上的 Agent 收到、干活、再用**你自己的 Apple ID** 回你。

```
iPhone「信息」 ──▶ iCloud 同步 ──▶ Mac 的 chat.db ──▶ bridge.py ──▶ Claude / dsh
      ▲                                                              │
      └──────────── Messages.app（AppleScript，你的账号）◀────────────┘
```

- 一个 Apple ID、一台 Mac、一个 macOS 用户，你和 Agent 共用。
- 每个会话（单聊或群）可以接不同的 Agent：比如联系人 A → Claude，联系人 B → dsh。
- 白名单路由：只有配置里列出的会话会被处理，其它会话只读不回。
- Claude 的权限审批也走 iMessage：回 `1` 允许、`2` 总是允许、`3` 拒绝。
- dsh 的会话可以直接放在 dsh 网页版里，网页和手机看到的是同一个会话。

## 为什么这么做

### 1. 为什么不给 Agent 单独开一个 Apple ID

最直观的做法是给 Agent 一个自己的 Apple ID，你和它像两个人一样聊天。但这样做的 Agent 是「另一个人」：

- 它要**以你的身份**办事：替你回消息、用你登录着的网站和 App、读写你的文件。换个账号就做不到了。
- 它和你**共用这台电脑**。一台 Mac 的一个 macOS 用户里，「信息」只能登录一个 Apple ID；要两个账号同时在线，就得再开一个 macOS 用户或者虚拟机，而 Agent 在另一个用户里就碰不到你的东西了。

所以这里的前提是：**Agent 就是你**，只是多了一只手。

### 2. 同一个账号，怎么分清哪句是你说的、哪句是 Agent 说的

在 `chat.db` 里，你在 iPhone 上打的字和 Agent 在 Mac 上发的消息，都是 `is_from_me = 1`，没有任何字段能区分来自哪台设备。

常见的做法是「按文本猜」：记住 Agent 刚发过什么，看到一样的就当回声丢掉，再加个限速兜底。实测这会吞掉你真正发的消息，也会重复回复。

这里用的是**确定性规则**：

- Agent 发出的每条消息都带固定前缀（Claude 默认 `🤖 `，dsh 默认 `🐳 `）。带前缀的一律跳过，不带的就是你。
- 另外再记最近 200 条自己发出的文本指纹，作为第二道保险。
- 没有限速，不会吞消息。

### 3. 为什么不直接「发给自己」

iMessage 发给自己（自己的手机号或邮箱）是可以的，本项目也支持（`self_chat`）。但它有个副作用：**每条消息在 `chat.db` 里有两行**：

| 行 | `is_from_me` | `reply_to_guid` |
|---|---|---|
| 本体 | 1 | （上一条） |
| 回声 | 0 | 指向本体的 guid |

桥接会把回声折成本体只处理一次（`reply_to_guid` 是确定性的关联），但 Mac 上的「信息」会把每条都显示两遍，看着很乱。

### 4. 推荐做法：一个「沉默的对方」

找一个**你控制、但平时不会说话**的 iMessage 地址，建一个联系人（比如叫「Claude」），跟它单聊：

- 你发的消息：`is_from_me = 1`，只有一行，没有回声。
- Agent 的回复：也是你的账号发的，带 🤖 前缀。
- 对方永远不说话，等于你在一个干净的对话里自问自答。

对方地址的选择：

- **最好用一个独立 Apple ID 的邮箱**：邮箱不会被运营商回收，也不会退化成短信。
- 用手机号也行，但停用的号码可能被运营商重新放号，新机主就会收到你和 Agent 的全部对话；号码长期不在线时，消息也可能变成短信。
- 想接多个 Agent，就准备多个这样的地址，每个对应一个 Agent。

### 5. iMessage 的投递机制（踩过的坑）

- 能不能给某个地址发 iMessage，取决于这个地址**有没有在苹果那里注册 iMessage**：至少有一台设备用该 Apple ID 登录了「信息」，并在「发送与接收」里勾选了这个地址。光注册 Apple ID 不够。
- 地址没注册时，iPhone 在查询阶段就判定「不是 iMessage」，直接按**短信**发。邮箱收不了短信，于是消息停在「未发送」。这时 `chat.db` 里是 `service = 'SMS'`、`is_sent = 0`、`error = 0`，**没有错误码**。
- 「设置 → 信息 → 以短信发送」只管「iMessage 发失败后要不要改发短信」，管不到查询阶段的判断。
- 判断方法：在 iPhone 新建信息，输入对方地址，变蓝就是已注册，变红/变灰就是没注册。
- 一个地址可能在你给账号加/换手机号、重新验证时被从 iMessage 地址里去掉。到「设置 → Apple 账户 → 登录与安全 → 电子邮件与电话号码」重新添加、验证，再去「发送与接收」勾上。
- 群聊没有回声，只有一份；但 iMessage 会按你用手机号还是邮箱发，把同一个群分裂成两个 `chat guid`，两个都要加进白名单。

### 6. 权限：为什么用 Agent SDK 而不是 `claude -p`

`claude -p` 没有人能点「允许」，规则外的工具会被直接拒绝。这里用 Python 版 Claude Agent SDK：

- `setting_sources = user/project/local`：你 `~/.claude/settings.json` 里的放行规则照常生效，不会每次重置。
- 规则外的操作走 `can_use_tool` 回调，在 `approval_chat` 里发「[权限 #N] … 回 1/2/3」。选 `2` 会把 CLI 给出的建议规则写回配置，以后不再问。超时按拒绝。
- 审批回答只认你自己在审批会话里的消息，群里别人回 1 无效。
- 默认可以用 `auto` 模式：日常操作自动放行，危险的才问你。`/mode` 随时切换。

### 7. dsh：为什么要通过网页版插件，而不是 `dsh headless`

`dsh headless --session-id` 可以续接会话，但如果同时开着 `dsh web`，网页版会接管它看得到的每个会话的写锁并且不释放，下一轮续接就报 `already owned by an active write handle`，只能每轮新开会话。

两种解决办法都实现了：

- `agent: "dsh"`：headless CLI，用 `dsh-sessions.patch.yml` 把会话存到单独目录（`$DSH_HOME/sessions-imessage`），网页版看不到就不会抢锁。代价是网页上看不到这些会话。
- `agent: "dsh-web"`（推荐）：通过 [dsh-plugin-discord](https://github.com/ghbhiee/dsh-plugin-discord) 的 `/plugins/imessage` 接口，在网页版进程里建会话、发消息。它复用了 Discord 插件管会话的同一套代码（`SessionBridge`），会话标题是 `[iMessage] MM-DD HH:MM`，网页上能直接看到、接着聊；`/new`、`/sessions`、`/use <id>`、`/current`、`/stop` 和 Discord 里一样用。

### 8. 完全磁盘访问权限：为什么要单独复制一份 Python

读 `~/Library/Messages/chat.db` 需要「完全磁盘访问权限」，launchd 启动的进程，权限挂在它启动的那个可执行文件上。如果直接用 uv 共享的 Python，等于给这台机器上所有用这个 Python 的脚本都开了全盘权限。所以把 Python 解释器复制一份到 `runtime/`，只给这一份开。

## 安装

要求：macOS、已登录 iMessage 的「信息」、[uv](https://docs.astral.sh/uv/)、Claude Code CLI（用 Claude 时）、dsh（用 dsh 时）。

```bash
git clone https://github.com/ghbhiee/imessage-agent.git ~/imessage-agent
cd ~/imessage-agent

# 专用 Python（只给它开完全磁盘访问权限）
uv python install 3.12
cp -R "$(dirname "$(dirname "$(uv python find 3.12)")")" runtime
uv venv --python runtime/bin/python3.12
uv sync

cp config.example.json config.json
.venv/bin/python bridge.py --chats      # 列出会话和 chat guid，填进 config.json
```

1. 系统设置 → 隐私与安全性 → 完全磁盘访问权限 → `+` → 选 `runtime/bin/python3.12`。
2. 前台试跑：`.venv/bin/python bridge.py`。第一次发消息时 macOS 会问「想控制 信息」，点允许。
3. 常驻：把 `com.example.imessage-agent.plist` 里的 `YOU` 换成你的用户名，放进 `~/Library/LaunchAgents/`，然后
   `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.example.imessage-agent.plist`。
   改完配置后重启：`launchctl kickstart -k gui/$(id -u)/com.example.imessage-agent`。

`.venv/bin/python bridge.py --send "hi"` 往 `self_chat` 发一条测试消息。

## 配置（config.json）

| 字段 | 说明 |
|---|---|
| `chats` | 白名单，key 是 chat guid（`--chats` 列出来的） |
| `chats.*.agent` | `claude`（默认）/ `dsh` / `dsh-web` |
| `chats.*.allow_from` | 谁能指挥它：`["me"]` 只听你；也可加对方号码/邮箱，或 `"*"`。**别人能指挥的 Agent 拿的是你的全部权限**，慎加 |
| `chats.*.trigger` | 群里必须以它开头才算指令（如 `"@c"`）；`null` = 每条都算 |
| `chats.*.prefix` / `cwd` / `model` / `dsh_profile` | 按会话覆盖 |
| `self_chat` | 发给自己的会话 guid（用于回声去重；不想路由就别放进 `chats`） |
| `approval_chat` | Claude 的权限请求发到哪个会话 |
| `agent_prefix` | Claude 回复的前缀，默认 `🤖 ` |
| `permission_mode` | Claude 默认权限模式：`default` / `acceptEdits` / `auto` / `plan` / … |
| `max_backlog_age_s` | 桥接重启后，超过这个秒数的旧消息不再处理 |
| `dsh_web_url` / `dsh_web_secret_file` | dsh-web 后端的地址和密钥文件 |

## 会话里的命令

| 命令 | Claude | dsh-web |
|---|---|---|
| `/new` | 新会话 | 新会话（网页上会出现新的 `[iMessage]` 会话） |
| `/stop` | 中断当前任务，拒绝待审批 | 取消当前回合 |
| `/status` | 状态、排队、会话、权限模式 | — |
| `/mode <模式>` | 切换权限模式 | — |
| `/sessions` `/use <id>` `/current` | — | 列出 / 接管 / 查看会话 |
| `/help` | 帮助 | 帮助 |

## 已知限制

- iPhone 上 Agent 的回复也显示在「我」这一侧，只能靠前缀区分。
- 只发最终回复，不发中间过程。长回复按 3000 字切分。
- Claude 每轮会加载你全部的 user 设置、skills 和 MCP，最简单的一轮也要约 $0.15；可按会话指定 `model`。
- 轮询 `chat.db`（默认 1 秒一次），不是事件推送。
- 依赖 `attributedBody` 的 typedstream 解码（新版 macOS 的 `text` 列常为空），苹果改格式时可能需要更新。

## License

MIT
