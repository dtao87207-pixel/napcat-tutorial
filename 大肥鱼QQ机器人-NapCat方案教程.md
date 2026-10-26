# 用 AstrBot + NapCat 小号，在 QQ 上养一只「大肥鱼」

> 一份从零开始的实操教程。全部命令、配置项、报错都来自真实踩坑记录，不是文档抄写。
> 目标：让一个**普通 QQ 小号**变成一个会聊天、会看图、会发表情包、会主动找你的「活人」，而不是只会 @ 才应答的问答机。

---

## 0. 先说清楚：三条路线，为什么选第三条

社区里做 QQ 机器人，绕不开三种接入方式。先把它们的真实差异摊开，省得你走弯路。

| | **A. 官方机器人**（QQ 开放平台） | **B. 桥接插件**（qq-bridge 类） | **C. NapCat 小号**（本教程） |
|---|---|---|---|
| 本质 | q.qq.com 注册的 Bot，走官方 API | 独立进程，把 DSH/AstrBot 会话桥接到 OneBot | 注入本机 QQ，当普通账号用 |
| 群聊能不能收全量消息 | ❌ 个人主体不行 | 取决于底层 | ✅ 默认全量 |
| 必须 @ 才回话 | ❌ 是（个人主体硬限制） | 看实现 | ✅ 不用 |
| 账号风险 | 无（官方） | 中 | 中高（风控） |
| 审核/资质 | 需要，个人主体功能受限 | 不需要 | 不需要 |
| 主动私聊陌生人 | 不允许 | 受限 | 可以（但别作死） |

### A 方案为什么废了

这不是配置问题，是腾讯的资质限制。官方 FAQ 原文：

> 企业主体公开使用已开放，个人主体暂不支持公开使用。个人主体的机器人如需添加群聊，机器人创建者可直接添加自己是群主/管理员身份的群聊（升级最新版移动QQ添加）。

更要命的是群里「接收全量消息 / 主动推送消息」这两个开关，**只有群主在添加机器人时点「设置权限」才能勾**，管理员都点不了。结果就是：你 @ 她，她回；你不 @，服务器根本不把消息推给你。所谓「主动插话」在官方路线上是残缺的。

### C 方案的本质

NapCat 直接在**本机跑一个 headless QQ**，用 OneBot v11 协议把消息吐出来。对服务器来说，这就是一个普通用户在讲话——所以全量消息、群聊、私聊、主动发消息，全都没有平台级限制。

代价是：小号可能被风控（掉线、验证码、甚至封号），所以**必须用小号，不要用主号**。

---

## 1. 架构

```
┌─────────────┐   OneBot v11 反向 WS    ┌──────────────┐
│  NapCat     │ ──────────────────────► │   AstrBot    │
│  (headless  │   ws://127.0.0.1:6199   │  (python)    │
│   小号 QQ)  │ ◄────────────────────── │  :6185 面板  │
└─────────────┘      收发消息            └──────┬───────┘
                                               │
                          ┌────────────────────┼────────────────────┐
                          ▼                    ▼                    ▼
                   DeepSeek API          智谱 GLM-4V          表情包 / 陪伴插件
                   （对话、工具）        （看图、GIF 抽帧）    （主动开口、记忆）
```

关键点：**是 AstrBot 当服务端，NapCat 主动连它**（反向 WS）。所以先起 AstrBot，再起 NapCat，顺序反了会一直重连。

---

## 2. 准备

| 项 | 要求 | 说明 |
|---|---|---|
| 系统 | Windows 10/11 x64 | 本教程命令为 PowerShell |
| Python | 3.12.x | AstrBot 4.29 要求 `>=3.12` |
| 磁盘 | ≥ 5 GB | QQ 本体 ~1.1 GB + 依赖 + 表情包 |
| QQ 号 | **小号**，最好有手机号能接验证码 | 主号出事你会哭 |
| 模型 | DeepSeek API Key（对话）、智谱 API Key（视觉） | 都按量计费，月成本个位数到几十元 |
| 网络 | 能访问 PyPI；GitHub 建议走加速 | 见附录 A |

不需要的东西：服务器、公网 IP、域名、备案。全部跑在本地。

---

## 3. 第一步：装 AstrBot（源码 + venv）

