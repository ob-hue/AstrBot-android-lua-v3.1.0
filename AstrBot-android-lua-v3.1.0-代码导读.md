# AstrBot-android-lua v3.1.0 · 代码导读

> 一句话：在 **Lua 沙盒 App 的原生界面里**，给 Ollama + AstrBot + NapCat 做「一键启动 + 后台看护 + 磁盘清理 + WebUI 直通」，
> 并且**向量模型名可以在界面上直接改**，不再写死。
> 全部逻辑在 App 内部闭环，不依赖任何外部程序。

---

## 0. 版本沿革

| 版本 | 变化 |
|---|---|
| `0.2.1` | 上游原版：主页两张卡（AstrBot / NapCat），手动启停，纯 Lua |
| `v2.2.0` | 新增：全栈状态卡、一键启停、后台看护（60s）、`boot_services.sh`、`clean_daemon.sh` |
| `v2.2.2` | 与 v2.2.0 内容一致，仅重新打包 |
| `v2.3.0` | Ollama 全栈整合；UI 单卡化（删两张卡）；WebUI 双快捷入口；QQ 号显示修复；临时下载包清理 |
| **`v3.1.0`** | **向量模型动态配置**：状态卡点一下即可改模型名 → 持久化 → 通过 `OLLAMA_MODEL` 传给脚本；界面显示版本号 |

**改动范围**：只动了 `main.lua`、`boot_services.sh`、`clean_daemon.sh` 三个文件中的前两个 + 文档。
`clean_daemon.sh` 本版未改（仍是 v2.3.0 的六类清理），`agent/`、`sandbox/`、`agent.lua`、`AGENTS.md` 共 7 个文件**一字未改**。

| 文件 | v2.2.2 | v2.3.0 | v3.1.0 |
|---|---|---|---|
| `main.lua` | 1552 行 | 1905 行 | **2002 行** |
| `boot_services.sh` | 250 行 | 350 行 | **357 行** |
| `clean_daemon.sh` | 185 行 | 277 行 | 277 行（未改） |

---

## 1. 目录结构与部署位置

```
scripts/                     ← 这就是包解压后的落地位置
├── main.lua         2002 行   ★ 主脚本：界面 + 业务逻辑 + 模型配置
├── boot_services.sh  357 行   ★ 启动全栈（Ollama → NapCat → AstrBot）+ webui.json 防重置
├── clean_daemon.sh   277 行   ★ 磁盘守卫（默认只报告，含临时下载包清理）
├── 代码导读.md              ← 本文档（不参与运行，可放可不放）
├── agent/
│   ├── main.lua       718 B   受保护入口，注册主页顶栏两个 Agent 按钮
│   └── sandbox       2998 B   命令行工具：reload / ping / run / log
├── sandbox/
│   ├── audio_player.lua        内置库（随引擎维护，别动）
│   └── audio_recorder.lua
├── agent.lua         7305 B    Agent 辅助模块（被 agent/main.lua require）
└── AGENTS.md        61839 B    上游完整 API 文档 ← 想加功能先查它
```

**部署**：把文件原样放进 App 的 `scripts/` 目录，`main.lua` 覆盖旧的。
`boot_services.sh` / `clean_daemon.sh` 要 `chmod +x`。

> ⚠️ `agent/main.lua` 是**受保护加载**的：即使 `main.lua` 写崩了，
> 顶栏那两个 Agent 按钮仍然在，能救回来。

---

## 2. 运行时架构

```
┌──────────────────────── App 进程（LuaJIT 2.1）────────────────────────┐
│                                                                      │
│  main.lua                                                            │
│    ├─ APP_VERSION = "v3.1.0"      ← 状态卡标题直接显示               │
│    ├─ app.page("home", fn)        声明式 UI（每次渲染重建）           │
│    ├─ nav.tabs{...}               三个页：主页 / WebUI / 终端         │
│    └─ app.agent_actions{...}      顶栏按钮（在 agent/main.lua 里注册）│
│                                                                      │
│  状态来源：ctx.running[key]        ← 宿主维护，无需轮询               │
│  实况补充：host.exec(/dev/tcp)     ← 探 11434 + dashboard             │
│  模型配置：host.get/set("ollama_embed_model") ← 界面点改、持久保存    │
│  后台定时：host.interval(ms, cb)   ← 退到后台仍运行（App 有保活服务） │
└────────────────────────────┬─────────────────────────────────────────┘
                             │ host.spawn（带 OLLAMA_MODEL=...）
                             ▼
┌──────────────── 容器（proot Debian，root 环境）──────────────────────┐
│  boot_services.sh   → Ollama(:11434, 模型 $OLLAMA_MODEL) → NapCat → AstrBot
│  clean_daemon.sh    → 日志截断 / 缓存清理 / 超期下载包清理           │
│  /root/ollama.log  /root/astrbot.log  /root/napcat.log               │
└─────────────────────────────────────────────────────────────────────┘
```

