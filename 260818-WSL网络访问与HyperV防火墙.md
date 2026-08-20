---
agent: organize
model: deepseek-v4-flash
session_id: ses_ff27021a0ffeBhx5RjGz7K5ghZ
mode: knowledge
host_os: windows
hostname: X1CW
ip: 192.168.0.150
source: opencode export ses_ff27021a0ffeBhx5RjGz7K5ghZ
summary: 围绕「WSL 网络访问与 Hyper-V 防火墙」的认知演化链：从 SSH 隧道/端口转发的应用层本质与「地址所有权」原理（转发只能在拥有该 IP 的一端发生），到 Windows 自带转发工具谱系（netsh interface portproxy 早于 WSL、netsh/cmd 语法、pwsh 无端口转发 cmdlet、防火墙只放行不转发），再到 WSL2 三种并列网络模式（NAT 独立 172.x 网段 / Mirrored 共享宿主机 IP 与 localhost / Bridged 与宿主机同网段），并纠正「NAT 配同网段」的概念混淆。核心收敛：Hyper-V 防火墙（NetSecurity 模块三大子集之一 NetFirewallHyperV，17 个命令，WSL VMCreatorId {40E0AC32-46A5-438A-A0B2-2B479E8F2E90}）与 netsh 是「放行 vs 转发」两个层面；因 LoopbackEnabled 默认 true，宿主机↔WSL 回环流量免 Hyper-V 防火墙规则，故 NAT/桥接下 netsh 转发默认无需 Hyper-V 防火墙命令，仅外部/LAN 入站或 GPO/CSP 策略收紧时才需要；WSL 2.0.9+ 默认开启 Hyper-V 防火墙，mirrored 模式并非全自动透传（特定端口如 22 需显式放行）。避坑含多次未查证臆断的教训（socat 自相矛盾、frp 延申、mirrored 绝对化、NAT 下两层必需的错误推断）。
prev:
created: 260818
---

# WSL 网络访问与 Hyper-V 防火墙

## 一、主题与结论

WSL2 的网络访问（宿主机↔WSL 端口互通）由「转发」与「放行」两个独立层面构成：**转发**（netsh portproxy / SSH 隧道）负责把连接送进 WSL；**放行**（Hyper-V 防火墙）负责过滤进入 WSL 虚拟网络的入站流量。由于 WSL 默认 `LoopbackEnabled=true`，宿主机↔WSL 的回环流量**默认免 Hyper-V 防火墙规则**，因此 **NAT/桥接模式下 netsh 转发默认不需要 Hyper-V 防火墙命令**；Hyper-V 防火墙规则仅在**外部/LAN 入站**或**策略（GPO/CSP）收紧**时才必需。

## 二、认知演化链

### 1. 端口转发机制的本质：应用层转发，非协议栈行为

- SSH 隧道 `-L`/`-R` 是**应用层转发**：`-R`（本机执行，把本机端口推给远程机）与 `-L`（远程机执行，从本机拉端口）效果等价，区别只在**谁发起连接、认证方向**；实际选哪种取决于哪端能主动连上另一端（防火墙/NAT 哪边放行）。
- 端口转发 = 在某 IP 上**开监听**并把流量引向另一 IP:端口。这是应用层工具（ssh、netsh、socat、nginx）的行为，不是 TCP/IP 协议栈规定行为，也不是防火墙行为——**防火墙只过滤/放行，不改变流量目的地，做不了转发**。
- 现实佐证：`ssh -L` 绑定端口报 `bind ... Permission denied` 时，若 `netstat`/`Get-NetTCPConnection` 确认端口未被占用，则多半是**安全软件对特定高位端口绑定的应用层拦截**，换一个端口即可（13080 被拦、30810 放行的实例）。

### 2. 「地址所有权」决定转发方向（核心原理）

