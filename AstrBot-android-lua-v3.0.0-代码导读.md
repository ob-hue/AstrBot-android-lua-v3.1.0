# AstrBot-android-lua v2.3.0 · 代码导读

> 一句话：在 **Lua 沙盒 App 的原生界面里**，给 Ollama + AstrBot + NapCat 做「一键启动 + 后台看护 + 磁盘清理 + WebUI 直通」。
> 全部逻辑在 App 内部闭环，不依赖任何外部程序。

---

## 0. 版本沿革

| 版本 | 变化 |
|---|---|
| `0.2.1` | 上游原版：主页两张卡（AstrBot / NapCat），手动启停，纯 Lua |
| `v2.2.0` | 新增：全栈状态卡、一键启停、后台看护（60s）、`boot_services.sh`、`clean_daemon.sh` |
| `v2.2.2` | 与 v2.2.0 内容一致，仅重新打包 |
| **`v2.3.0`** | **Ollama 全栈整合；UI 单卡化（删两张卡）；WebUI 双快捷入口；QQ 号显示修复；临时下载包清理** |

**改动范围**：只动了 `main.lua`、`boot_services.sh`、`clean_daemon.sh` 三个文件。
`agent/`、`sandbox/`、`agent.lua`、`AGENTS.md` 共 7 个文件**一字未改**。

| 文件 | v2.2.2 | v2.3.0 |
|---|---|---|
| `main.lua` | 1552 行 | **1905 行** |
| `boot_services.sh` | 250 行 | **350 行** |
| `clean_daemon.sh` | 185 行 | **277 行** |

---

## 1. 目录结构与部署位置

```
scripts/                     ← 这就是包解压后的落地位置
├── main.lua         1905 行   ★ 主脚本：界面 + 业务逻辑
├── boot_services.sh  350 行   ★ 启动全栈（Ollama → NapCat → AstrBot）+ webui.json 防重置
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
│    ├─ app.page("home", fn)      声明式 UI（每次渲染重建）             │
│    ├─ nav.tabs{...}             三个页：主页 / WebUI / 终端           │
│    └─ app.agent_actions{...}    顶栏按钮（在 agent/main.lua 里注册）  │
│                                                                      │
│  状态来源：ctx.running[key]      ← 宿主维护，无需轮询                 │
│  实况补充：host.exec(/dev/tcp)   ← v2.3.0 探 11434 + dashboard        │
│  后台定时：host.interval(ms, cb) ← 退到后台仍运行（App 有保活服务）   │
└────────────────────────────┬─────────────────────────────────────────┘
                             │ host.spawn / host.stop
                             ▼
┌──────────────── 容器（proot Debian，root 环境）──────────────────────┐
│  boot_services.sh   → Ollama(:11434) → NapCat → AstrBot              │
│  clean_daemon.sh    → 日志截断 / 缓存清理 / 超期下载包清理           │
│  /root/ollama.log  /root/astrbot.log  /root/napcat.log               │
└─────────────────────────────────────────────────────────────────────┘
```

**关键设计**：Lua 只负责「按一下」和「读状态」，真正的启停与清理全在 shell 里。
好处是脚本能单独跑、单独调试，改 shell 不用重载 UI。

---

## 3. 三个核心文件

### 3.1 `main.lua`

#### 模块地图

