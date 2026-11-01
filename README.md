# dsh-phone-access

把 **DeepSeek Harness 桌面端**（Windows 打包版）开放给同一局域网的手机浏览器。
不改动 DSH 安装目录内的任何文件，全部是外部独立进程 + 脚本，可整目录删除。

> 目标场景：电脑上正在跑的长会话，想在手机上接着看/接着发；或者出门在外用手机、笔记本留在家里连着 WiFi。

---

## 它解决的两个真实问题

**1. 桌面端默认只听 `127.0.0.1`，手机根本连不进来**
DSH 的 `dsh web` 必须绑定一个**具体的网卡地址**，并且显式拒绝通配地址（`0.0.0.0`）：

```
--host   bind address: one concrete IPv4 or IPv6 literal of a local interface;
         wildcard addresses are rejected
```

**2. 另起一个 `dsh web` 进程，打不开桌面端正在用的会话**

```
error.sessionInUse  →  "当前会话已被占用，可能是其他正在运行的 DSH 导致的
                        （如其他 dsh web、桌面端），请退出其他正在运行的 DSH 后重试。"
```

会话是**单写者**的。所以「手机上继续电脑那条会话」不能靠起第二个 harness —— 必须让手机连到**桌面端自己的那个 harness 进程**。这就是 `64602` 反代存在的原因。

---

## 架构

```
手机浏览器 ──HTTP──┐
                   ├─▶ :64602  lan-bridge.mjs   反代 ──▶ 127.0.0.1:<桌面端 host 端口>
                   │      · 重写 Host / Origin 为 loopback authority
                   │      · 注入 owner 级 dsh-auth-* cookie（用凭据库里的持久签名密钥）
                   │      · /e/<PIN> 发 gate cookie，其余请求无 cookie 一律 401
                   │      → 与桌面端共享同一条会话、同一套插件与权限
                   │
                   └─▶ :64601  phone-entry.mjs  PIN 跳板 ──302──▶ :64600/?token=...
                          :64600  dsh web（独立 host，官方 token 鉴权）
                          → 自己的一套会话，和桌面端互不干扰
```

三个监听都由 `lan-watchdog.ps1` 巡检：每 15 秒确认**端口绑在当前 LAN 地址上**，
不在就重建；桌面端换端口时自动重建桥。巡检由**任务计划程序**托管（登录触发 + 每 5 分钟保活），
不是桌面端的子进程 —— 否则桌面端一重启，整棵进程树被清掉，巡检跟着死。

---

## 前置条件

| 项 | 要求 |
|---|---|
| 系统 | Windows 10/11 |
| DSH | 桌面打包版，默认装在 `%LOCALAPPDATA%\Programs\DeepSeek Harness`（可用 `-DshRoot` 覆盖） |
| 网络 | 手机与电脑在**同一网段**（同一个 WiFi/路由器，非访客网络） |
| 权限 | 加防火墙规则 / 注册计划任务需要管理员 |
| 可选 | Python 3 + `pip install qrcode pillow`（只用于生成二维码 PNG） |

---

## 快速开始

```powershell
# 1) 管理员 PowerShell
cd <本目录>
powershell -ExecutionPolicy Bypass -File .\install-dsh-phone-access.ps1 -InstallTask
```

脚本会：