**关键设计**：Lua 只负责「按一下」和「读状态」，真正的启停与清理全在 shell 里。
模型名是**唯一需要跨进程传递的参数**，用环境变量 `OLLAMA_MODEL` 传，不做任何文件协议。

---

## 3. 三个核心文件

### 3.1 `main.lua`

#### 模块地图

| 模块 | 行 | 作用 |
|---|---|---|
| `APP_VERSION` | 16 | `"v3.1.0"`，状态卡标题直接用 |
| `ports` | 22–31 | 端口持久化（`host.get/set`） |
| **Ollama 核心** | **33–225** | 端口/模型/安装探测、serve 拉起、停止、传参、模型编辑器 |
| `GH_PROXIES` / `gh_*` | 234–342 | GitHub 镜像测速与选择 |
| `ENV_STEPS` / `env_*` | 345–698 | 环境安装（base / uv / napcat / astrbot） |
| `astrbot_start_command()` | 700 | 拼出 AstrBot 启动命令，含"环境未装完整就退出"的守卫 |
| `astrbot_toggle(running)` | 711 | AstrBot 启停（`host.spawn` / `host.stop`） |
| `NC` 模块 | 726–1518 | NapCat 实例管理：创建、端口分配、OneBot 配置、launcher、启停、登出 |
| **QQ 号还原** | **769–887** | `detect_qq` / `is_qq` / `hydrate_qq` / `qq_refresh` |
| `stack_state(ctx)` | 1243 | 汇总状态（含 Ollama、真实 UIN、模型名） |
| `boot_all(ctx)` | 1282 | 启动全栈（Ollama 排最前，带 `OLLAMA_MODEL`） |
| `WATCH` + `stack_watch_*` | 1339–1456 | 60s 巡检、退避拉起、通知栏 |
| `stop_all(ctx)` | 1459 | 全部停止（含 Ollama） |
| `napcat_tile` | 1654 | 单账号 UI 行 |
| `accounts_expansion` | 1723 | 账号 + AstrBot 端口（折叠区） |
| `stack_card` | 1827–1894 | **主页唯一主卡（标题带版本号）** |
| `do_backup` / `do_restore` | 1896 / 1916 | 备份与恢复 |
| `app.page("home", ...)` | 1989 | 主页组装（3 个区块） |

#### ① Ollama 核心与模型配置（33–225）

| 函数 | 行 | 作用 |
|---|---|---|
| `valid_model_name(s)` | 46 | 只允许 `字母/数字/- _ . : / +`，长度 1–120 |
| `ollama_port()` | 50 | 读 `host.get("ollama_port")`，默认 11434 |
| `ollama_model()` | 57 | 读 `host.get("ollama_embed_model")`；**清洗+校验，脏值回退默认** |
| `ollama_installed()` | 68 | 探 6 个路径（`/root/.ollama/ollama`、ubuntu 映射、PATH），5s 缓存 |
| `OLLAMA_SERVE_SH` | 86 | 拉起脚本：先探 11434 → 找 BIN → `exec serve` |
| `ollama_start(quiet)` | 106 | 直接 spawn serve（无脚本时的兜底） |
| `ollama_stop()` | 117 | `host.stop("ollama")` + `pkill -f "ollama serve"` |
| `ollama_toggle(on)` | 123 | 停止 / 启动（启动走 `ollama_boot_service`） |
| `ollama_env(model)` | 128 | 拼 `OLLAMA_MODEL="xxx"`，**唯一的传参出口** |
| `ollama_boot_service(quiet)` | 134 | 有 `boot_services.sh` 就 `OLLAMA_MODEL=... /bin/sh ... ollama`，否则回落 serve |
| `ollama_check_model(name)` | 145 | 后台 `ollama list` 核实模型在不在，缺则通知提醒 `ollama pull` |
| `LIVE` / `live_probe` | 169–196 | 4s 节流的端口实况（11434 + dashboard），变化才重建 |
| `edit_ollama_model()` | 200 | 点击 → `host.input` → 校验 → `host.set` → 提示 + 核实 |

**模型名的三条安全线**（这是 v3.1.0 最关键的设计）：