| 模块 | 行 | 作用 |
|---|---|---|
| `ports` | 13–23 | 端口持久化（`host.get/set`） |
| **Ollama 核心** | **25–136** | **端口/模型/安装探测、serve 拉起、停止、实况探测** |
| `GH_PROXIES` / `gh_*` | 145–253 | GitHub 镜像测速与选择 |
| `ENV_STEPS` / `env_*` | 256–609 | 环境安装（base / uv / napcat / astrbot），用 `host.exists` 判断装没装 |
| `astrbot_start_command()` | 611 | 拼出 AstrBot 启动命令，含"环境未装完整就退出"的守卫 |
| `astrbot_toggle(running)` | 622 | AstrBot 启停（`host.spawn` / `host.stop`） |
| `NC` 模块 | 637–1427 | NapCat 实例管理：创建、端口分配、OneBot 配置、launcher、启停、登出 |
| **QQ 号还原** | **680–798** | **`detect_qq` / `is_qq` / `hydrate_qq` / `qq_refresh`（v2.3.0）** |
| `stack_state(ctx)` | 1154 | 汇总状态（含 Ollama 与真实 UIN） |
| `boot_all(ctx)` | 1193 | 启动全栈（Ollama 排最前） |
| `WATCH` + `stack_watch_*` | 1249–1365 | 60s 巡检、退避拉起、通知栏 |
| `stop_all(ctx)` | 1368 | 全部停止（含 Ollama） |
| `napcat_tile` | 1563 | 单账号 UI 行 |
| `accounts_expansion` | 1632 | 账号 + AstrBot 端口（折叠区，替代旧 `napcat_card`） |
| `stack_card` | 1736 | **主页唯一主卡** |
| `do_backup` / `do_restore` | 1799 / 1819 | 备份与恢复 |
| `app.page("home", ...)` | 1892 | 主页组装（3 个区块） |

#### ① Ollama 三件套（25–136）

```lua
ollama_port()        -- 34   读 host.get("ollama_port")，默认 11434
ollama_model()       -- 40   读 host.get("ollama_embed_model")，默认 nomic-embed-text
ollama_installed()   -- 48   探 6 个路径（/root/.ollama/ollama、ubuntu 映射、PATH 三处），5s 缓存
ollama_start(quiet)  -- 86   host.spawn 一段 shell：先探 11434 → 找 BIN → exec serve
ollama_stop()        -- 97   host.stop("ollama") + pkill -f "ollama serve"
ollama_toggle(on)    -- 103  按当前状态启停
live_probe(force)    -- 118  4s 节流，host.exec 探 11434 + dashboard，变化才重建整页
```

`OLLAMA_SERVE_SH`（66–84）就是拉起的 shell 本体：

```sh
PORT=11434
(exec 3<>/dev/tcp/127.0.0.1/"$PORT") && exit 0      # 已在跑就直接退出
BIN=...                                              # 找可执行文件
export OLLAMA_HOST="0.0.0.0:$PORT"
exec "$BIN" serve                                    # 占住 spawn，便于 host.stop
```

> 为什么用 `exec` 而不是 `nohup ... &`：这样 `ctx.running["ollama"]` 才有意义，
> 「全部停止」用 `host.stop("ollama")` 就能干净收掉。

#### ② QQ 号还原（680–798）——修「登录后仍显示未登录」

**旧逻辑的病根**：`ins.qq` 只在创建账号时写成 `""`，之后**从来没有任何代码回填它**。
`NC.detect_qq()` 早就存在，但只被 `ensure_ws` 用来拼文件名，结果没写回实例记录 →
扫码登录成功、`onebot11_<uin>.json` 也生成了，卡片照样写「未登录」。
更糟的是 `write_launcher` 每次启动都用空的 `ins.qq` 重写 `webui.json`，
把 `autoLoginAccount` 一起抹掉。

**v2.3.0 改成三路只读探测 + 回写**：

| 优先级 | 来源 | 实现 | 同步/异步 |
|---|---|---|---|
| ① | 实例记录 `ins.qq` | 已识别 / 手填 | 同步 |
| ② | `config/onebot11_<uin>.json` | 文件名即 UIN，NapCat 登录后才生成 | 同步 |
| ③ | `config/webui.json` 的 `autoLoginAccount` | 正则取数字 | 同步 |
| ④ | 实例 `logs/*.log`（单实例再兜 `/root/napcat.log`） | `grep -aoE` 抓 `uin/QQ/autoLoginAccount`，只 tail 300KB | 异步，8s 节流 |