1. 自动探测 **DSH 安装目录**、**当前 LAN 地址**（取默认路由所在网卡），生成 8 位 PIN；
2. 把组件写到 `%USERPROFILE%\Documents\dsh-phone-access\`（`-OutDir` 可改）；
3. 注册计划任务 `DSH LAN Phone Access`（登录触发 + 5 分钟保活）；
4. 为 64600/64601/64602 添加**仅本地子网**的入站规则；
5. 拉起三个监听并打印两个入口 URL。

输出形如：

```
shared session (Desktop app):  http://<LAN_IP>:64602/e/<PIN>
separate host (own sessions):  http://<LAN_IP>:64601/e/<PIN>
```

手机扫码或直接打开第一条。首次进入会种 cookie（桥的 gate cookie 30 天、官方会话 cookie 30 天），
之后同一浏览器直接访问即可。

**任一端口都别暴露到公网。** 见下面的安全模型。

---

## 两个入口怎么选

| 入口 | 连到谁 | 会话 | 何时用 |
|---|---|---|---|
| `:64602/e/<PIN>` | 桌面端自己的 harness 进程 | **和电脑同一条**，实时同步 | 日常：手机上接着电脑的对话 |
| `:64601/e/<PIN>` | 独立的第二个 `dsh web` | 手机自己的一套 | 桌面端关掉了，或想手机单独跑任务 |
| `:64600/?token=…` | 同上（裸 token） | 同上 | 兜底；token 每次 host 重启都会变 |

同一会话**两端不要同时输入**：两边都能看到彼此进度，但同时提交会有先后竞争。

---

## 安全模型（必读）

1. **64602 等价于本机完全控制权。** 桥注入的是 owner 级 cookie，进去就能执行命令、读写文件。
   所以它自带 PIN 门：`GET /` 不带 `dsh-gate` cookie 一律 401，只有走过 `/e/<PIN>` 才发 cookie。
2. **PIN 就是凭证。** 泄露 = 交出机器。轮换方式：改 `config.ps1` 里的 `PhonePin`，
   重启 `phone-entry.ps1` 与 `lan-bridge.ps1`（gate cookie 与 PIN 绑定，会自动失效）。
3. **明文 HTTP。** cookie 在局域网内是明文传输；同网段的攻击者可以嗅探/ARP 欺骗。
   要更强就自己加一层：WireGuard/Tailscale 组网、或在 DSH 侧用 `--tls-cert/--tls-key` 上 TLS。
4. **防火墙只放行需要的来源。** 默认 `-RemoteAddress LocalSubnet`；
   可以收紧到单台手机：
   ```powershell
   Set-NetFirewallRule -DisplayName 'DeepSeek Harness LAN 64602' -RemoteAddress <PHONE_IP>
   ```
   ⚠️ 手机换网段/IP 变了就会连不上 —— 每次换网络记得同步。
5. **不要提交生成的运行时文件。** `config.ps1`（含 PIN 与路径）、`*.log`（含 token）、
   `desktop-host.port`、`*.pid`、`qr-*.png` 都已在 `.gitignore` 里。
   本仓库的源码文件**不含任何凭据**。
6. **DSH 的凭据库本身不要外传**：`%USERPROFILE%\.dsh\.credentials.yaml`
   （里面有浏览器会话签名密钥和账号 token）。桥只读它，不复制、不外发。

---

## 版本耦合（会坏的地方）

桥不是官方 API，它依赖 DSH 0.2.1-rc.3 的三处内部实现。升级 DSH 后若 `64602` 变成 401，
按顺序看这三处：

| 依赖点 | 在 `dsh-client-connection` 里的位置 | 症状 |
|---|---|---|
| cookie 载荷版本 `COOKIE_PAYLOAD_VERSION` 与字段名 `version/authority/issuedAt/expiresAt` | `encodeCookie` / `decodeCookie` | 401 |
| 凭据记录键 `credentialKey('client-connection','browser-session')` 与 `payload.secret` | `initializeSecret` | 401 / 桥启动即 500 |
| `/api` 的 Host/Origin 栅栏规则 | `isTrustedApiRequest` | 403 `forbidden` |
| 桌面端 host 子进程的命令行形状 `dsh-desktop-host\lib\index.js` | 端口发现用 | 桥报 “desktop harness host not found” |

**降级路径不依赖内部**：`64601`/`64600` 走的是官方 `--host` + `?token=` 机制，
DSH 升级不会坏（只是不共享桌面端会话）。桥挂了就先退回这条。

更正统的长期方案只有一个：等上游给桌面端加 `--host` 或配对码机制，那时本仓库可以整体删除。

---

## 排查表（都是实际踩过的）

| 现象 | 原因 | 处理 |
|---|---|---|
| 页面 401「entry link required」 | gate cookie 掉了（清过浏览器数据 / 换了浏览器） | 重开 `/e/<PIN>` 地址 |
| 页面 401「dsh web authentication required」 | 用的是裸 `:64600` 且没有 token/cookie | 改用 `/e/<PIN>` 入口 |
| 打开正常但提示「会话已被占用」 | 连的是 `64601`（独立 host），而桌面端占着这条会话 | 换 `64602`，或先在桌面端切走该会话 |
| 页面空白、`/api/*` 全 403 `forbidden` | 反代让请求走了 keep-alive 复用连接，第二条请求没被重写 | 保留 `Connection: close`（源码里已固定） |
| 手机连不上，电脑上 `curl` 却正常 | 监听还绑在**已失效的旧 IP** 上（`Get-NetTCPConnection` 仍显示 LISTEN） | 巡检会杀旧重建；手工：`restart` 巡检 |
| 换了 WiFi 就全断 | 电脑/手机换了网段；或防火墙 `RemoteAddress` 还是旧 IP | 看 `lan-watchdog.log` 里的 `LAN address: 'x' -> 'y'` |
| 桌面端重启后桥失效 | 桌面端换了 loopback 端口，或桥被当成子树杀掉 | 巡检每轮重查端口；巡检必须由计划任务托管 |
| 手机能进、但点开会话报锁 | 两个 harness 进程争同一会话 | 只保留一个入口在用；或统一走 64602 |

---

## 卸载

```powershell
Unregister-ScheduledTask -TaskName 'DSH LAN Phone Access' -Confirm:$false
Get-NetTCPConnection -LocalPort 64600,64601,64602 -State Listen |
  ForEach-Object { Stop-Process -Id $_.OwningProcess -Force -ErrorAction SilentlyContinue }
'DeepSeek Harness LAN 64600','DeepSeek Harness LAN 64601','DeepSeek Harness LAN 64602' |
  ForEach-Object { Remove-NetFirewallRule -DisplayName $_ -ErrorAction SilentlyContinue }
Remove-Item "$env:USERPROFILE\Documents\dsh-phone-access" -Recurse -Force
```

DSH 安装目录全程未被修改，卸载即回到原状。

---

## 文件清单

仓库里只有这三个文件（源码，不含凭据）：

| 文件 | 作用 |
|---|---|
| `install-dsh-phone-access.ps1` | 自包含安装器：探测环境、**生成**下列运行文件、注册任务与防火墙 |
| `README.md` | 本说明：架构、安全模型、版本耦合、排查表 |
| `PROMPT.md` | 可直接粘贴给 AI 的提示词（带附件版 / 纯规范版 / 一句话版） |

安装器在你的机器上生成、**不入库**（见 `.gitignore`）：

| 生成物 | 作用 |
|---|---|
| `lan-bridge.mjs` | 反代：头重写 + cookie 签名 + PIN 门（唯一依赖 DSH 内部的文件） |
| `phone-entry.mjs` | PIN 跳板：`/e/<PIN>` → 302 到当前 token URL，其余 404 |
| `lan-watchdog.ps1` | 巡检：地址漂移检测、僵尸监听清理、端口跟随 |
| `start-lan-web.ps1` | 起独立 host（`dsh web --host … --port …`） |
| `phone-entry.ps1` / `lan-bridge.ps1` | 两个入口进程的启动器（含环境变量） |
| `show-urls.ps1` | 打印入口 URL，可选生成二维码 PNG |
| `config.ps1` | 本机路径/端口/PIN（**含 PIN，勿提交**） |
| `lan-web.out.log`、`*.log`、`*.pid`、`desktop-host.port` | 运行期状态（日志里含 token） |

---

## 许可证

MIT（或换成你偏好的）。自用工具，无担保：它能把机器交到手机手上，风险自负。