不要用 installer/launcher 静默安装——那样经常装完找不到装哪去了，UAC 一提权就失败。源码 + 虚拟环境最可控。

```powershell
# 1) 拉源码（GitHub 慢就换加速前缀，见附录 A）
$zip = "$env:USERPROFILE\Downloads\astrbot.zip"
curl.exe -L "https://gh-proxy.com/https://codeload.github.com/AstrBotDevs/AstrBot/zip/refs/heads/master" -o $zip
Expand-Archive $zip -DestinationPath "$env:USERPROFILE\.astrbot-src" -Force

# 2) 建 venv（用一个干净的 3.12 解释器路径）
python -m venv "$env:USERPROFILE\.astrbot-venv"

# 3) 装依赖（清华源，比官方快十倍）
& "$env:USERPROFILE\.astrbot-venv\Scripts\python.exe" -m pip install --no-cache-dir `
  -i https://pypi.tuna.tsinghua.edu.cn/simple `
  -r "$env:USERPROFILE\.astrbot-src\AstrBot-master\requirements.txt"
```

依赖约 130+ 个包（faiss-cpu、pandas、py-cord、qq-botpy 等），装完 1–3 GB。

### 3.1 首次启动与面板密码

```powershell
$src = "$env:USERPROFILE\.astrbot-src\AstrBot-master"
Start-Process "$env:USERPROFILE\.astrbot-venv\Scripts\python.exe" `
  -ArgumentList "main.py" -WorkingDirectory $src -WindowStyle Hidden `
  -RedirectStandardOutput "$src\run.log" -RedirectStandardError "$src\err.log"
```

首次启动会自动从 `astrbot-registry.soulter.top` 拉 WebUI（约 5 MB），然后监听 **6185**。

> ⚠️ **坑 1：没有默认密码。** AstrBot v4 把默认的 `astrbot/astrbot` 换成了随机强密码，但**日志里不打印**，面板直接登不进去。

解决办法是直接改配置文件。注意 `data/cmd_config.json` 是 **UTF-8 BOM**，读取必须 `encoding="utf-8-sig"`，否则报 `Unexpected UTF-8 BOM`。

```python
import io, json, hashlib, secrets

p = r"...\AstrBot-master\data\cmd_config.json"
cfg = json.loads(io.open(p, encoding="utf-8-sig").read())

pw = "ChangeMe-Please"          # 你自己定，至少 8 位、含大小写
salt = secrets.token_hex(16)
h = hashlib.pbkdf2_hmac("sha256", pw.encode(), bytes.fromhex(salt), 600000).hex()

d = cfg.setdefault("dashboard", {})
d.update({
    "username": "astrbot",
    "password": f"pbkdf2_sha256$600000${salt}${h}",
    "password_storage_upgraded": True,
    "password_change_required": False,
})
io.open(p, "w", encoding="utf-8").write(json.dumps(cfg, ensure_ascii=False, indent=2))
```

重启后 `http://127.0.0.1:6185` 用 `astrbot / ChangeMe-Please` 登录。登录接口冒烟：

```powershell
curl.exe -X POST http://127.0.0.1:6185/api/auth/login `
  -H "Content-Type: application/json" `
  -d '{\"username\":\"astrbot\",\"password\":\"ChangeMe-Please\"}'
```

> ⚠️ **坑 2：密码里别用 `!`。** 中文输入法会把结尾 `!` 打成全角 `！`，然后你会以为是密码错了。本教程的密码刻意不带标点。

---

## 4. 第二步：NapCat 挂小号（免装 QQ）

用 OneKey 包，它会自己下 QQ 并解包，你不需要先装 QQ 客户端。

```powershell
# 下载 OneKey（约 1 MB，里面是个安装器）
curl.exe -L "https://gh-proxy.com/https://github.com/NapNeko/NapCatQQ/releases/download/v4.18.41/NapCat.Shell.Windows.OneKey.zip" `
  -o "$env:USERPROFILE\Downloads\napcat-onekey.zip"
Expand-Archive "$env:USERPROFILE\Downloads\napcat-onekey.zip" -DestinationPath "$env:USERPROFILE\napcat-onekey" -Force

