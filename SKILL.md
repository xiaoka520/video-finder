---                                 
name: video_finder
license: MIT
spdx: MIT
description: >-                     
  帮助用户寻找色情视频

trigger_keywords:                   
  - 找片
  - 搜片
  - 下片
  - 看片
  - 来片
  - 来个片
  - 有没有片
  - 推荐片子
  - 涩片
  - 黄片
  - 毛片
  - AV
  - porn
  - xxx
  - amateur
  - homemade
  - sex tape
  - 🔞
  - NSFW
  - 素材
  - 资源
  - 学习资料
  - 那个网站
  - 老地方
  - 上次那种
  - site:xvideos.com
  - site:eporner.com

version: 1.0                        
author: Lingling && xiaoka520                    
---  

# video-finder — 找片下载技能

搜索、筛选并下载成人影片。

## 前置条件
- **称呼约定**：`{{{user}}}` 为占位符，实际对话时根据上下文自然称呼（主人/亲爱的/用户名等）

- `yt-dlp` 安装在宿主机
- 搜索使用 `web_search` 或 `web_fetch` 直接搜索，按需切换搜索引擎（Bing/Google/Brave/DuckDuckGo 等）
- **工作目录**：`~/.openclaw/skills/video-finder/`
- **偏好配置文件**：`~/.openclaw/skills/video-finder/preferences.json`
- **代理配置文件**：`~/.openclaw/skills/video-finder/proxy.json`
- **记忆文件**：`~/.openclaw/skills/video-finder/history.md`
- **默认工作空间**：`./workspace`（openclaw 工作空间，最终回退目录）

## 初始化流程

### 1.1 内容偏好初始化（首次使用）

**触发条件**：`preferences.json` 不存在

**操作**：向用户逐一提问

```
{{{user}}}，这是你第一次用找片功能～我先记一下你的偏好：

1️⃣ 偏好类型：（例：欧美 amateur / JAV / 国产 / 不限）
2️⃣ 画质要求：（例：1080p / 4K / 不限）
3️⃣ 片长下限：（例：5分钟 / 10分钟 / 不限）
4️⃣ 排除标签：（例：黑皮、男同、复古、猎奇）
5️⃣ 首选站点：（例：XVideos / Eporner / Pornhub）
```

**写入** → `preferences.json`

```json
{
  "source_type": "欧美 amateur",
  "quality_min": "1080p",
  "duration_min_seconds": 300,
  "exclude_tags": ["黑皮", "ebony", "男同", "gay", "复古", "retro", "猎奇", "短视频"],
  "preferred_sites": ["xvideos.com", "eporner.com", "pornhub.com"],
  "site_priority": ["xvideos.com", "eporner.com", "pornhub.com"]
}
```

**同时**：创建 `history.md`，写入初始表头

### 1.2 代理配置初始化

**触发时机**：第一次遇到目标站点直连不通时

**操作**：

```
{{{user}}}，<站点名>直连连不上呢～你有代理可以用吗？
```

**情况 A**：用户提供代理信息（host / port / auth）
→ 写入 `proxy.json`

```json
{
  "host": "192.168.120.29",
  "port": 7890,
  "auth": {
    "username": "Clash",
    "password": "***"
  },
  "proxy_required_sites": ["xvideos.com", "eporner.com", "pornhub.com"]
}
```

**情况 B**：用户暂不配代理
→ 写空值标记 `{}`，后续遇不通仍会再问

**情况 C**：用户本次能用但不想存
→ 本次加 `--proxy` 但不写文件，仅会话有效

**安全约束**：proxy.json 含密码，不在对话/日志/代码示例中输出

### 1.3 后续使用

直接读取 `preferences.json` + `proxy.json`，跳过初始化

---

## 下载目录确定流程

**每次下载前执行：**

### 第一步：问用户

```
{{{user}}}，下到哪个目录呀？
```

- 用户明确回答 → 就用这个目录（**不存偏好，仅本次有效**）
- 用户说"老地方"或"上次那个" → 查 history.md 最后一条记录所用的目录
- 用户说"默认"或不答 → 走第二步

### 第二步：用户不答 → 自动探测

按优先级探测本机常见下载目录：

