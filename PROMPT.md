# 可粘贴的提示词

两种用法，按你手里有没有本仓库的文件二选一。

---

## 用法 A：把仓库文件一起丢给 AI（推荐，成功率最高）

把 `README.md`、`install-dsh-phone-access.ps1` 一起上传/粘贴，然后发送下面这段：

```
附件是我要给 DeepSeek Harness 桌面版加"手机访问"的实现和一个自包含安装器。请在我的
Windows 机器上照做，不要重新发明方案：

1. 先读 README.md 的"架构""安全模型""排查表"三节，确认你理解：
   - 64602 是反代到桌面端自己的 harness 进程（共享会话），64601/64600 是独立 host；
   - 桥会注入 owner 级 cookie，PIN 门是必需的，不是可选项。
2. 按顺序检查前置条件：DSH 是否装在默认路径、手机与电脑是否同一网段、
   当前 PowerShell 是否管理员。缺什么先告诉我，别猜。
3. 以管理员身份运行：
   powershell -ExecutionPolicy Bypass -File .\install-dsh-phone-access.ps1 -InstallTask
4. 逐条验证并把实际输出贴给我：
   a) Get-NetTCPConnection -LocalPort 64600,64601,64602 -State Listen  → 三条都在，且
      LocalAddress 等于当前 LAN 地址（不是旧 IP）；
   b) curl 走 /e/<PIN> 得到 302，再带 cookie 取 / 得到 200；
   c) Get-ScheduledTask -TaskName 'DSH LAN Phone Access' 状态为 Ready/Running；
   d) 桌面端重启一次后，等 30 秒再跑 a)，确认端口跟到了新的 loopback 端口。
5. 汇报时给我：两个入口 URL、PIN 存放位置、防火墙当前放行范围、以及一台新手机
   第一次访问的完整步骤。

硬性约束：
- 绝对不要使用 0.0.0.0 绑定（DSH 会拒绝，且等于把 RCE 挂到网上）；
- 绝对不要把任何端口映射到公网、不要用 ngrok/cloudflared 之类隧道；
- 不要把 %USERPROFILE%\.dsh\.credentials.yaml、config.ps1、任何 *.log 的内容
  写进聊天记录或提交到仓库；
- 不要用 New-NetIPAddress 给网卡加固定 IP（在 DHCP 接口上会把网卡切成手动模式，
  直接把网络搞断；这是本项目明确踩过的坑）。
```

---

## 用法 B：只有这段文字，让 AI 从零实现

不提供代码，只给不变量和验收标准。适合想看 AI 自己实现、或想移植到 Linux/macOS 的人。