```lua
NC.hydrate_qq(ins)   -- 703  ①②③ 同步命中就回写 ins.qq 并落库
qq_refresh(force)    -- 766  只给"还没有 QQ 号"的实例跑 ④，命中 → 重建页面
```

关键副作用（这才是真修好了）：

- `NC.start()`（1102）在 `write_launcher` **之前**调 `hydrate_qq` →
  下次写 `webui.json` 时 `autoLoginAccount` 是真实 QQ，快速登录不再被抹掉；
- `NC.edit()`（1084）同样先补一次；
- 看护每轮 `stack_watch_tick` 末尾 `qq_refresh(true)`，登录后最多 60s 内卡片跟上；
- 状态灯从「QQ 已登录 N」变成**直接显示号码**：`QQ 123456789`。

#### ③ 单卡化后的主页（1736–1797）

`astrbot_card`（v2.2.x 里叫 `quick_start_card`）和 `napcat_card` **两个函数已整个删除**，
主页只剩一张「全栈状态」卡：

```
┌ 全栈状态 ─────────────────────────────────────────────┐
│ ● AstrBot 运行中   ● NapCat 1/1   ● Ollama 已就绪      │  ← 第一排三个服务灯
│ ● QQ 123456789     ● 链路就绪                          │  ← 第二排 QQ / 链路
│ [AstrBot WebUI]  [NapCat WebUI]                       │  ← v2.3.0 新增双入口
│ [一键启动全栈]    [全部停止]                            │
│ [清理空间]        [启动/停止 Ollama]                    │
│ 自动看护  ▢                                            │
│ 每 60 秒巡检：Ollama(:11434 / nomic-embed-text) …       │
│ ▸ 账号与端口（1）  ← 折叠区：账号列表 / AstrBot 服务 / 端口 │
└───────────────────────────────────────────────────────┘
```

- 灯的颜色：绿=通；红=该通不通；**灰=没装 Ollama**（不吓人）。
- `NapCat WebUI`：单实例直接开；多实例弹 `host.sheet` 让用户选（每个实例 token 不同）。
- 折叠区里有旧的账号列表（`napcat_tile` 原样复用）和 AstrBot 端口编辑。

#### ④ 后台看护（1249–1365）

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

一轮 `stack_watch_tick`（1298）做的事：

1. `host.exec` 一次探完 **11434 + dashboard + 所有 NapCat WebUI 端口**（`/dev/tcp`，不依赖 nc/curl）
2. **Ollama 不通且已安装 → 拉起**（没装就跳过，不然每分钟白拉一次）
3. AstrBot 不通 → `astrbot_toggle(false)`
4. 每个 NapCat 端口不通 → `NC.start(id)`
5. 带退避：连续失败 3 次后降频为每 10 个周期一次
6. 状态签名变了才更新通知栏：`AstrBot 运行 · Ollama 运行 · NapCat 1/1 · QQ 1`
7. 末尾 `qq_refresh(true)` 补捞真实 UIN

### 3.2 `boot_services.sh`（350 行）

**它解决两个具体故障**：

1. App 自带启动器每次启动都重写 `webui.json`，清空 `autoLoginAccount` 和 token → 每次都要重新扫码；
2. AstrBot 的知识库 / 向量检索依赖本地 embedding，而 Ollama 常常没人拉起 → 功能静默不可用。