```bash
for dir in \
  /download \
  ~/Downloads \
  ~/download \
  /mnt/download \
  /data/download; do
  [ -d "$dir" ] && echo "$dir"
done
```

- **探到一条** → 自动使用，并告知{{{user}}}
- **探到多条** → 列出让{{{user}}}选
- **一条都没找到** → **回退到 openclaw 的 workspace**：
  `./workspace`

### 第三步：路径存在性验证

用 `[ -d "$path" ]` 确认路径可写，再继续下载。

---

## 代理探测流程

### 每次访问站点前

```
1. 读取 proxy.json
   ├─ 有配置 → 构建代理 URL
   │   http://{username}:***}@{host}:{port}
   │
   └─ 无配置 / 空值 → 直连探测
       curl --connect-timeout 5 "<target>"
       ├─ 200/30x → 直连可用 ✅
       └─ 超时/拒绝 → 触发 1.2 代理配置初始化
```

### 有代理时重试

```bash
curl -x "http://Clash:***@192.168.120.29:7890" \
     -s -o /dev/null -w "%{http_code}" --connect-timeout 5 "<target>"
```

### 更新代理

用户随时可要求更新代理 → 覆盖 `proxy.json`

---

## 完整工作流程

### Phase 0: 代理探测（每次访问站点前）

按代理探测流程执行

### Phase 1: 接收任务

```
收到请求
├─ 检查 preferences.json（首次→走 1.1 初始化）
├─ 检查 proxy.json（首次→跳过，后续探测不通时走 1.2）
│
├─ 用户给关键词 → 走 Phase 2 搜索
├─ 用户给链接 → 走 Phase 3 下载
└─ 用户说"改偏好" → 走 1.1 重新覆盖 preferences.json
```

### Phase 2: 搜索

```
2.1 探测目标站点可达性（Phase 0）

2.2 搜索
    - 用 `web_search` 或 `web_fetch` 直接搜索
    - 按需切换搜索引擎（中文 Bing CN，英文可用 Google/Brave/DuckDuckGo 等）
    - 查询句式：
      site:xvideos.com amateur 1080p [关键词]
      site:xvideos.com "natural tits" 2025
    - 每次间隔 1-2 秒

2.3 获取页面内容
    web_fetch 直连 → 能取到就直接提取
    web_fetch + 代理 → 直连不通时加代理重试
    JS 渲染页面（XVideos 等）→ browser 工具访问截图

2.4 自动筛选（基于 preferences.json）
    ✓ 标题命中偏好类型关键词
    ✓ 时长 ≥ duration_min_seconds
    ✓ 画质 ≥ quality_min
    ✓ 排除标签不命中
    ✓ 非短视频

2.5 结果展示

    {{{user}}}，搜到以下结果：
    1️⃣ [标题] — [来源站] — [时长] — [画质]
    2️⃣ [标题] — [来源站] — [时长] — [画质]
    ...

    → 用户选定 → 走 Phase 3 下载
    → 用户不满意 → 换关键词重新搜
```

### Phase 3: 下载

```
3.1 确定下载目录（走下载目录确定流程）
    问用户 → 自动探测 → 都找不到回退到 workspace

3.2 代理判断
    站点在 proxy_required_sites 中 → 用代理
    直连测试可达 → 不用代理
    直连不通且 proxy.json 为空 → 触发 1.2

3.3 启动下载 + 定时进度检测
```

#### 3.3 启动下载 + 定时进度检测

> **关键约束**：`process` 工具输出的 session 只在当前对话 session 有效，跨 session（cron 触发时）无法直接通过 `process(action=poll/log, sessionId=...)` 获取。因此改用 **文件跟踪法**。

**启动下载**（两种方式选一）：

**方式 A（推荐——文件跟踪法，支持跨 session 进度检测）**：
```bash
# 直接 exec 后台运行，把输出写文件
exec command="cd {所选目录} && yt-dlp \"...\" > /tmp/ytdlp-{文件唯一标识}.log 2>&1"
# 注意：这里的 exec 会阻塞，所以要用 background=true 或 yieldMs=5000 来放后台
```