1. **写入前校验**：`valid_model_name` 只放行 `[A-Za-z0-9._:/+-]`，所以不可能把引号、空格、`$()` 拼进 shell；
2. **读取时再校验**：`ollama_model()` 会重新清洗一遍，即使有人手改了存储里的值，脏值也只会回退默认，
   绝不会被拼进 `host.spawn` 的命令串；
3. **默认值单一来源**：`OLLAMA_DEFAULT_MODEL`（Lua）与 `${OLLAMA_MODEL:-nomic-embed-text}`（sh）同名同义。

#### ② QQ 号还原（769–887）

**旧逻辑的病根**：`ins.qq` 只在创建账号时写成 `""`，之后**从来没有任何代码回填它**。
`NC.detect_qq()` 早就存在，但只被 `ensure_ws` 用来拼文件名，结果没写回实例记录 →
扫码登录成功、`onebot11_<uin>.json` 也生成了，卡片照样写「未登录」。

**三路只读探测 + 回写**：

| 优先级 | 来源 | 实现 | 同步/异步 |
|---|---|---|---|
| ① | 实例记录 `ins.qq` | 已识别 / 手填 | 同步 |
| ② | `config/onebot11_<uin>.json` | 文件名即 UIN，NapCat 登录后才生成 | 同步 |
| ③ | `config/webui.json` 的 `autoLoginAccount` | 正则取数字 | 同步 |
| ④ | 实例 `logs/*.log`（单实例再兜 `/root/napcat.log`） | `grep -aoE` 抓 `uin/QQ/autoLoginAccount`，只 tail 300KB | 异步，8s 节流 |

- `NC.start()`（1193）在 `write_launcher` **之前**调 `hydrate_qq` → 下次写 `webui.json` 时
  `autoLoginAccount` 是真实 QQ，快速登录不再被抹掉；
- 看护每轮末尾 `qq_refresh(true)`，登录后最多 60s 内卡片跟上；
- 状态灯直接显示号码（如 `QQ 123456789`）。

#### ③ 单卡化后的主页（1827–1894）

`astrbot_card`（v2.2.x 里叫 `quick_start_card`）和 `napcat_card` 两个函数已整个删除，
主页只剩一张「全栈状态」卡，**标题带版本号**：

```
┌ 全栈状态 · v3.1.0 ────────────────────────────────────┐
│ ● AstrBot 运行中   ● NapCat 1/1   ● Ollama 已就绪      │  ← 第一排三个服务灯
│ ● QQ 123456789     ● 链路就绪                          │  ← 第二排 QQ / 链路
│ [AstrBot WebUI]  [NapCat WebUI]                       │
│ [一键启动全栈]    [全部停止]                            │
│ [清理空间]        [启动/停止 Ollama]                    │
│ 🖊 模型: nomic-embed-text（点击修改）  ★ v3.1.0 新增   │  ← 点开就是输入框
│ 自动看护  ▢                                            │
│ 每 60 秒巡检：Ollama(:11434) · AstrBot · NapCat …       │
│ ▸ 账号与端口（1）  ← 折叠区：账号列表 / AstrBot 服务 / 端口 │
└───────────────────────────────────────────────────────┘
```

- 灯的颜色：绿=通；红=该通不通；**灰=没装 Ollama**（不吓人）。
- 模型按钮：`模型: <当前模型>（点击修改）`，点击弹 `host.input`，保存后立即生效（下一次启停/预热用新模型）。
- `NapCat WebUI`：单实例直接开；多实例弹 `host.sheet` 让用户选。

#### ④ 后台看护（1339–1456）

```lua
local WATCH = {
  timer     = nil,
  ms        = 60000,   -- 巡检间隔
  fails     = {},      -- 每个 key 的连续失败次数
  last_sig  = nil,     -- 上次状态签名（变了才发通知）
  notify_id = 9901,    -- 常驻状态通知（避开 1001，那是 App 保活服务）
  alarm_id  = 9902,    -- 掉线告警通知
  max_tries = 3,       -- 连续失败几次后降频
  cooldown  = 10,      -- 之后每 N 个周期试一次
}
```

一轮 `stack_watch_tick`（1390）：

1. 一次 `host.exec` 探完 **11434 + dashboard + 所有 NapCat WebUI 端口**（`/dev/tcp`）
2. **Ollama 不通且已安装 → `ollama_boot_service(true)`**（内部带 `OLLAMA_MODEL`，预热的就是用户选的模型）
3. AstrBot 不通 → `astrbot_toggle(false)`
4. 每个 NapCat 端口不通 → `NC.start(id)`
5. 带退避：连续失败 3 次后降频为每 10 个周期一次
6. 状态签名变了才更新通知栏
7. 末尾 `qq_refresh(true)` 补捞真实 UIN

