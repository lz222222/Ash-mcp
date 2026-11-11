# Ash MCP 使用手册

三个自建 MCP 的接入与使用说明：**小红书**、**网易云（Cove Resonance）**、**心潮·念**。

三个服务都跑在本机，通过**出站隧道**接进 ChatGPT，不需要公网端口，也不需要 VPS。

---

## 目录

- [整体架构](#整体架构)
- [关于域名](#关于域名)
- [通用：在 ChatGPT 里添加连接器](#通用在-chatgpt-里添加连接器)
- [1. 小红书 MCP](#1-小红书-mcp)
- [2. 网易云 MCP（Cove Resonance）](#2-网易云-mpccove-resonance)
- [3. 心潮·念 MCP](#3-心潮念-mcp)
- [端口总表](#端口总表)
- [安全约定](#安全约定)

---

## 整体架构

```text
ChatGPT 连接器
      │  HTTPS（出站）
      ▼
Cloudflare 临时隧道 / OpenAI Secure MCP Tunnel
      │  本机回环
      ▼
本机 MCP 服务（18060 / 8787 / 18110）
      │
      ├─ 无头浏览器（小红书）
      ├─ 网易云一起听
      └─ Ombre Brain 记忆库（18001）
```

三个服务互不依赖，可以单独启停。端口、配置、日志都在各自目录里，不共用。

---

## 关于域名

目前用的是 Cloudflare 临时隧道，**每次重启地址都会变**，ChatGPT 里的连接器也要跟着改。

想固定下来，建议注册一个国内可用的域名：

| 候选 | 说明 |
| --- | --- |
| `ashmcp.cn`（推荐） | 短、和仓库同名，国内注册约 30–40 元/年，需实名 |
| `ash-mcp.cn` | 可读性更好，稍长 |
| `ashmcp.com` | 国际通用，价格略高 |

拿到域名后，把 Cloudflare Tunnel 换成**命名隧道**，用子域名分别指过去：

```text
xhs.ashmcp.cn   →  http://127.0.0.1:18060
music.ashmcp.cn →  http://127.0.0.1:8787
mind.ashmcp.cn  →  http://127.0.0.1:18110
```

不买域名也能用，只是每次重启要换一次地址。

> `.cn` 域名需要实名认证；如果解析到 Cloudflare，建议先确认注册商支持修改 NS。不想折腾就继续用临时隧道。

---

## 通用：在 ChatGPT 里添加连接器

1. ChatGPT → 设置 → **连接器** → 高级设置 → 打开**开发者模式**
2. 添加**自定义连接器**
3. 名称随意，URL 填对应的 `/mcp` 地址
4. 认证方式选 **无**
5. 在对话里启用这个连接器，模型才能看到工具

连接器添加后，第一次调用可能会让你确认一次。之后在对话里直接说需求即可。

---

## 1. 小红书 MCP

让 AI 直接逛小红书：搜索、读笔记、看评论、点赞收藏、发图文和视频。

### 基本信息

| 项 | 值 |
| --- | --- |
| 目录 | `xiaohongshu-mcp/` |
| 程序 | `bin/xiaohongshu-mcp-windows-amd64.exe` |
| 本地端口 | `18060` |
| MCP 路径 | `/mcp` |
| 登录态 | `cookies.json`（含指纹 seed） |
| 内置浏览器 | `%LOCALAPPDATA%\xiaohongshu-mcp\browser\148.0.7778.215\` |

### 首次登录

```powershell
cd xiaohongshu-mcp
.\bin\xiaohongshu-login-windows-amd64.exe
```

会弹出一个**非无头**浏览器窗口，用手机小红书扫码。登录成功后 cookies 自动写入 `cookies.json`，窗口自己关掉。

如果扫码后窗口一直不退出（页面跳到个人主页，工具没识别到），直接用调试端口把 cookies 导出来：

```powershell
# 读取 chrome 的 DevToolsActivePort 拿到端口号，然后
node grab-cookies.js <端口> cookies.json 1393162135
```

`grab-cookies.js` 会通过 CDP 的 `Storage.getCookies` 拉取全部小红书 cookie，写成工具认识的 v2 格式。

### 启动 / 停止

```powershell
cd xiaohongshu-mcp
.\start-xhs.ps1     # 起服务 + 起隧道，输出公网 /mcp 地址
.\stop-xhs.ps1      # 全部停掉
```

启动脚本做三件事：拉起无头服务 → 等 18060 监听 → 起 cloudflared 并把地址写进 `data/public-url.txt`。

### 必须注意：Host 头

服务端用的是官方 Go SDK，自带 **DNS rebinding 防护**，只接受 `localhost` / `127.0.0.1` 的 Host。隧道透传过来的域名会被拒绝：

```text
Forbidden: invalid Host header "xxx.trycloudflare.com"
```

所以 cloudflared 必须带上 Host 改写：

```powershell
cloudflared tunnel --url http://127.0.0.1:18060 --http-host-header 127.0.0.1:18060
```

`start-xhs.ps1` 里已经写好了。

### 工具（18 个）

| 分类 | 工具 |
| --- | --- |
| 登录 | `check_login_status`、`get_login_qrcode`、`delete_cookies` |
| 浏览 | `list_feeds`、`search_feeds`、`get_feed_detail` |
| 用户 | `get_my_profile`、`user_profile` |
| 互动 | `like_feed`、`favorite_feed`、`post_comment_to_feed`、`reply_comment_in_feed` |
| 通知 | `get_unread_count`、`list_notifications`、`like_notification`、`reply_notification` |
| 发布 | `publish_content`、`publish_with_video` |

`feed_id` 和 `xsec_token` 是一对，都要从列表或搜索接口拿，不能自己拼。

### 常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 启动卡在「下载内置浏览器」 | 从 `cdn.one-world.ai` 拉 190MB，国内很慢 | 手动下载后解压到缓存目录，见上表路径 |
| `ERR_PROXY_CONNECTION_FAILED` | 系统代理指向了没开的代理软件 | 关闭系统代理，或确保代理在运行 |
| 公网访问 403 `invalid Host header` | 隧道没改写 Host | 加 `--http-host-header` |
| 工具报未登录 | `cookies.json` 过期 | 重新扫码或重新导出 cookies |

内置浏览器带指纹补丁，**不要用本机 Edge/Chrome 替换**，否则指纹参数会被静默忽略。

---

## 2. 网易云 MCP（Cove Resonance）

把网易云「一起听」接进 ChatGPT：读聊天室消息、看播放状态、读歌词、控歌，再把回复送回房间。

### 基本信息

| 项 | 值 |
| --- | --- |
| 目录 | `cove-resonance/` |
| 本地端口 | `8787` |
| MCP 路径 | `/mcp/music`（兼容 `/mcp`） |
| 配置 | `.env` |
| 启动 | `start-cove.ps1` / `stop-cove.ps1` |

### 配置 `.env`

```env
PORT=8787
BRIDGE_PUBLIC_ORIGIN=https://<你的隧道地址>   # 不带 /mcp，启动脚本会自动写回
TOGETHER_ENABLED=true
TOGETHER_POLL_INTERVAL_MS=4000
TOGETHER_HEARTBEAT_INTERVAL_MS=10000
NETEASE_COOKIE=MUSIC_U=<你自己的会话 cookie>
```

`BRIDGE_PUBLIC_ORIGIN` 必须和当前隧道地址一致，否则 MCP App 的 CSP 和 SSE 会挂。`start-cove.ps1` 每次启动会重新拉隧道并自动更新这一行。

### 启动 / 停止

```powershell
cd cove-resonance
.\start-cove.ps1        # 先起 cloudflared，再起 node 服务
.\stop-cove.ps1
```

只想重启服务、保留当前隧道：

```powershell
.\start-cove.ps1 -SkipTunnel
```

### 两种监听方式（二选一）

- **Long-wait（推荐，多端可用）**：模型主动调用 `cove_bridge_wait` 挂着等消息，拿到事件后 ACK，处理完再继续等。
- **Widget Listener**：靠前端挂件推事件，网页端会弹人工确认，桌面端不可用。

Long-wait 的正确循环：

```text
cove_bridge_wait  →  hasEvent=true
cove_bridge_wait_ack(eventId)
处理事件（modelContext / visibleText）
cove_bridge_reply(回复内容, replyRoute=netease.chatroom)
再次 cove_bridge_wait（用户要求继续听时）
```

### 工具（26 个）

| 分类 | 工具 |
| --- | --- |
| Bridge | `cove_bridge_wait`、`cove_bridge_wait_ack`、`cove_bridge_reply`、`cove_bridge_release`、`cove_bridge_sync`、`cove_bridge_delivered`、`cove_bridge_dismissed`、`cove_bridge_listener_session` |
| 一起听 | `netease_together_now`、`netease_together_realtime_status`、`netease_together_pause`、`netease_together_resume`、`netease_together_next`、`netease_together_goto`、`netease_together_enqueue_next`、`netease_together_send_message`、`netease_together_leave` |
| 账号 | `netease_search_songs`、`netease_my_playlists`、`netease_playlist_tracks`、`netease_create_playlist`、`netease_add_to_playlist`、`netease_remove_from_playlist`、`netease_like_song`、`netease_play_history`、`netease_daily_recommend` |

### 常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 手机上发的消息 ChatGPT 看不到 | 模型没有主动调 `cove_bridge_wait` | 明确要求它「用 45 秒超时循环拉取」 |
| 服务端有消息、客户端一直空 | 隧道地址变了，`BRIDGE_PUBLIC_ORIGIN` 还是旧的 | 重新跑 `start-cove.ps1` |
| 回复发不出去 | 没先 ACK，或 `replyRoute` 没填 `netease.chatroom` | 按上面的循环顺序来 |
| 切歌「假成功」 | 目标歌不在当前一起听队列 | 先确认队列，再用 `netease_together_goto` |

---

## 3. 心潮·念 MCP

让 AI 有连续的**内在状态**（驱力、情绪、挂念）和**长期记忆**，不是每次对话从零开始。

### 组成

| 组件 | 端口 | 作用 |
| --- | --- | --- |
| 心潮 | `18110` | 驱力 / 情绪 / 觉察 / 此刻 |
| Ombre Brain | `18001` | 记忆库：breath、hold、dream、trace |
| 隧道客户端 | 健康检查 `8081` | OpenAI Secure MCP Tunnel，出站接入 ChatGPT |
| 本地面板 | `8090` | 查看状态 |
| SOCKS / HTTP 代理 | `1080` / `8888` | 隧道控制面出站用 |

### MCP 地址

```text
https://<隧道域名>/mcp/<MCP_PATH_TOKEN>
```

路径里那段就是 `.env` 里的 `MCP_PATH_TOKEN`，相当于一把钥匙。隧道本身不做 OAuth（`OAUTH_ENABLED=false`），靠路径令牌 + 隧道凭据。

### 启动 / 停止

```powershell
cd xinchao-nian
.\start-all.ps1     # SOCKS → HTTP 代理 → OB → 心潮 → 隧道 → 面板 → 看门狗
.\stop-all.ps1
```

启动脚本会自检：心潮 `/health` 的版本与模式、隧道的 `live/ready/MCP 探活/控制面` 状态。

单独重启记忆库：

```powershell
.\restart-ombre.ps1
```

### 看门狗

`watchdog-ombre.ps1` 每 30 秒检查一次：

- 18001 端口是否在监听
- 近 180 秒日志里是否出现 `Connection error` 一类关键字

命中就自动重启 Ombre Brain，两次重启至少间隔 3 分钟，避免连环重启。日志在 `data/watchdog.log`。

### 工具（10 个）

| 工具 | 作用 |
| --- | --- |
| `xinchao_context` | 取当前内在状态与「你不在的时候」发生了什么 |
| `xinchao_event` | 上报一次互动，让心潮判断类型与轻重 |
| `xinchao_box` | 只有 AI 能看的黑匣子，可到期、露头、提醒 |
| `xinchao_cabin_note` / `xinchao_cabin_inbox` | 写信与查看收到的信 |
| `xinchao_awareness` | 每日觉察候选的确认或放下 |
| `xinchao_anchor_update` | 更新性格锚点 |
| `xinchao_personality_reflect` / `xinchao_personality_stats` | 性格复盘与统计 |
| `xinchao_handoff_note` | 交接便签 |

### 常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 日记写入超时 | Ombre Brain 跑久了连接池僵死 | `restart-ombre.ps1`，看门狗会兜底 |
| 隧道报 429 / 配置受速率限制 | 隧道客户端没起，或探活失败 | 看 `tunnel\err.log`，重启 `start-tunnel.ps1` |
| 401 权限错误 | API Key 与隧道所属组织不匹配，或隧道 ID 错 | 核对 `tunnel\.env` |
| 手机网页连不上心潮 | 地址填成了 MCP 地址，或用了 `localhost` | 基础地址只填到域名，MCP 才加 `/mcp/<token>` |

心潮、Ombre、隧道三者是**串行启动**的，前一个没起来后一个会报错。启动失败先看 `data\` 下对应的 `err.log`。

---

## 端口总表

| 端口 | 服务 | 说明 |
| --- | --- | --- |
| `18060` | 小红书 MCP | HTTP，`/mcp` |
| `8787` | 网易云 Bridge | HTTP，`/mcp/music` |
| `18110` | 心潮 | HTTP，`/mcp/<token>` |
| `18001` | Ombre Brain | 内部记忆后端 |
| `8090` | 心潮面板 | 本机查看 |
| `8081` | 隧道健康检查 | `live` / `ready` |
| `1080` / `8888` | SOCKS / HTTP 代理 | 隧道控制面出站 |

---

## 安全约定

- **不要提交** `cookies.json`、`.env`、`tunnel.env`、任何 token 或 `MUSIC_U`。
- 临时隧道地址是公开的，谁拿到 URL 都能调工具；用完就停，别长期挂着。
- 小红书内置浏览器带指纹补丁，换浏览器等于换指纹，容易被风控。
- 本手册只写结构和用法，不含任何真实密钥；换机器时按各目录的 `.env.example` 重新填。