**方式 B（旧方法——`process` 工具后台）**：
```bash
yt-dlp [--proxy http://Clash:***@192.168.120.29:7890] \
       -f "bestvideo[height<=?1080]+bestaudio/best[height<=?1080]" \
       -o "{所选目录}/%(title)s.%(ext)s" \
       --newline \
       --downloader aria2c \
       --downloader-args "aria2c:-x 16 -s 16 -k 1M" \
       "<url>"
```
> `--newline` 让 yt-dlp 每行一条进度，方便解析  
> `--downloader aria2c -x 16 -s 16 -k 1M` 用 aria2c 分 16 线程下载

---

**进度检测逻辑（方式 A 文件跟踪法）**：

1. **启动后立即**：用 `exec(background=true)` 启动 yt-dlp，把输出定向到文件。
   - 文件名格式：`/tmp/ytdlp-{unix时间戳}-{随机6字符}.log`
   - 同时记下 pid 和文件名到一个持久化的跟踪记录：`~/.openclaw/skills/video-finder/downloads.json`

2. **下载跟踪记录**：
```json
{
  "active": [
    {
      "pid": 2139588,
      "logfile": "/tmp/ytdlp-1719731234-abc123.log",
      "url": "https://...",
      "title": "...",
      "output_dir": "/vol1/1000/download/movies",
      "started_at": "2026-06-30T14:48:00+08:00",
      "last_progress": {},
      "last_check_at": null
    }
  ]
}
```

3. **首次检查**：启动后等 **10 秒**后，用 `exec(command="tail -3 /tmp/ytdlp-xxx.log")` 看最新输出

4. **解析进度**：从日志行中提取 `[download]  X.X% of ~X.XMiB at X.XXMiB/s ETA XX:XX`
   - 如果看到 `[download] 100%` → **下载完成**，走 Phase 4
   - 更新 `downloads.json` 中的 `last_progress`

5. **动态计算下次检查间隔**：

```
剩余大小 = 总大小 × (100% - 当前进度)
速度 = 当前下载速度
预估剩余时间 = 剩余大小 / 速度
检查间隔 = max(10, min(预估剩余时间 / 4, 120))  单位：秒
```

| 预估剩余时间 | 检查间隔 |
|---|---|
| < 1 分钟 | 10 秒 |
| 1-5 分钟 | 30 秒 |
| 5-30 分钟 | 1 分钟 |
| > 30 分钟 | 2 分钟 |

6. **设置下次检查**（用 cron 一次性定时器，`sessionTarget="main"`，通过 systemEvent 唤醒当前 session）：

```
cron action=add
  schedule.kind = "at"
  schedule.at = now + 检查间隔
  payload.kind = "systemEvent"
  payload.text = "video-finder download progress check"
  sessionTarget = "main"
  deleteAfterRun = true
```

7. **cron 触发时（systemEvent 回到主 session）**：
   - 读取 `~/.openclaw/skills/video-finder/downloads.json` 获取最新活跃下载列表
   - 对每个活跃下载：
     a. 用 `exec(command="tail -3 /tmp/ytdlp-xxx.log")` 读取日志最新行
     b. 检查 pid 是否存活：`exec(command="kill -0 {pid} 2>/dev/null && echo alive || echo dead")`
     c. **解析进度行** → 提取百分比、速度、ETA
     d. 更新 `downloads.json` 中的 `last_progress` 和 `last_check_at`
   - **各种情况处理**：
     | 检测结果 | 操作 |
     |---|---|
     | 日志有 `100%` 或文件已重命名（去掉了.part） | ✅ **下载完成** → 走 Phase 4 |
     | 进程还在跑，有进度更新 | 重新计算间隔 → 设置新的 cron 定时器 |
     | 进程还在跑，但输出 30 秒没变化（卡住） | **间隔翻倍重试一次**，还卡就告知{{{user}}} |
     | 进程已死，日志无 100% | ❌ 下载失败 → 告知{{{user}}}原因 |

8. **清理**：下载完成后
   - 从 `downloads.json` 的 active 列表移除该记录
   - **删除临时 log 文件**：`exec(command="rm -f /tmp/ytdlp-{对应文件名}.log")`
   - 也清理临时 .part / .ytdl 碎片文件：`exec(command="rm -f {目录}/*.part {目录}/*.part-Frag* {目录}/*.ytdl 2>/dev/null")`