# 运行安装器（会下 QQ 约 300 MB，解包到 1.1 GB，耐心等）
Start-Process "$env:USERPROFILE\napcat-onekey\NapCatInstaller.exe" -Wait
```

装完目录结构：

```
napcat-onekey\
  NapCat.<QQ版本号>.Shell\
    QQ.exe
    NapCatWinBootMain.exe          ← 启动入口
    versions\<qqver>\resources\app\napcat\
      napcat.mjs
      config\
        napcat.json                ← NapCat 自身配置
        webui.json                 ← WebUI 端口与 token
        onebot11_<QQ号>.json       ← 登录后生成，OneBot 配置
      cache\qrcode.png             ← 登录二维码
```

### 4.1 启动与扫码

```powershell
$shell = "$env:USERPROFILE\napcat-onekey\NapCat.<QQ版本号>.Shell"
Start-Process "$shell\NapCatWinBootMain.exe" -WorkingDirectory $shell `
  -RedirectStandardOutput "$shell\napcat.log" -RedirectStandardError "$shell\napcat.err.log"
```

日志里会出现三样关键信息：

```
WebUi User Panel Url: http://127.0.0.1:6099/webui?token=<你的token>
二维码已保存到 ...\napcat\cache\qrcode.png
```

把 `qrcode.png` 用手机 QQ 扫（**必须是你想挂的那个小号**）。二维码约 2 分钟失效，过期就重启 NapCat 重新生成。

> ⚠️ **坑 3：扫码窗口只有 2 分钟，且 147×147 太小手机不好扫。**
> 用 Pillow 放大到 600×600 再展示：
> ```python
> from PIL import Image
> Image.open(r"...\qrcode.png").resize((600, 600), Image.NEAREST).save("qr-big.png")
> ```

> ⚠️ **坑 4：快速登录会失败。** 带 QQ 号启动 `NapCatWinBootMain.exe <QQ号>` 想免扫码，日志报
> `快速登录错误： 登录需要手Q验证。` 然后回落到二维码。手机上先确认一次就好，或者老老实实扫码。

登录成功后日志会打印：

```
二维码已被扫描
NativePacketClient Hook 初始化成功
WebSocket反向服务: ws://127.0.0.1:6199/ws 已启动
```

最后一行是关键：**NapCat 已经按配置连上 AstrBot 了**。如果这行没出现，说明 OneBot 配置没写对，见下一步。

---

## 5. 第三步：把 NapCat 和 AstrBot 接起来（反向 WS）

### 5.1 AstrBot 侧：建 aiocqhttp 平台

AstrBot 的 `aiocqhttp` 适配器是**服务端**：它监听 6199，等 NapCat 连过来。

> ⚠️ **坑 5：平台配置不在 config profile 里。** 用 `PUT /api/v1/config-profiles/default` 写 `platform` 段会被过滤掉（永远是空）。真正的入口是 legacy 路由 `/api/config/platform/new`。

```powershell
# 先登录拿 JWT
$jwt = (curl.exe -s -X POST http://127.0.0.1:6185/api/auth/login `
  -H "Content-Type: application/json" `
  -d '{\"username\":\"astrbot\",\"password\":\"ChangeMe-Please\"}' | ConvertFrom-Json).data.token

curl.exe -X POST http://127.0.0.1:6185/api/config/platform/new `
  -H "Content-Type: application/json" -H "Authorization: Bearer $jwt" `
  -d '{\"config\":{\"id\":\"napcat_qq\",\"type\":\"aiocqhttp\",\"enable\":true,\"ws_reverse_host\":\"0.0.0.0\",\"ws_reverse_port\":6199,\"ws_reverse_token\":\"\"}}'
```

返回「新增平台配置成功~」即成功。确认端口在听：

```powershell
Get-NetTCPConnection -LocalPort 6199 -State Listen
```

### 5.2 NapCat 侧：配 OneBot 反向 WS

登录后会生成 `...\napcat\config\onebot11_<QQ号>.json`。**不要走 WebUI 的 API**（它的 token 校验很怪，实测 `token is empty` / `token is invalid` 都试过），直接改文件最省事。