### 3.2 `boot_services.sh`（357 行）

**它解决两个具体故障**：

1. App 自带启动器每次启动都重写 `webui.json`，清空 `autoLoginAccount` 和 token → 每次都要重新扫码；
2. AstrBot 的知识库 / 向量检索依赖本地 embedding，而 Ollama 常常没人拉起、或模型名写死不合口味。

| 函数 | 行 | 作用 |
|---|---|---|
| `OLLAMA_PORT/MODEL/WAIT/AUTO_PULL` | 42–45 | `OLLAMA_MODEL="${OLLAMA_MODEL:-nomic-embed-text}"`，**外部传入优先** |
| `port_open()` | 57 | `/bin/sh` 是 dash 时没有 `/dev/tcp`，自动回退 `bash -c` |
| `snapshot_config()` | 129 | 只有当前配置**有账号**时才备份到 `/root/.webui.json.keep` |
| `restore_config()` | 120 | 只在当前配置**被清空**时用备份覆盖 |
| `watch_and_restore()` | 147 | 后台 30 秒内每 100ms 调一次 `restore_config`（抢改） |
| `find_ollama()` | 163 | 依次找 `$APP_HOME/.ollama/ollama`、`/root/.ollama/ollama`、`/usr/local/bin`、`/usr/bin`、`PATH` |
| `ollama_warm()` | 172 | `ollama list` 判断**用户指定模型**在不在；在就 `curl /api/embed` 预热一次 |
| `start_ollama()` | 205 | 已通→只预热；没装→`absent`；否则 `nohup ollama serve` + 轮询 11434 |
| `start_napcat()` | 231 | 路线 A：`start_napcat.sh`（不改写配置）<br>路线 B：App launcher + 抢改 |
| `start_astrbot()` | 264 | **必须用 `.venv/bin/python`** |
| `clean_start()` | 288 | 拉起 `clean_daemon.sh --loop`（pid 锁，幂等） |
| `status_json()` | 315 | 输出 `{"napcat":"up","astrbot":"down","ollama":"up",...}` |

**启动顺序**：

```
snapshot_config
  → clean_start
  → start_ollama      ★ 11434 就绪 + 预热 $OLLAMA_MODEL
  → start_napcat
  → start_astrbot     ★ 此时 embedding 一定可用
  → status_json
```

**模型名的来源优先级**（从高到低）：

| 优先级 | 来源 | 例子 |
|---|---|---|
| 1 | 命令行环境变量 | `OLLAMA_MODEL="bge-m3" sh boot_services.sh start` |
| 2 | App 状态卡里的设置 | `host.set("ollama_embed_model", "bge-m3")` → Lua 拼进 `host.spawn` |
| 3 | 脚本默认值 | `nomic-embed-text` |

没装 Ollama 不是错误：只把 `OLLAMA` 标成 `absent`。模型缺失也只在日志里提示一句，
不会卡住开机（要自动拉模型就设 `OLLAMA_AUTO_PULL=1`）。

用法：

```sh
sh boot_services.sh            # 启动全栈（Ollama → NapCat → AstrBot）
sh boot_services.sh status     # 状态 JSON（三服务）
sh boot_services.sh stop       # 全停（含 pkill ollama serve）
sh boot_services.sh ollama     # 只拉起 Ollama 并预热当前模型
OLLAMA_MODEL=bge-m3 sh boot_services.sh start   # 临时换模型启动
sh boot_services.sh snapshot   # 手动备份当前配置
sh boot_services.sh clean      # 后台起定时清理守护
```

### 3.3 `clean_daemon.sh`（277 行，本版未改）

**三重保险**——因为这个脚本要删东西，误删代价太大：

| 保险 | 做法 |
|---|---|
| ① 模式分级 | 默认 `report` 只看不动；`--apply` 只演练；`--apply --yes` 才真删 |
| ② 逐项过闸 | `guard()`（64 行）拒绝 `.db/.sqlite/webui.json/config/_home`；归档再多一道 `archive_guard()` |
| ③ 只清内容 | 只删缓存目录**里面的东西**，目录本身、`_home` 根一律不动 |

清哪六类（`one_pass()`，261 行）：