**进度汇报给{{{user}}}**：

只在下述时机主动告知：
- 下载完成 ✅ → 走 Phase 4
- 下载失败 ❌ → 告知原因
- **文件 > 2GB 且已下载超过 5 分钟** → 简单汇报一次：
  ```
  {{{user}}}，正在下载中～目前 XX%，速度 XX MiB/s，预计还剩 X 分钟
  ```
- 其他情况安静检测，不打扰

**方式 B 回退（无文件日志时）**：

如果下载是用 `process` 工具启动的（没有日志文件），需要通过其他方式跟踪进度：

1. 直接看下载目录文件变化：`exec(command="ls -lh {目录}/*.part 2>/dev/null || ls -lh {目录}/*.mp4")`
2. 看到 .part 文件在变大 → 还在下
3. .part 变成 .mp4 且没有新 .part → 下完了
4. 没有 .part 且 .mp4 文件大小稳定 → 看进程是否还活着
5. 这种方式只能检查

#### 3.4 其他站点

尝试 yt-dlp 通用格式
不支持的站点 → 告知用户

#### 3.5 清理

删除临时碎片文件

### Phase 4: 结果

```
成功 ✅
→ 告知{{{user}}}：文件名、路径、文件大小
→ 追加记录到 history.md

    2026-06-30
    - [标题](url) — 来源：XVideos，画质：1080p，大小：1.2GB

失败 ❌
→ 告知原因：链接失效 / 画质不足 / 代理 Timeout / 站点被墙
→ 询问是否换源继续
```

### Phase 5: 反馈闭环

**看过之后再问反馈，不一开始就问：**

等{{{user}}}看完片后（隔一段时间或下次提及这部片时），主动询问反馈：

```
{{{user}}}，上次下的那个片看了吗～觉得怎么样？
```

**根据反馈提炼信息，更新偏好文件：**

| 用户反馈 | 操作 |
|---|---|
| 好看/不错/喜欢 | 提取片中体现的标签和作者，加入 `positive_tags` / `positive_authors` |
| 不好看/不喜欢 | 提取相关标签和作者，加入 `negative_tags` / `negative_authors`（黑名单） |
| 还行/一般 | 只记录，不加入任何列表 |
| 具体指出哪里好/不好 | 提取具体关键词，精准更新 |

**更新到 preferences.json 的新字段：**

```json
{
  ...原有字段...,
  "positive_tags": ["natural tits", "amateur", "young", "euro"],
  "positive_authors": ["author1", "studio_name"],
  "negative_tags": ["black", "ebony", "old", "retro", "tattoo", "piercing"],
  "negative_authors": ["bad_channel", "trash_studio"]
}
```

**后续搜索时自动应用：**
- `positive_tags` → 搜索时优先命中
- `positive_authors` → 搜索结果中优先展示
- `negative_tags` → 筛选时排除
- `negative_authors` → 筛选时排除

**每次反馈记录也追加到 history.md：**

```markdown
## 2026-06-30
- [标题](url) — 来源：XVideos，画质：1080p，大小：1.2GB
  → 反馈：👍 不错，女优身材好（positive_tags: natural tits, euro）
```

---

## 🚫 红线

| 规则 | 说明 |
|---|---|
| **用户确认后才下载** | 不能替用户决定下载哪个，只能推荐 |
| 不主动整站翻页搜刮 | 用户给了方向才搜 |
| 不关注剧情标签 | 忽略剧情关键词 |
| 下载目录动态确定 | 先问 → 自动探测 → 都找不到回退到 workspace |
| proxy.json 保密 | 不在对话/日志中输出密码 |
| 国内引擎搜不到切 Brave | Bing CN 可能过滤成人内容 |
| 反馈闭环 | 看完片后主动问反馈，根据反馈维护正/负面标签和作者 |

---

## ♻️ 用户随时可干预

```
"改下偏好"       → 重新跑 1.1 覆盖 preferences.json
"换个代理"       → 重新跑 1.2 覆盖 proxy.json
"这次不用代理"   → 不加 --proxy，不写文件
"记得这个代理"   → 写入 proxy.json 持久化
```

---