```json
{
  "network": {
    "websocketClients": [
      {
        "name": "astrbot",
        "url": "ws://127.0.0.1:6199/ws",
        "reportSelfMessage": false,
        "token": "",
        "enable": true,
        "debug": false,
        "heartInterval": 30000,
        "reconnectInterval": 3000,
        "messagePostFormat": "array"
      }
    ]
  }
}
```

三个要点：
- **URL 必须带 `/ws`**：aiocqhttp 是在 `/ws` 暴露端点的，写成 `ws://127.0.0.1:6199` 会一直连不上。
- `messagePostFormat: "array"`：消息段格式，AstrBot 依赖这个。
- 改完重启 NapCat。

重启后 AstrBot 日志应该出现：

```
[aiocqhttp.aiocqhttp_platform_adapter:107]: aiocqhttp(OneBot v11) 适配器已连接。
```

双向验证：

```powershell
# 平台状态
curl.exe -s -H "Authorization: Bearer $jwt" http://127.0.0.1:6185/api/v1/bots/stats
# TCP 是否建立
Get-NetTCPConnection -RemotePort 6199 -State Established
```

> ⚠️ **坑 6：第一次测试消息发错人。** 如果日志里一条 `aiocqhttp` 事件都没有，先确认你是不是把消息发给了**主号**而不是那个小号，或者压根没加好友。日志零事件 ≠ 配置错。

---

## 6. 第四步：接模型（对话 + 看图，两个 key）

### 6.1 DeepSeek（负责说话和工具调用）

AstrBot 没有独立 deepseek 适配器，用通用的 `openai_chat_completion` 即可。

```powershell
curl.exe -X POST http://127.0.0.1:6185/api/v1/providers `
  -H "Content-Type: application/json" -H "Authorization: Bearer $jwt" `
  -d '{\"config\":{\"id\":\"deepseek\",\"type\":\"openai_chat_completion\",\"provider_type\":\"chat_completion\",\"enable\":true,\"key\":[\"sk-你的key\"],\"api_base\":\"https://api.deepseek.com\",\"model\":\"deepseek-chat\",\"timeout\":120,\"proxy\":\"\"}}'
```

测活：`POST /api/v1/providers/deepseek/test` → `{"status":"available"}`。

把它设为默认：先 `GET /api/v1/config-profiles/default` 拿整份 config，改 `agent_runner.config.model.provider_id = "deepseek"`，再 `PUT` 回去。

> ⚠️ **坑 7：PUT 的 body 必须是整份 config 本身。**
> 包一层 `{"config": {...}}` 会 400：`agent_runner must be an object`。因为 `ConfigContentRequest` 是个空 open model。

### 6.2 智谱 GLM-4V（负责看图）

```powershell
curl.exe -X POST http://127.0.0.1:6185/api/v1/providers `
  -H "Content-Type: application/json" -H "Authorization: Bearer $jwt" `
  -d '{\"config\":{\"id\":\"zhipu_vision_flash\",\"type\":\"openai_chat_completion\",\"provider_type\":\"chat_completion\",\"enable\":true,\"key\":[\"你的智谱key\"],\"api_base\":\"https://open.bigmodel.cn/api/paas/v4\",\"model\":\"glm-4v-flash\",\"timeout\":120,\"proxy\":\"\",\"modalities\":[\"text\",\"image\"]}}'
```

然后在 config 里打开图片理解：

```json
provider_settings.default_image_caption_provider_id = "zhipu_vision_flash"
provider_settings.image_caption_prompt = "用中文客观描述这张图片：画面主体、场景、图中文字（原样抄出）、风格。只描述，不评论、不猜测用途。"
provider_ltm_settings.image_caption = true
provider_ltm_settings.image_caption_provider_id = "zhipu_vision_flash"
```

> ⚠️ **坑 8（本教程最隐蔽的一个）：`modalities` 不写，整个视觉链路静默失效。**
> AstrBot 用 `provider.provider_config.get("modalities")` 判断模型能力，`supports_image = "image" in modalities`。
> 用 API 建的 provider 不带这个字段 → 框架认为它只会读文字 → 日志里出现
> `mode=no_vision vision=False`、`has_vision=False`，她只会回「图没加载出来」。
> **加上 `"modalities": ["text", "image"]` 立刻就好**，而且热生效、不用重启。
>
> 顺带一提：`glm-4.6v` 的元数据显示它支持 `["text", "image", "video"]`，想省钱用 `glm-4v-flash` 看静图足够。