- 一个进程**只能在自己的网络命名空间里绑定自己接口拥有的 IP**。WSL2 是轻量 VM，它的 IP（如 172.x）归属 WSL 虚拟网卡；Windows 的 IP（宿主机 IP / 127.0.0.1）归属 Windows 网络栈。
- 因此「WSL 内把端口『挂』到宿主机 IP 上」**做不到**——不是有没有工具的问题，而是**地址所有权**的本质限制。
- 宿主机**能控制** WSL 的 IP（虚拟交换机由宿主机分配管理），但那是「控制权」，不等于 WSL 获得宿主机 IP 的**绑定权**；此结论与用什么 shell 无关（不是 bash/pwsh 的问题）。
- 正确能力分配：**转发动作只能发生在拥有目标 IP 的那一端**——Windows 做转发（netsh/ssh -L）可行；WSL 侧工具（socat/frp 客户端）只是「主动出站连出去」，属于另一套架构，**不是**「WSL 内转发到宿主机 IP」。

### 3. Windows 自带转发工具谱系

- **netsh interface portproxy**：Windows 自带端口转发命令。netsh 基础命令 Windows 2000 时代就有，**`interface portproxy` 子命令是后来（Windows Server 2008 / Vista 时期）才加入的**，早于 WSL（2016）。语法为 **netsh/cmd 风格**（`netsh interface portproxy add v4tov4 listenaddress=... listenport=... connectaddress=... connectport=...`），**不是 pwsh cmdlet 语法**；在 pwsh 中直接调用 netsh.exe 即可运行。
- **PowerShell/pwsh：没有端口转发 cmdlet**。最接近的 `New-NetNatStaticMapping` 属 WinNAT 网络地址转换，不是单机端口转发。
- **Windows 防火墙（netsh advfirewall / wf.msc / New-NetFirewallRule）只能放行/阻止，不能转发**。
- 收敛结论：Windows 自带工具中，机对机、局域网内、不做外网的「WSL 端口 → 宿主机端口」转发**只有 netsh portproxy 和 ssh -L 两条路**（ssh -L 需 WSL 装 sshd）；其余（socat、nginx、frp、rinetd）均为第三方，不在 Windows 自带范围。

### 4. WSL2 三种并列网络模式

| 模式 | 网络架构 | WSL 的 IP |
|---|---|---|
| **NAT**（默认） | WSL 在虚拟 NAT 网段后 | **172.x.x.x 私有网段**，与宿主机不同网段，重启会变 |
| **Mirrored** | 与宿主机共享 IP/端口空间 | **共享宿主机 IP**（含 localhost 回环） |
| **Bridged**（桥接） | WSL 像独立设备连入局域网 | **从路由器获取与宿主机同网段的 IP**（需先在 Hyper-V 管理器建外部虚拟交换机，`.wslconfig` 配 `networkingMode=bridged` + `vmSwitch`） |

- **概念纠正**：「NAT 模式 + 同网段 IP」组合**不成立**——NAT 的架构定义决定它在独立私有网段；要同网段 IP 选 **Bridged**。三者是并列关系（`networkingMode` 三选一），不是包含关系。

### 5. Mirrored 模式的边界：非全自动透传

- 微软官方文档（learn.microsoft.com/windows/wsl/networking）：mirrored 模式下宿主机与 WSL 可通过 `localhost` 双向互通（内置能力，含 IPv6、VPN 兼容、Multicast、从 LAN 直接访问 WSL）。
- **但存在关键边界**：WSL 2.0.9+ **默认开启 Hyper-V 防火墙**，它会对 WSL 的入站连接做过滤。要让入站（从 Windows/LAN 连进 WSL 服务）通，**必须显式放行**——实测案例：宿主机 22 端口空着，WSL 内 sshd 监听 22，mirrored 模式仍透传不了，需显式设置。
- 官方文档给出的显式命令（PowerShell）：
  ```powershell
  Set-NetFirewallHyperVVMSetting -Name '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -DefaultInboundAction Allow
  # 或按端口：
  New-NetFirewallHyperVRule -Name "MyWebServer" -DisplayName "My Web Server" -Direction Inbound -VMCreatorId '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -Protocol TCP -LocalPorts 80
  ```
- 注意：这个显式设置是 **Hyper-V 防火墙入站放行**，不是 netsh 转发，也不是「自动全通」。mirrored 共享 localhost 时访问同一服务只需**错开端口**（如两个 ssh：一个 22、一个 2222），不需要转发。

### 6. Hyper-V 防火墙体系（NetFirewallHyperV）