| 函数 | 行 | 作用 |
|---|---|---|
| `port_open()` | 51 | **v2.3.0 修复**：`/bin/sh` 是 dash 时没有 `/dev/tcp`，自动回退 `bash -c` 再试一次 |
| `snapshot_config()` | 123 | 只有当前配置**有账号**时才备份到 `/root/.webui.json.keep` |
| `restore_config()` | 114 | 只在当前配置**被清空**时用备份覆盖；已有账号就不碰 |
| `watch_and_restore()` | 141 | 后台 30 秒内每 100ms 调一次 `restore_config`（抢改） |
| `find_ollama()` | 157 | 依次找 `$APP_HOME/.ollama/ollama`、`/root/.ollama/ollama`、`/usr/local/bin`、`/usr/bin`、`PATH` |
| `ollama_warm()` | 166 | 通过 `ollama list` 判断模型在不在；在就 `curl /api/embed`（旧版 `/api/embeddings`）预热一次 |
| `start_ollama()` | 199 | 已通→只预热；没装→`absent`；否则 `nohup ollama serve` + 轮询 11434 最多 `OLLAMA_WAIT` 秒 |
| `start_napcat()` | 225 | 路线 A：走 `start_napcat.sh`（不改写配置）<br>路线 B：走 App launcher + 抢改 |
| `start_astrbot()` | 258 | **必须用 `.venv/bin/python`**（系统 python3 缺 aiohttp） |
| `clean_start()` | 282 | 拉起 `clean_daemon.sh --loop`（带 pid 锁，幂等） |
| `status_json()` | 309 | 输出 `{"napcat":"up","astrbot":"down","ollama":"up",...}` |

**启动顺序（v2.3.0 的关键改动）**：

```
snapshot_config
  → clean_start                     # 幂等拉起清理守护
  → start_ollama      ★ 11434 就绪 + 向量模型预热
  → start_napcat
  → start_astrbot     ★ 此时 embedding 一定可用
  → status_json
```

**Ollama 没装不是错误**：只把 `OLLAMA` 标成 `absent`，其余照常启动。
模型缺失也只在日志里提示一句，不会卡住开机（要自动拉模型就设 `OLLAMA_AUTO_PULL=1`）。

用法：

```sh
sh boot_services.sh            # 启动全栈（Ollama → NapCat → AstrBot）
sh boot_services.sh status     # 状态 JSON（三服务）
sh boot_services.sh stop       # 全停（含 pkill ollama serve）
sh boot_services.sh ollama     # 只拉起 Ollama 并预热
sh boot_services.sh snapshot   # 手动备份当前配置
sh boot_services.sh clean      # 后台起定时清理守护
```

### 3.3 `clean_daemon.sh`（277 行）

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
| **5** | **临时下载包** | **`clean_tmp_archives()` 226** | **`/root/` 与 `/storage/emulated/0/Download/` 下超 60 分钟未改的 `*.tar.zst` / `*.zip`** |
| 6 | 体检 | `report()` 250 | 打印占用 Top 8 + 下载目录大小 + 磁盘使用率 |

#### 新增：临时下载包清理（186–248）

```sh
ARCHIVE_KEEP_MIN=60     # 超过 60 分钟未修改才算临时
ARCHIVE_MAX_DEPTH=1     # 只扫根层，不钻进子目录
```

`archive_guard()` 六道闸，逐项校验，任一不过就拒绝：

| 闸 | 条件 | 拒绝什么 |
|---|---|---|
| ① | 后缀必须是 `*.tar.zst` / `*.zip` | 其它一概不碰 |
| ② | 文件名黑名单 | `*backup*` `*AstrBotBubble*` `*AstrBot-android-lua*` `*plugin*` `*config*` |
| ③ | 路径黑名单 | `*/AstrBot/*` `*/napcat_instances/*` `*/.venv/*` `*/.cache/*` `*/data/*` |
| ④ | `guard()` | `.db` / `webui.json` / `_home` 等 |
| ⑤ | 必须是普通文件且非符号链接 | 软链不跟、不删 |
| ⑥ | `find -mmin +60` 必须成立 | 刚下载的、正在用的不删 |

真删走 `rm -f`（`del_archive()`，212），**不递归**；`--apply` 只打印。

实测行为（隔离目录跑 `--apply --yes`）：