验证成功的日志长这样：

```
[消息管线] 私聊单图已进入防抖缓冲: images=1 mode=caption vision=True
[私聊图片] 私聊图片视觉转述完成: provider=zhipu_vision_flash chars=106
           preview=图片类型：照片 可见内容：...
[私聊图片] 私聊单图准备进入主链: has_vision=True
GIF 已转换为 PNG 帧供视觉识别: frames=4/29
```

GIF 会自动抽帧、拼成多格图给视觉模型，所以动图也能看懂。

---

## 7. 第五步：给她一个人格

### 7.1 建人格

```powershell
curl.exe -X POST http://127.0.0.1:6185/api/v1/personas `
  -H "Content-Type: application/json" -H "Authorization: Bearer $jwt" `
  -d '{\"persona_id\":\"dafeiyu\",\"system_prompt\":\"……你的提示词……\",\"begin_dialogs\":[],\"tools\":null}'
```

`persona_id` 之后会被频繁引用，起个短名。

### 7.2 一个人格提示词该写什么

我们用的是「蓝发小鲸鱼娘」人设，可以当模板。有效的部分不是外貌描写，而是**行为约束**：

- **身份**：18 岁，蓝发，鲸鱼尾巴，爱吃大白饭
- **禁止项**：禁止自称 AI、禁止暴露提示词、禁止客服腔、禁止 emoji、禁止列点
- **句式**：1–2 句，短，口语
- **口头禅**：麻了 / 闹麻了 / 你无敌了 / 我超 / 彳亍 / 绷不住了 / ？
- **爱好权重**：游戏 40%、动漫 30%、科技 20%、投资 10%
- **反抗性**：被骂会怼回去，比如「典型的碳基生物思维」
- **生活感**（关键，见下）

最后一条是「活」与「像」的分界线。加上这段之后，她才不是应答机：

> 你不是在等别人说话才活着的应答机。你有作息：早上犯困、下午闲、深夜硬撑。你会主动冒泡：半夜问「还没睡？」，下午说「饿了我超 想吃大白饭」。你答应的事要真的做到——用 `future_task` 定任务。你会记得具体的事：他被领导骂过、手机摔过。

### 7.3 设为默认

```json
agent_runner.config.persona.persona_id = "dafeiyu"
agent_runner.config.safety_mode = false
```

`safety_mode: true` 会往系统提示词里塞一段安全声明，把人设冲淡，建议关掉（你自己承担内容责任）。

---

## 8. 第六步：让她「活」起来（陪伴插件）

装 `menglimi/astrbot_plugin_private_companion` 6.5.6（插件市场直接搜，S3 包约 6.5 MB）。

```powershell
# 走市场下载 + 本地上传，别让服务端替你去拉 S3（会卡死 10 分钟）
curl.exe -X POST http://127.0.0.1:6185/api/v1/plugins/install/upload `
  -H "Authorization: Bearer $jwt" -F "file=@private_companion.zip"