- **表现形式**：不是独立二进制，是 **PowerShell 模块 `NetSecurity` 里的函数（Function）**（Source 列为 NetSecurity），底层走 Windows 防火墙服务。
- **NetSecurity 是总集合（模块名）**，其下三大平行子集（名词前缀不同）：
  1. **NetFirewall** — Windows 防火墙（宿主机自身端口/程序/配置文件过滤），约 15 个命令
  2. **NetIPsec** — IPsec 连接安全/加密认证策略，约 40+ 个命令
  3. **NetFirewallHyperV** — Hyper-V 防火墙（虚拟机/WSL 虚拟网络过滤），**17 个命令**
- 更精确的表述：`NetSecurity` 是**模块名**（决定动词 Get/Set/New...），真正的前缀（名词部分）是 `NetFirewall` / `NetIPsec` / `NetFirewallHyperV` 三组（决定归属哪一类），这一层之下才是具体命令。
- **Hyper-V 防火墙 17 个命令清单**（全在 NetSecurity 模块）：
  - 查询类：`Get-NetFirewallHyperVRule`、`Get-NetFirewallHyperVVMSetting`、`Get-NetFirewallHyperVProfile`、`Get-NetFirewallHyperVPort`、`Get-NetFirewallHyperVVMCreator`（5 个）
  - 创建类：`New-NetFirewallHyperVRule`、`New-NetFirewallHyperVVMSetting`、`New-NetFirewallHyperVProfile`（3 个）
  - 修改类：`Set-NetFirewallHyperVRule`、`Set-NetFirewallHyperVVMSetting`、`Set-NetFirewallHyperVProfile`（3 个）
  - 删除类：`Remove-NetFirewallHyperVRule`、`Remove-NetFirewallHyperVVMSetting`、`Remove-NetFirewallHyperVProfile`（3 个）
  - 启停类：`Enable-NetFirewallHyperVRule`、`Disable-NetFirewallHyperVRule`（2 个）
  - 重命名类：`Rename-NetFirewallHyperVRule`（1 个）
- **VMCreatorId**：WSL 的虚拟机创建者标识为 `{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}`，Hyper-V 防火墙规则用 `-VMCreatorId` 定位 WSL。
- **生效范围**：Hyper-V 防火墙作用于「虚拟机虚拟网络」这一层，**与 WSL 用哪种网络模式（NAT / Mirrored / Bridged）无关**——只要 WSL 仍是 Hyper-V 虚拟机，过滤就作用其上，三种模式下入站都受约束。

### 7. 关键关系收敛：netsh 与 Hyper-V 防火墙在 NAT/桥接下默认不需要配合

- 微软 Hyper-V Firewall 官方文档关键字段：**LoopbackEnabled** — "Tracks if loopback traffic between the host and the container is allowed, **without requiring any Hyper-V Firewall rules**. WSL enables it by default, to allow the Windows Host to talk to WSL, and WSL to talk to the Windows Host."
- 推论：netsh 建立的「宿主机→WSL」连接属于 **host→container 回环流量**，WSL 默认 `LoopbackEnabled=true` → **默认放行，不需要 Hyper-V 防火墙规则**。
- 因此：
  - **NAT + netsh**：只需 netsh，不需要 Hyper-V 防火墙命令（老板的直觉正确）
  - **Bridged + netsh**：同样只需 netsh（或直接连 WSL 的 LAN IP）
  - **Hyper-V 防火墙规则何时才需要**：① 从外部/LAN 其他设备入站连到 WSL（真正的外部入站）；② WSL 的 `LoopbackEnabled` 被改成 `False`；③ 默认 `DefaultInboundAction` 被企业策略（GPO/CSP）改为 Block 且策略生效
  - **mirrored**：宿主访问 WSL 走共享 localhost 也属 loopback（默认放行）；实测「必须显式设置」属于上述例外情形，不是「mirrored 本身强制要求」
- 端口转发本质是「打通端口」的操作：若转发不经过 VM 防火墙拦截（或默认放行），netsh 单独够用；Hyper-V 防火墙指令生效的前提是**它被需要**——只在入站到 WSL 被默认拦截时才显式配置，**不是必然步骤**。