| 文件 | 结果 |
|---|---|
| `napcat-installer.zip`（3h 前） | ✅ 删 |
| `uv-aarch64-unknown-linux-gnu.tar.zst`（3h 前） | ✅ 删 |
| `download.zip`（Download 目录，3h 前） | ✅ 删 |
| `fresh.zip` / `live.zip`（刚下载） | ⛔ 保留（未超期） |
| `AstrBotBubble-backup-20200101.zip` | ⛔ 保留（备份） |
| `astrbot_plugin_foo.zip` | ⛔ 保留（插件包） |
| `AstrBot-android-lua-v2.3.0.zip` | ⛔ 保留（组件包） |
| `old.db` / `notes.txt` / `link.zip`（软链） | ⛔ 保留（类型不符） |

```sh
sh clean_daemon.sh                    # 报告
sh clean_daemon.sh --apply            # 演练
sh clean_daemon.sh --apply --yes      # 真删
sh clean_daemon.sh --loop             # 每 7200 秒自动（守护模式）
```

**绝对不碰**：`napcat_instances/*_home`（登录态+聊天记录）、`AstrBot/data/*.db`、
`.venv`、`webui.json`、任何令牌文件、任何名字像备份/插件/组件包的压缩包。

---

## 4. 点一次「一键启动全栈」，发生了什么

```
用户点击
  └─ stack_card 的 onTap → boot_all(ctx)
       └─ host.exists("/root/boot_services.sh") == true
            └─ host.spawn("/bin/sh /root/boot_services.sh start", "全栈启动", "stack_boot")
                 ├─ key = "stack_boot" → ctx.running["stack_boot"] = true
                 ├─ 输出流式进「终端」页
                 └─ 3 秒后 live_probe(true) 让灯跟上

容器里：
  snapshot_config → clean_start → start_ollama
    ├─ 已通 11434？→ 只预热模型
    ├─ 没装 ollama？→ OLLAMA=absent，继续
    └─ 否则 nohup ollama serve + 轮询 11434（最多 30s）→ ollama_warm
  → start_napcat（路线 A/B + 抢改 webui.json）
  → start_astrbot（.venv/bin/python）
  → status_json：{"napcat":"up","astrbot":"up","ollama":"up","at":"..."}

60 秒后：
  stack_watch_tick → 一次探完 11434 + dashboard + 各 WebUI 端口
    → 谁不通拉起谁（Ollama 仅在已安装时）→ 通知栏更新 → qq_refresh 补捞 UIN
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

-- 文件
host.read_file(p) / host.write_file(p, s) / host.exists(p) / host.mkdirs(p) / host.list_dir(p)

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
| | `ollama_embed_model` | `nomic-embed-text` | 状态卡上显示的向量模型名 |
| `boot_services.sh` | `NAPCAT_PORT` / `ASTRBOT_PORT` | 6100 / 6185 | 探测端口 |
| | `OLLAMA_PORT` | 11434 | Ollama 端口 |
| | `OLLAMA_WAIT` | 30 | 等 11434 就绪的秒数 |
| | `OLLAMA_MODEL` | `nomic-embed-text` | 预热的向量模型 |
| | `OLLAMA_AUTO_PULL` | 0 | 1=模型缺失时自动 `ollama pull`（需联网） |
| | `WEBUI_KEEP` | `/root/.webui.json.keep` | 配置备份路径 |
| `clean_daemon.sh` | `KEEP_DAYS` | 3 | 缓存保留天数，0=全清 |
| | `LOG_CAP_MB` | 5 | 日志上限 |
| | `CLEAN_INTERVAL` | 7200 | 守护模式间隔 |
| | `ARCHIVE_KEEP_MIN` | 60 | 下载包超期分钟数 |
| | `ARCHIVE_MAX_DEPTH` | 1 | 扫描深度（1=只扫根层） |
| | `STORAGE_DOWNLOAD` | `/storage/emulated/0/Download` | 下载目录 |

---

## 7. 日志在哪

| 日志 | 位置 | 看什么 |
|---|---|---|
| **Lua 运行日志** | 设置页顶栏「Lua 日志」 | 脚本加载错误、`host.log`、`print` |
| 脚本输出 | App「终端」页 | `boot_services.sh` / `clean_daemon.sh` 的实时输出 |
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
sh /root/boot_services.sh ollama      # 只验 Ollama：应看到 "Ollama 就绪（Ns）"
sh /root/clean_daemon.sh              # 应打印六段报告，不删任何东西
```