```
目标：让同一局域网里的手机浏览器能用上 DeepSeek Harness 桌面版（Windows 打包版），
并且手机上能继续电脑上已有的那条会话。

背景约束（这些是事实，不要试图绕过，只能满足）：
1. DSH 的 `dsh web` 只接受具体的网卡 IP，拒绝 0.0.0.0；
2. 会话是单写者的：另起一个 `dsh web` 进程去打开桌面端正在用的会话，会返回
   error.sessionInUse（"当前会话已被占用"）。要共享会话，只能连桌面端自己的那个
   harness 进程；
3. 桌面端的 harness host 只监听 127.0.0.1 的随机端口，它的 launch token 只存在于
   进程内存里，拿不到；
4. 但它的浏览器会话签名密钥是持久的，存在 %USERPROFILE%\.dsh\.credentials.yaml 的
   records["client-connection/browser-session"].payload.secret（base64url，32 字节）。

请交付两样东西：
A. `dsh web` 直连入口：绑定 LAN 地址，用官方 `?token=` 换 cookie，包一层
   PIN 跳板（GET /e/<PIN> → 302 到带 token 的 URL，其余路径 404，绝不能做成开放跳转）。
B. 反代入口：把 LAN 端口转发到桌面端的 loopback 端口，使手机与电脑共享同一会话。

必须满足的不变量（照抄，每一条都有原因）：
1. 反代必须把 Host 和 Origin **同时**改写成 127.0.0.1:<端口>；只改 Host 会被
   Host/Origin 同源栅栏判 403，响应体只有一个 "forbidden"。
2. cookie 必须按 authority 绑定：
     name  = "dsh-auth-" + base64url(sha256(authority))
     body  = base64url(JSON.stringify({version:1, authority, issuedAt, expiresAt}))
     value = "v1." + body + "." + base64url(hmacSHA256(secret, body))
   并把请求里所有 dsh-auth-* 的 cookie 丢掉，只注入这一个。
3. 每个上游连接只承载一个请求：去掉客户端的 Connection 头，非 WebSocket 一律
   发 `Connection: close`。否则浏览器会复用 keep-alive 连接，后续请求不经改写直接
   打上游，被栅栏拒成 403（这是最难查的一个坑）。
4. 反代端口必须自带门：GET /e/<PIN> 下发一个 HttpOnly gate cookie，其余请求没有
   该 cookie 一律 401。因为注入的 cookie 是 owner 级，端口等于完全控制权。
5. 需要三个监听：64600（独立 host）/64601（PIN 跳板）/64602（反代）。三者都要由
   一个巡检进程保活，巡检每轮要做两件事：
   - 解析「当前 LAN 地址」（取默认路由所在网卡的 IPv4）；
   - 判断端口是否**绑在这个地址上**。只判断"有没有在 LISTEN"不够：网卡换 IP 后旧
     监听仍然显示 LISTEN，但对手机已经不可达，必须杀掉重建。
6. 巡检不能是桌面端的子进程。桌面端重启会清掉整棵进程树，把巡检一起带走。
   用 Windows 任务计划程序托管（登录触发 + 每 5 分钟保活触发 + 互斥锁防重复）。
7. 桌面端换 loopback 端口后，反代要自动跟随：每轮重新扫描 127.0.0.1 上
   commandLine 含 `dsh-desktop-host\lib\index.js` 的进程，读它的监听端口，写进一个
   port 文件，反代每次连接现读；端口变化则重建反代。
8. 防火墙只放行本地子网（LocalSubnet）；能收紧到单台手机 IP 更好，但要提醒用户
   IP 变了会连不上。
9. 明文 HTTP：必须在文档里写清 cookie 可被同网段嗅探，并给出 TLS/组网选项。
10. 不要用 New-NetIPAddress 给 DHCP 网卡加固定地址（会把接口切成手动模式、断网）；
    要固定地址请在路由器做 DHCP 保留。

验收标准（必须逐条实测并把输出贴出来）：
- 三个端口都 LISTEN 在**当前** LAN 地址上；
- /e/<PIN> → 302 → / → 200（拿到首页 HTML）；/api/* 不出现 401/403；
- 手机浏览器（或等价的窄视口浏览器）能打开，并且打开电脑上正在跑的那条会话时
  不再出现"当前会话已被占用"；
- 手动杀掉三个进程之一，30 秒内自动恢复；杀掉桌面端进程后重启它，反代能跟到新端口。

输出要求：先给结论（能用/不能用），再给文件清单与关键代码，最后给一份"出问题怎么查"
的对照表。所有路径、IP、PIN 用占位符，不要泄露我机器上的真实值。
```

---

## 附：一句话版本（偷懒用）

```
参考 github 上的 dsh-phone-access 思路，给 DeepSeek Harness 桌面版加手机访问：
64600 独立 dsh web + 64601 PIN 跳板 + 64602 反代到桌面端自身 harness 进程（共享会话）。
反代要同时改 Host 和 Origin、按 authority 签 dsh-auth-* cookie、每连接一请求发
Connection: close、并自带 /e/<PIN> gate cookie。三个端口由一个跑在任务计划程序下的
巡检保活，判断"是否绑在当前 LAN 地址"而不是"是否 LISTEN"。先给我方案再动手。
```