## 三、关键结论

1. **转发与放行是两个层面**：netsh/SSH 隧道是转发器（把连接送进 WSL），Hyper-V 防火墙是过滤器（放行/阻止进入 WSL）；不是覆盖关系，也不是必然二选一。
2. **地址所有权决定转发方向**：转发只能发生在拥有目标 IP 的一端；WSL 无法绑定宿主机 IP。
3. **Windows 自带转发工具仅 netsh portproxy 与 ssh -L**；防火墙只放行不转发；pwsh 无端口转发 cmdlet。
4. **WSL2 三模式**：NAT（默认，172.x 独立网段）/ Mirrored（共享宿主机 IP 与 localhost）/ Bridged（与宿主机同网段）；「NAT 配同网段」不成立，同网段属 Bridged。
5. **NAT/桥接下 netsh 转发默认不需要 Hyper-V 防火墙规则**（LoopbackEnabled 默认 true）；Hyper-V 防火墙规则仅在外部/LAN 入站或 GPO/CSP 策略收紧时必需。
6. **mirrored 非全自动透传**：WSL 2.0.9+ Hyper-V 防火墙默认开启，特定端口（如 22）入站需显式放行（Set-NetFirewallHyperVVMSetting / New-NetFirewallHyperVRule）；同一服务错开端口即可互不干扰，无需转发。

## 四、避坑与误区

1. **未查证就臆断（本主题最深刻的教训，多次发生）**：
   - 曾断言「WSL 内用 socat/frp 也能转发到宿主机 IP」——与地址所有权结论自相矛盾；socat 例子编造不成立（`TCP:宿主机IP:端口` 需要宿主机侧先有监听，且那只是 WSL 连出去，不是把端口挂到宿主机 IP）。
   - 曾把 frp/cloudflared（转发到外部网络的出站型）塞进「WSL 端口→宿主机端口」的回答——答非所问，属延伸回答。
   - 曾把 mirrored 说成「全自动、零配置、无需显式设置」——被老板实测（22 端口透传不了）推翻，官方文档证实入站受 Hyper-V 防火墙约束。
   - 曾断言「NAT 下 netsh + Hyper-V 防火墙两层都必需」——查证 LoopbackEnabled 文档后纠正为「默认不需配合，仅外部入站/策略收紧时必需」。
   - **教训**：涉及 WSL 网络/防火墙的事实性问题，必须先查微软官方文档（wsl/networking、hyper-v-firewall）再作答，不凭记忆。
2. **netsh 与 Hyper-V 防火墙是「转发 vs 放行」两层**：不要混淆为「netsh 覆盖 Hyper-V 防火墙」或「必然两层都要配」；依赖关系看「入站到 WSL 是否被默认拦截」。
3. **端口转发依赖关系**：转发本身是「打通端口」的操作，不是「转发之外还要额外放行」两件独立的事；Hyper-V 防火墙指令生效的前提是「它被需要」。
4. **「NAT + 同网段 IP」组合不成立**：NAT 天生在独立 172.x 网段；要同网段选 Bridged，不是 NAT。
5. **`ssh -L` bind Permission denied 排查**：先 `netstat -ano | findstr <port>` 确认是否被占用；未被占用时可能是安全软件对特定高位端口绑定的应用层拦截，换端口即可（13080 被拦 / 30810 放行的实例）。
6. **命令名易错**：Hyper-V 防火墙设置类命令是 `*NetFirewallHyperVVMSetting`（带双 V），不是 `*NetFirewallHyperVSetting`；以 `Get-Command -Name "*FirewallHyperV*"` 实际输出为准。

## 五、相关条目

- 微软官方文档：WSL 网络访问（learn.microsoft.com/windows/wsl/networking）、Hyper-V Firewall（learn.microsoft.com/windows/security/operating-system-security/network-security/windows-firewall/hyper-v-firewall）
- 本主题源于 agent-mate 项目（dsh-easy-up）「SSH 隧道便于远程访问本机」需求，与 260815-SiteLink-SSH隧道GUI工具可行性调研.md 同源场景
- 输出目录既有条目：260814-dsh平台连通性探测记录.md（Windows 网络/端口相关）