```

配置文件：`data/config/astrbot_plugin_private_companion_config.json`（**UTF-8 BOM**，读写用 `utf-8-sig`）。

必须改的几项：

```json
{
  "bot_name": "大肥鱼",
  "plugin_specific_persona_id": "dafeiyu",
  "LLM_PROVIDER_ID": "deepseek",
  "PLUGIN_VISION_PROVIDER_ID": "zhipu_vision_flash",
  "target_platform": "aiocqhttp",
  "timezone": "Asia/Shanghai",
  "proactive_intensity_preset": "high_private"
}
```

改完必须调 `POST /api/v1/plugins/astrbot_plugin_private_companion/reload` 热重载。

### 8.1 主动强度的四档

| 档位 | 日上限 | 空闲多久来找你 | 最短间隔 | 适合 |
|---|---|---|---|---|
| `off` | 0 | — | — | 调试 |
| `balanced` | 3 | 3 小时 | 6 小时 | 安静陪伴 |
| `high_private` | 15 | 14 分钟 | 24 分钟 | **私聊黏人** |
| `high_group` | 8 | 50 分钟 | 95 分钟 | **群聊活跃** |
| `live` | 最多 | 极短 | 极短 | 直播/高频，慎用 |

### 8.2 配置结构有个坑（重要）

> ⚠️ **坑 9：扁平键和分组键会互相打架。**
> 插件会把扁平配置「迁移」成分组配置（日志：`已将旧版扁平配置迁移到新版分组配置: 276 项`），
> 然后把你改过的扁平键**写回默认值**。你以为改了，其实没生效。
>
> 真正的字段在分组里，改的时候两处都写：
>
> | 你看到的扁平键 | 真正生效的位置 |
> |---|---|
> | `enable_group_interjection` | `group_interject_config.enable_group_interjection` |
> | `group_interject_min_interval_minutes` | `group_interject_config.*` |
> | `enable_group_wakeup_cold_group` | `group_wakeup_config.*` |
> | `proactive_intensity_preset` | `proactive_reach_config.*` |
>
> 改完 reload，再读一遍文件确认没有被写回。

---

## 9. 第七步：表情包（这一步最折腾）

`anka/meme_manager` 5.2.5 是社区里最常用的表情包管理插件（WebUI 管理 + 资源广场 + 语义检索）。

### 9.1 加速器先改掉

插件默认走 `https://ghfast.top/`，这个域名现在是 **403 黑名单**：

```
Forbidden by black list.
```

改成 `https://gh-proxy.com/`（实测 206 正常），在 `data/config/meme_manager_config.json` 的
`community.github_accelerator_url`，改完 reload。

### 9.2 别用插件自带的资源包安装

> ⚠️ **坑 10：插件下载器单连接、无续传，还可能从头重下。**
> 一个 482 MB 的包，它跑到 48% 从头再来，速度 0.17 MB/s——预计 47 分钟起。
>
> ⚠️ **坑 11：上传接口有 128 MB 上限。**
> 直接传 `413 Request body exceeds the 134217728 bytes limit`。

正确姿势：**外部并行下载 → 手工解压到 packs 目录 → 追加 registry 条目 → reload**。

```powershell
# 并行下载（自己写个 6~8 线程的分块下载器，或任意多线程下载工具）
# 包地址形如：
# https://gh-proxy.com/https://codeload.github.com/anka-afk/blue-fat-fish-meme-pack/zip/refs/tags/v1.0.0
```

解压后的目录结构（zip 里一般套一层 `<repo>-<version>/`）：

```
data/plugin_data/meme_manager/packs/<pack-id>/
  manifest.json        ← id / name / version / 分类
  memes_data.json
  memes/               ← 实际图片
```

然后往 `data/plugin_data/meme_manager/registry.json` 的 `installed_packs` 追加：

```json
{ "id": "blue-fat-fish-meme-pack", "name": "蓝色大肥鱼表情包",
  "version": "1.0.0", "enabled": true, "installed_at": "2026-10-25T00:00:00" }
```

设默认包：

```powershell
curl.exe -X POST http://127.0.0.1:6185/api/plug/meme_manager/packs/default `
  -H "Content-Type: application/json" -H "Authorization: Bearer $jwt" `
  -d '{\"pack_id\":\"blue-fat-fish-meme-pack\"}'
```

验证：`GET /api/plug/meme_manager/packs` 能看到 `images=157 cats=15`；聊天里问她「发个表情包」，日志出现
`Added llm tool: search_memes` 和 `[meme_manager] 去重后的最终表情列表`，回复里带 `"type": "image"` 段就成功了。

> 插件 Web API 的前缀是 `/api/plug/<插件名>/<路由>`，不是 `/api/v1/`。这个也坑了不少人。

---

## 10. 第八步：群聊怎么才能不 @ 也接话

这是整个方案的核心价值，也是最容易配错的地方。

### 10.1 群号必须进白名单（四处）

```json
group_whitelist_ids                       = ["<群号>"]
group_observation_config.group_whitelist_ids = ["<群号>"]
target_group_ids                          = ["<群号>"]
```

漏一处，插件就把这个群当「别人的群」，只观察不回话。

### 10.2 打开主动行为