| # | 目标 | 函数 | 细节 |
|---|---|---|---|
| 1 | NapCat 媒体缓存 | `clean_napcat_media()` 104 | 只在 `nt_data`/`cache`/`temp` 路径下找 `Pic/Video/Ptt/Record`，保留最近 3 天 |
| 2 | 表情包小偷缓存 | `clean_stealer()` 126 | `plugin_data/*stealer*/` 下的 memes/images/pics/cache |
| 3 | 包管理器缓存 | `clean_pkg_cache()` 141 | apt / pip / uv / npm，20MB 以上才动 |
| 4 | 日志限流 | `cap_log()` 157 | 超 5MB 就 `truncate -s 5M` |
| 5 | 临时下载包 | `clean_tmp_archives()` 226 | `/root/` 与 `/storage/emulated/0/Download/` 下超 60 分钟未改的 `*.tar.zst` / `*.zip` |
| 6 | 体检 | `report()` 250 | 占 Top 8 + 下载目录大小 + 磁盘使用率 |

`archive_guard()` 六道闸：后缀 → 文件名黑名单（`backup`/`AstrBotBubble`/`AstrBot-android-lua`/`plugin`/`config`）
→ 路径黑名单（`AstrBot/`、`napcat_instances/`、`.venv/`、`.cache/`、`data/`）→ `guard()`
→ 普通文件且非软链 → `find -mmin +60` 成立。真删只走 `rm -f`、不递归。

**绝对不碰**：`napcat_instances/*_home`（登录态+聊天记录）、`AstrBot/data/*.db`、
`.venv`、`webui.json`、任何令牌文件、任何名字像备份/插件/组件包的压缩包。

---

## 4. 点一次「一键启动全栈」，发生了什么

```
用户点击
  └─ stack_card 的 onTap → boot_all(ctx)
       └─ host.exists("/root/boot_services.sh") == true
            └─ host.spawn('OLLAMA_MODEL="bge-m3" /bin/sh /root/boot_services.sh start',
                          "全栈启动", "stack_boot")     ★ v3.1.0 带模型名
                 ├─ key = "stack_boot" → ctx.running["stack_boot"] = true
                 ├─ 输出流式进「终端」页
                 └─ 3 秒后 live_probe(true) 让灯跟上

容器里：
  snapshot_config → clean_start → start_ollama
    ├─ 已通 11434？→ 只预热 $OLLAMA_MODEL
    ├─ 没装 ollama？→ OLLAMA=absent，继续
    └─ 否则 nohup ollama serve + 轮询 11434（最多 30s）→ ollama_warm（按 $OLLAMA_MODEL）
  → start_napcat（路线 A/B + 抢改 webui.json）
  → start_astrbot（.venv/bin/python）
  → status_json：{"napcat":"up","astrbot":"up","ollama":"up","at":"..."}

60 秒后：
  stack_watch_tick → 探 11434 + dashboard + 各 WebUI 端口
    → 谁不通拉起谁（Ollama 走 ollama_boot_service，仍带当前模型名）
    → 通知栏更新 → qq_refresh 补捞 UIN
```

---

## 5. 改代码常用 API 速查

```lua
-- 执行
host.spawn(cmd, 标签名, key)     -- 长命令，流式进终端 tab，ctx.running[key] 跟踪状态
host.stop(key)                   -- 停
host.exec(cmd, cb)               -- 短命令，cb({code, output})
host.run(程序, 参数表, cb)        -- 宿主层进程，cb({code, stdout, stderr})

-- 定时
host.interval(ms, cb) → id       -- 重复；退后台仍跑；reload 时自动清理
host.clear_interval(id)
host.delay(ms, cb)               -- 单次

-- 文件 / 设置
host.read_file(p) / host.write_file(p, s) / host.exists(p) / host.mkdirs(p) / host.list_dir(p)
host.get(k) / host.set(k, v)     -- set 触发页面重建

-- UI
card / tile / row / expanded / button / icon / text / toggle / menu / spacer / padding / expansion
app.page("名字", function(ctx) return {...} end)
nav.tabs({...})   host.nav.go(n)   host.toast(s)   host.input(opts, cb)   host.confirm(s, cb)
host.webview_open(url, 标题)      host.sheet({ title=, items= })

-- 通知
host.notify{ id=, title=, body=, ongoing= }    -- 同 id 覆盖
```

**引擎是 LuaJIT 2.1（= Lua 5.1）**：没有 `//` 整除、没有 `goto`，`os/io/ffi` 全可用。

---

## 6. 配置项