**第 3 步：重载 Lua**

主页顶栏的刷新图标，或 `bash agent/sandbox reload`。

**第 4 步：验证界面**

打开主页 → 最上面是「全栈状态」卡：五个灯 + 两排按钮 + 自动看护 + 折叠的「账号与端口」。

**第 5 步：开自动看护**

卡上「自动看护」开关默认开；关掉的话只手动。

---

## 9. 已知边界与坑

1. **App 被杀 = 全停**。Lua 和定时器都活不了。只能靠系统电池白名单，任何脚本都救不了。
2. **没装 Ollama 时灯是灰的**，不是错误。装上后把二进制放 `/root/.ollama/ollama` 或进 PATH，
   下次巡检/点「启动 Ollama」即可变绿。
3. **首次 embedding 会慢**。`ollama_warm` 预热失败只说明模型没加载进内存（或没下载），
   服务仍可用；想自动下载模型设 `OLLAMA_AUTO_PULL=1`。
4. **Ollama 端口别在 Lua 里改**。`main.lua` 的 `ollama_port` 只是读取，真正的 `OLLAMA_PORT`
   在 `boot_services.sh` 里；两边不一致会导致灯和实际服务对不上。
5. **路线 B 有竞态**。如果只能用 App 的 launcher，webui.json 的重写和我们的抢改是赛跑；
   有 `start_napcat.sh` 时走路线 A，零竞态。
6. **`ctx.running` 只在页面渲染时读**，但 v2.3.0 加了 `live_probe`（端口实况，4s 节流），
   手动启停 / 脚本拉起都能在几秒内反映到灯上。
7. **QQ 号是"猜到"的**。三条同步路径都是文件证据；日志正则（④）只是兜底，
   多实例时**不会**去读公共的 `/root/napcat.log`，避免串号。识别到了就回写实例记录，
   所以只会猜一次。
8. **临时下载包只扫根层**（`ARCHIVE_MAX_DEPTH=1`）。想清更深目录把它调大——
   `archive_guard` 的路径黑名单仍会挡住 AstrBot / napcat_instances / .venv。
9. **`truncate` 保留的是文件头部**。日志被截断后最近的几行会丢，排查问题前先看再清。
10. **`port_open` 依赖 bash 兜底**。Debian 的 `/bin/sh` 是 dash 时没有 `/dev/tcp`，
    v2.3.0 已自动回退 `bash -c`；容器里连 bash 都没有才会一直判 down。

---

## 10. 想改哪里

| 需求 | 改哪 |
|---|---|
| 巡检更勤/更懒 | `main.lua` 1249 行 `WATCH.ms` |
| 换 Ollama 端口/模型 | `boot_services.sh` 36–39 行 `OLLAMA_PORT` / `OLLAMA_MODEL` |
| 让模型缺失时自动下载 | 启动加 `OLLAMA_AUTO_PULL=1` |
| 看护不拉起 Ollama | `main.lua` 1313 行那段 `if up[op] == false` |
| 加一个开机动作 | `boot_services.sh` 319 行 `case` 加分支 |
| 下载包少清/多清 | `clean_daemon.sh` 186 行 `archive_guard()` 的黑名单 |
| 清更多缓存目录 | `clean_daemon.sh` 104 行 `clean_napcat_media()` 的 `pat` 列表 |
| 状态卡再加一行灯 | `main.lua` 1736 行 `stack_card()` 里加 `row{...}` |
| 换通知样式 | `main.lua` 1347 行 `host.notify{...}` |
| 改 QQ 识别规则 | `main.lua` 731 行 `QQ_SCAN_SH` 里的 `grep -aoE` 正则 |