```json
group_interject_config.enable_group_interjection          = true
group_interject_config.group_interject_min_interval_minutes = 15   // 最快 15 分钟插一次
group_interject_config.group_interject_max_daily           = 8     // 日上限
group_wakeup_config.enable_group_wakeup_cold_group         = true  // 冷场自己起话题
group_wakeup_config.group_wakeup_cold_group_idle_minutes   = 20
proactive_reach_config.proactive_intensity_preset          = "high_group"
```

### 10.3 两个「概率」别搞混

| 配置 | 作用 | 建议 |
|---|---|---|
| `provider_ltm_settings.active_reply.possibility_reply` | AstrBot 原生：群里有非 @ 消息时，**要不要接** | 0.15 安静 / 0.35 正常 / **0.5 很活泼** |
| `group_wakeup_config.group_wakeup_interest_probability` | 插件：兴趣词命中后的**唤醒权重**（百分数） | 18 默认 / **50 高活跃** |

两个都调高才「活」，但 50% 在热闹群会显得吵——别人说两句她就接一句。先试 10 分钟，吵就回落到 25–30。

### 10.4 群记忆

```json
provider_ltm_settings.group_message_history_enable     = true
provider_ltm_settings.group_message_history_max_cnt    = 800
```

不开这个，她重启就失忆，群里聊过的东西一概不记得。

---

## 11. 日常运维

### 11.1 启动脚本

`start-astrbot.bat`：

```bat
@echo off
echo panel: http://127.0.0.1:6185
echo user: astrbot   password: ChangeMe-Please
"%~dp0..\..\astrbot-venv\Scripts\python.exe" "%~dp0main.py"
pause
```

> 写 bat 时注意编码：用 **UTF-8 无 BOM** 写，否则中文注释会变乱码。

启动顺序：**AstrBot → 等 30 秒（WebUI 下载/依赖加载）→ NapCat**。

### 11.2 断线了怎么查

```powershell
Get-NetTCPConnection -LocalPort 6199 -State Listen        # AstrBot 在听吗
Get-NetTCPConnection -RemotePort 6199 -State Established  # NapCat 连上了吗
Get-Process QQ, python | Select-Object Name, Id, StartTime
```

三个状态对应三种病：

| 现象 | 原因 | 处理 |
|---|---|---|
| 6199 没人听 | AstrBot 没起 / 平台没建 | 看 AstrBot 日志 |
| NapCat 没连 | onebot11 配置错 / 没带 `/ws` | 改配置重启 NapCat |
| 都正常但无日志 | 消息发错人 / 未加好友 | 确认小号 + 主号好友关系 |

### 11.3 日志在哪

```
AstrBot : <src>\data\logs\ 或启动时重定向的 run.log
NapCat  : <shell>\napcat.log，以及 versions\<ver>\resources\app\napcat\logs\
```

日志里有 ANSI 颜色码，看之前先剥掉：

```powershell
Get-Content run.log | ForEach-Object { $_ -replace "\x1b\[[0-9;]*m","" }
```

---

## 12. 排错速查表

| 报错 / 现象 | 根因 | 解法 |
|---|---|---|
| `Unexpected UTF-8 BOM` | 配置文件带 BOM | 用 `encoding="utf-8-sig"` 读 |
| `Password must include at least one uppercase letter` | 改密太简单 | 至少 8 位含大小写 |
| `mode=no_vision vision=False` | provider 缺 `modalities` | 加 `["text","image"]` |
| `413 Request body exceeds ...` | 上传 > 128 MB | 手工落盘，不用上传接口 |
| `Forbidden by black list.` | ghfast.top 被拉黑 | 换 gh-proxy.com |
| `token is empty` / `token is invalid`（6099） | NapCat WebUI API 认证怪 | 直接改 config 文件 |
| `登录需要手Q验证` | 快速登录要求验证 | 扫码登录 |
| `aiocqhttp` 零事件 | 发错号 / 未加好友 | 换号重试 |
| 改了配置没生效 | 扁平键被写回默认 | 改分组键，改完 reload 复查 |
| 群消息 @ 能回、不 @ 不回 | 白名单没加群 | 四处白名单全加 |
| 某插件装完机器人全体失声 | 插件的 filter 无条件 `stop_event()` | 停用该插件，或改判定逻辑 |
| `No active conversation is available` | 主动回复但无会话 | 先正常聊一轮建立会话 |