| 在哪 | 变量 | 默认 | 说明 |
|---|---|---|---|
| `main.lua` `WATCH` | `ms` | 60000 | 巡检间隔 |
| | `max_tries` | 3 | 连续失败几次后降频 |
| | `cooldown` | 10 | 降频后每 N 周期试一次 |
| | `notify_id` / `alarm_id` | 9901 / 9902 | 通知 id（别用 1001） |
| `main.lua` 持久键 | `ollama_port` | 11434 | Ollama 端口（Lua 侧只读，改了要同步脚本） |
| | `ollama_embed_model` | `nomic-embed-text` | **向量模型名（状态卡点改，v3.1.0）** |
| `boot_services.sh` | `NAPCAT_PORT` / `ASTRBOT_PORT` | 6100 / 6185 | 探测端口 |
| | `OLLAMA_PORT` | 11434 | Ollama 端口 |
| | `OLLAMA_MODEL` | `nomic-embed-text` | **向量模型（外部传入优先）** |
| | `OLLAMA_WAIT` | 30 | 等 11434 就绪的秒数 |
| | `OLLAMA_AUTO_PULL` | 0 | 1=模型缺失时自动 `ollama pull`（需联网） |
| | `WEBUI_KEEP` | `/root/.webui.json.keep` | 配置备份路径 |
| `clean_daemon.sh` | `KEEP_DAYS` | 3 | 缓存保留天数，0=全清 |
| | `LOG_CAP_MB` | 5 | 日志上限 |
| | `CLEAN_INTERVAL` | 7200 | 守护模式间隔 |
| | `ARCHIVE_KEEP_MIN` | 60 | 下载包超期分钟数 |
| | `ARCHIVE_MAX_DEPTH` | 1 | 扫描深度（1=只扫根层） |
| | `STORAGE_DOWNLOAD` | `/storage/emulated/0/Download` | 下载目录 |

### 6.1 如何自定义向量模型（v3.1.0）

**为什么需要**：不同人的知识库/文档语言不同，`nomic-embed-text` 未必最合适。
中文场景常用 `bge-m3`、`quentinz/bge-large-zh-v1.5`，长上下文可用 `qwen2.5:7b` 这类通用模型。
v3.1.0 之前模型名写死在 Lua 和 shell 两处，换模型要改代码；现在**只改一处设置**。

**三步操作**：

| 步 | 做什么 | 界面/命令 |
|---|---|---|
| 1 | 下载模型 | `ollama pull bge-m3`（在终端页执行；`OLLAMA_AUTO_PULL=1` 时启动会自动拉） |
| 2 | 在状态卡改模型 | 主页 →「模型: nomic-embed-text（点击修改）」→ 输入 `bge-m3` → 确定 |
| 3 | 重启/预热 | 点「启动 Ollama」（或「一键启动全栈」），日志会显示 `模型 bge-m3` |

**改完之后发生了什么**：

1. `host.set("ollama_embed_model", "bge-m3")` 持久化，重启 App 仍在；
2. 页面立即重建，模型按钮文字变成 `模型: bge-m3（点击修改）`；
3. `host.toast` 提示「向量模型已切换至 bge-m3，请确保已用 ollama pull 下载该模型」，
   同时发一条通知（id 9903，同 id 覆盖不堆叠）；
4. 后台执行 `ollama list` 核实：已下载 → 通知「已就绪」，没下载 → 通知「请先 ollama pull」；
5. 下次 `host.spawn` 时自动带上 `OLLAMA_MODEL="bge-m3"`，`boot_services.sh` 的
   `ollama_warm()` 预热的就是 `bge-m3`。

**模型名的合法写法**（校验只放行这些字符）：

| 写法 | 例子 | 说明 |
|---|---|---|
| 纯名字 | `bge-m3` | 最常用 |
| 带 tag | `qwen2.5:7b` | 冒号后是 tag，等价 `:latest` 可省略 |
| 带命名空间 | `quentinz/bge-large-zh-v1.5` | 斜杠分隔 namespace |
| 带点/下划线/加号 | `nomic-embed-text`、`e5_large`、`bge++` | 都允许 |

不允许空格、引号、`$`、反引号等 shell 元字符——校验会直接拒绝并 toast 提示，
这是为了防止模型名被拼进 `host.spawn` 的命令串。

**改坏了怎么回退**：把模型名改回 `nomic-embed-text` 即可；或在终端执行
`OLLAMA_MODEL=nomic-embed-text sh /root/boot_services.sh start`。
存储里的脏值即使被手改坏，`ollama_model()` 也会自动回退默认值。

**注意**：模型名只影响**预热与 AstrBot 的 embedding 调用约定**；
AstrBot 侧用哪个模型，取决于它在 `data/cmd_config.json` 里的知识库/embedding 配置
（若插件读取环境变量，请把 `OLLAMA_EMBED_MODEL` 一类的键也对齐，本包不代改 AstrBot 配置）。

---