---

## 11. 从 v2.2.2 升到 v2.3.0

1. 覆盖 `main.lua`、`boot_services.sh`、`clean_daemon.sh` 三个文件（其余不动）。
2. `chmod +x boot_services.sh clean_daemon.sh`。
3. 主页刷新键重载 Lua。
4. **不需要迁移数据**：`napcat_instances` 记录格式未变，`webui.json` / `onebot11*.json` 未变。
5. 升级后第一次打开主页，等 3–5 秒：QQ 号会被自动识别并写回实例记录，
   Ollama 灯会从灰/红变绿（若已安装）。

---

## 12. v2.3.0 变更清单（Changelog）

### 全栈整合 Ollama

- `boot_services.sh`：新增 `OLLAMA_PORT/WAIT/MODEL/AUTO_PULL`；`find_ollama()`、`ollama_warm()`、
  `start_ollama()`；`start` 分支把 Ollama 放到 **NapCat / AstrBot 之前**，用 `/dev/tcp` 轮询
  11434 最多 30 秒，就绪后预热本地向量模型；`status_json` 增加 `ollama` 字段；
  `stop` 同时收掉 `ollama serve`；新增 `ollama` 子命令。
- `main.lua`：`stack_card` 增加 Ollama 状态灯（未安装显示灰色）；`boot_all` 原生兜底路径
  同样先拉 Ollama（错峰 1.5s）；`stop_all` 纳入 Ollama；`WATCH` 探测 11434，
  不通且已安装时自动拉起（未安装跳过，避免每分钟空拉）；通知栏 body 增加 Ollama 状态。

### 强化清理

- `clean_daemon.sh`：新增第 5 类 `clean_tmp_archives()`，清 `/root/` 与
  `/storage/emulated/0/Download/` 下超过 60 分钟未修改的 `*.tar.zst` / `*.zip`；
  新增 `archive_guard()`（六道闸：后缀 / 文件名黑名单 / 路径黑名单 / `guard()` / 普通文件非软链 / `-mmin`）；
  真删只走 `rm -f` 不递归；体检段落顺带打印下载目录占用。

### UI 极致精简 + WebUI 入口

- 删除 `astrbot_card`（v2.2.x 中名为 `quick_start_card`）与 `napcat_card` 两个函数。
- 新增【AstrBot WebUI】【NapCat WebUI】两个快捷按钮；多实例时用 `host.sheet` 选账号。
- 账号列表、AstrBot 启停、端口编辑收进 `stack_card` 的「账号与端口」折叠区。
- 状态灯从 4 个增至 5 个（新增 Ollama），灯数据改为「端口实况优先」。

### 修复 QQ 号显示

- 新增 `NC.is_qq()` / `NC.hydrate_qq()` / `qq_refresh()`：`onebot11_<uin>.json` →
  `webui.json autoLoginAccount` → 日志正则，三路识别并回写实例记录。
- `stack_card` / 状态灯 / `napcat_tile` 统一用识别后的真实 UIN 显示，不再写死「未登录」。
- `NC.start` / `NC.edit` 在 `write_launcher` 前先回填，避免 `webui.json` 的
  `autoLoginAccount` 被空值覆盖（顺带修好快速登录）。

### 顺带修掉的两个隐患

- `boot_services.sh` 的 `port_open()` 在 dash（`/bin/sh` 无 `/dev/tcp`）下永远判 down，
  现在自动回退 `bash -c`。
- `NC.detect_qq()` 在 config 目录不存在时 `list_dir` 返回 nil 会抛错，
  `hydrate_qq` 里用 `pcall` 兜住，避免整页渲染失败。