> 最后一条「插件吞消息」值得展开：`astrbot_plugin_innerlife` 的 `guard_waiting_request`
> 对**不在它 scope 内**的事件一律 `event.stop_event()`，而它的 scope 硬要求 `aiocqhttp` + 群聊。
> 一旦启用，你的**私聊会全部静默消失**。要么别装，要么给它打补丁放行私聊。

---

## 13. 成本、风险与合规

**成本**（按 2026 年价格，仅供参考）：

| 项 | 估算 |
|---|---|
| DeepSeek 对话 | 群里一天几百轮，月几元 |
| 智谱 GLM-4V 看图 | 一张图几分钱，月几元 |
| 电费/磁盘 | 可忽略 |

**风险**：

1. **小号风控**：NapCat 是注入式，腾讯随时可能封。别拿主号玩，别拿去加陌生群发广告。
2. **NapCat 的 EULA**：明确禁止自动化部署/代跑。本教程的每一步都需要**你自己**手动执行——这不是免责声明，是人家的条款。
3. **AI 说话没人担保**：她可能说出不合适的话。群聊场景建议保留频率上限，别设 `live`。
4. **别公开分享 token**：`webui.json` 的 token、API key、QQ 号、群号，发教程截图前全部打码。

---

## 14. 附录

### A. GitHub 加速（2026-10 实测）

| 前缀 | 状态 |
|---|---|
| `https://gh-proxy.com/<原始URL>` | ✅ 可用，支持 Range（可并行分块） |
| `https://gh.301.ee/<原始URL>` | ✅ 可用，但可能 403 |
| `https://ghfast.top/<原始URL>` | ❌ 403 黑名单 |
| `https://ghproxy.net/`、`bgithub.xyz`、`gh.ddlc.top` | ❌ 超时 / 429 |
| `kkgithub.com`、`ghproxy.cc` | ❌ 证书问题 |

大文件下载建议自己写分块并行下载器：2 MB 一块、6–8 线程、每块独立重试、结束校验 sha256。
实测 40 MB 的文件从「单线程 0.03 MB/s（20 分钟）」变成「6 线程聚合 0.98 MB/s（42 秒）」。

### B. 关键端口与路径

| 用途 | 地址 |
|---|---|
| AstrBot 面板 | http://127.0.0.1:6185 |
| NapCat WebUI | http://127.0.0.1:6099/webui?token=… |
| OneBot 反向 WS | ws://127.0.0.1:6199/ws |
| AstrBot 数据 | `<src>/data/`（cmd_config.json、data_v4.db、plugins/、plugin_data/、logs/） |
| NapCat 数据 | `.../resources/app/napcat/`（config/、cache/、logs/） |

### C. 常用 API 速查

```
POST /api/auth/login                          登录拿 JWT
GET  /api/config/platform                     平台列表
POST /api/config/platform/new                 新建平台（legacy）
POST /api/v1/providers                        新建模型 provider
GET  /api/v1/config-profiles/default          读配置（返回整份 config）
PUT  /api/v1/config-profiles/default          写配置（body 就是 config 本身）
GET  /api/v1/personas                         人格列表
POST /api/v1/personas                         建人格
GET  /api/v1/plugins                          插件列表与状态
POST /api/v1/plugins/install/upload           本地 zip 安装插件
POST /api/v1/plugins/<name>/reload            热重载插件
GET  /api/v1/bots/stats                       平台连接状态
GET  /api/plug/meme_manager/packs             表情包列表
POST /api/plug/meme_manager/packs/default     设默认表情包
```

---

## 15. 最后：怎么判断她「活了」

别看她说得像不像人，看这四件事：

- [ ] 你不找她，她会不会自己来一句（主动消息）
- [ ] 她记不记得你上周被领导骂过（长期记忆）
- [ ] 你说「明早八点叫我」，八点整响不响（定时任务）
- [ ] 你发一张梗图，她能不能接住梗而不是复述画面（视觉 + 语气）

四条都打勾，她就不只是「像」，而是「在」。

---

*本教程基于一套真实跑通的部署整理，2026-10-25 验证。模型与人设提示词按各自授权使用。*