## 7. 日志在哪

| 日志 | 位置 | 看什么 |
|---|---|---|
| **Lua 运行日志** | 设置页顶栏「Lua 日志」 | 脚本加载错误、`host.log`、`print` |
| 脚本输出 | App「终端」页 | `boot_services.sh` / `clean_daemon.sh` 的实时输出（含"向量模型: xxx"） |
| **Ollama** | `/root/ollama.log` | `serve` 启动失败、端口占用、模型加载 |
| AstrBot | `/root/astrbot.log` | 启动失败、缺依赖、embedding 调用报错 |
| NapCat | `/root/napcat.log` | 二维码、登录状态、真实 UIN |
| 配置抢改 | `/root/.webui_watch.log` | 每次"已用备份覆盖空白 webui.json" |
| 清理记录 | `/root/.clean_daemon.log` | 每次清了什么（含删掉的下载包） |

---

## 8. 重装 box 之后怎么部署

**第 1 步：放文件**

```sh
# 解压后把文件放进 App 的 scripts 目录（原样，别改层级）
chmod +x boot_services.sh clean_daemon.sh
```

**第 2 步：先单独验脚本**（绕过 Lua，确认 shell 侧没问题）

```sh
sh /root/boot_services.sh status      # 应输出 {"napcat":"down","astrbot":"down","ollama":"down",...}
sh /root/boot_services.sh ollama      # 应看到 "Ollama 就绪（Ns）" 与 "向量模型: ..."
OLLAMA_MODEL=bge-m3 sh /root/boot_services.sh ollama   # 验传参：日志里的模型名应变
sh /root/clean_daemon.sh              # 应打印六段报告，不删任何东西
```

**第 3 步：重载 Lua**

主页顶栏的刷新图标，或 `bash agent/sandbox reload`。

**第 4 步：验证界面**

打开主页 → 卡片标题应是「全栈状态 · v3.1.0」，五个灯 + 模型按钮 + 自动看护 + 折叠的「账号与端口」。

**第 5 步：改模型（可选）**

点「模型: …（点击修改）」→ 输入新模型 → 看 toast 与通知 → 点「启动 Ollama」验证预热。

---

## 9. 已知边界与坑

1. **App 被杀 = 全停**。Lua 和定时器都活不了。只能靠系统电池白名单，任何脚本都救不了。
2. **没装 Ollama 时灯是灰的**，不是错误。装上后把二进制放 `/root/.ollama/ollama` 或进 PATH 即可。
3. **换了模型必须自己 pull**。改模型名不会自动下载（除非 `OLLAMA_AUTO_PULL=1`）；
   没有模型时服务仍可用，只是 embedding 调用会失败。
4. **模型名有字符白名单**。空格/引号/`$` 等会被拒绝——这是防命令注入，不是 bug。
5. **模型名只影响本包**。AstrBot 内部用哪个 embedding 模型由它自己的配置决定，
   本包只负责把它拉起来并预热指定模型。
6. **Ollama 端口别在 Lua 里改**。`main.lua` 的 `ollama_port` 只是读取，真正的 `OLLAMA_PORT`
   在 `boot_services.sh` 里；两边不一致会导致灯和实际服务对不上。
7. **首次 embedding 会慢**。`ollama_warm` 预热失败只说明模型没加载进内存（或没下载），
   服务仍可用。
8. **路线 B 有竞态**。如果只能用 App 的 launcher，webui.json 的重写和我们的抢改是赛跑；
   有 `start_napcat.sh` 时走路线 A，零竞态。
9. **QQ 号是"猜到"的**。三条同步路径都是文件证据；日志正则只是兜底，
   多实例时不读公共 `/root/napcat.log`，避免串号。识别到就回写，只猜一次。
10. **临时下载包只扫根层**（`ARCHIVE_MAX_DEPTH=1`）。想清更深目录把它调大，
    `archive_guard` 的路径黑名单仍会挡住 AstrBot / napcat_instances / .venv。
11. **`truncate` 保留的是文件头部**。日志被截断后最近的几行会丢，排查问题前先看再清。
12. **`port_open` 依赖 bash 兜底**。Debian 的 `/bin/sh` 是 dash 时没有 `/dev/tcp`，
    已自动回退 `bash -c`；容器里连 bash 都没有才会一直判 down。

---

## 10. 想改哪里

| 需求 | 改哪 |
|---|---|
| 换向量模型（推荐） | **界面：主页「模型: …（点击修改）」** |
| 改默认模型常量 | `main.lua` 41 行 `OLLAMA_DEFAULT_MODEL` + `boot_services.sh` 43 行默认值 |
| 换 Ollama 端口 | `boot_services.sh` 42 行 `OLLAMA_PORT` |
| 让模型缺失时自动下载 | 启动加 `OLLAMA_AUTO_PULL=1` |
| 巡检更勤/更懒 | `main.lua` 1339 行 `WATCH.ms` |
| 看护不拉起 Ollama | `main.lua` 1403 行那段 `if up[op] == false` |
| 加一个开机动作 | `boot_services.sh` 325 行 `case` 加分支 |
| 下载包少清/多清 | `clean_daemon.sh` 186 行 `archive_guard()` 的黑名单 |
| 清更多缓存目录 | `clean_daemon.sh` 104 行 `clean_napcat_media()` 的 `pat` 列表 |
| 状态卡再加一行灯 | `main.lua` 1827 行 `stack_card()` 里加 `row{...}` |
| 换通知样式 | `main.lua` 1438 行 `host.notify{...}` |
| 改 QQ 识别规则 | `main.lua` 820 行 `QQ_SCAN_SH` 里的 `grep -aoE` 正则 |

---

## 11. 从 v2.3.0 升到 v3.1.0

1. 覆盖 `main.lua`、`boot_services.sh` 两个文件（`clean_daemon.sh` 未变，可不动）。
2. `chmod +x boot_services.sh`。
3. 主页刷新键重载 Lua；卡片标题应显示「全栈状态 · v3.1.0」。
4. **不需要迁移数据**：`napcat_instances`、`webui.json`、`ollama_embed_model`（没有就用默认）都不变。
5. 想立刻换模型：点模型按钮改 → `ollama pull 新模型` → 点「启动 Ollama」。

---

## 12. v3.1.0 变更清单（Changelog）

### 向量模型动态配置（核心）

- `main.lua` 新增 `valid_model_name()` / `ollama_env()` / `ollama_boot_service()` / `ollama_check_model()` / `edit_ollama_model()`。
- 状态卡新增可点击按钮 `模型: <当前模型>（点击修改）`，替代原先写死在巡检说明里的模型名。
- 点击 → `host.input` → 校验（`[A-Za-z0-9._:/+-]`，1–120 字符）→ `host.set("ollama_embed_model", v)`
  → 立即重建页面。
- `boot_all` 与后台看护统一通过 `OLLAMA_MODEL="..."` 前缀调起 `boot_services.sh`
  （`boot_all` 走 `start`，看护走 `ollama` 子命令）；手动「启动 Ollama」同样走这条路径。
- 切换后给 `host.toast` + 通知（id 9903）：「向量模型已切换至 xxx，请确保已使用 ollama pull 下载该模型」；
  并在后台 `ollama list` 核实，已下载/未下载各给一条通知。
- 模型名脏值双保险：写入前校验 + 读取时再校验，脏值回退 `nomic-embed-text`。

### `boot_services.sh`

- `OLLAMA_MODEL="${OLLAMA_MODEL:-nomic-embed-text}"`（外部传入优先），头部补用法示例。
- `start` 分支新增一行日志：`向量模型: $OLLAMA_MODEL`，方便在终端确认传参是否生效。
- 其余逻辑不变（Ollama → NapCat → AstrBot 顺序、11434 轮询、预热、防重置、清理守护）。

### 界面

- 卡片标题改为 `全栈状态 · v3.1.0`（版本号来自 `APP_VERSION` 常量，单一来源）。
- 新增模型按钮一行；巡检说明不再重复模型名（避免两处不一致）。

### 文档

- 《代码导读》升到 v3.1.0，新增 **§6.1 如何自定义向量模型**（三步操作、模型名合法写法、回退方式）。

---

### 历史：v2.3.0 变更摘要

- **Ollama 全栈整合**：`boot_services.sh` 在 AstrBot 之前拉起 `ollama serve` 并轮询 11434，
  就绪后预热模型；`stack_card` 增加 Ollama 灯；`boot_all` / `stop_all` / 看护纳入 Ollama。
- **清理强化**：`clean_daemon.sh` 新增临时下载包清理（`*.tar.zst` / `*.zip`，>60 分钟，六道闸）。
- **UI 单卡化**：删除 `astrbot_card`（即 `quick_start_card`）与 `napcat_card`，新增两个 WebUI 快捷入口。
- **QQ 号修复**：`onebot11_<uin>.json` / `webui.json autoLoginAccount` / 日志正则 三路还原并回写。
- **隐患修复**：`port_open` 兼容 dash；`NC.detect_qq` 在目录不存在时用 `pcall` 兜住。
