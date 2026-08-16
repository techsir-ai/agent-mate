260816.01

# Agent Mate — 开发设想与路线

> 由 agentbase-setup 与 dsh-easy-up 两个项目合并而来。以 dsh-easy-up 为实现主体(当前版),agentbase-setup 为总纲;agentbase 中有、dsh 中无的功能列入下一版本。

## 一、产品定位

在 Windows 上把 DeepSeek Harness(dsh)一键跑起来:从零装好前置环境,用官方方式部署 dsh,把 dsh web 做成点击开关的服务,并提供 SSH 隧道便于远程访问本机。

## 二、范围

- 当前版只围绕 dsh 一个 agent,只做 Windows。
- 不做通用多 agent 部署(多 agent 列下一版本)。
- 与 agentbase 合并统一为 Agent Mate,独立成产品。

## 三、技术栈(总则)

- Rust + Tauri 构 GUI(不用 Node/Python 承载,避免"鸡用蛋"自举困境)。
- xterm.js + portable-pty(Windows 走 ConPTY),调用系统 shell。
- 原子化设计为根本:有复用价值的能力独立成可复用模块,接口隔离、保护已测功能。
- 首发 Windows,架构预留三端(三端适配列下一版本)。

## 四、功能模块

功能模块按下述顺序排列。每个功能模块均设状态指示灯:

- **绿灯** = 该项已具备且 dsh 可用。
- **必须前置项**:缺了会阻塞 dsh 部署(红灯需处理)→ 如 Node.js(dsh 硬前置,缺了不可用)。
- **参考/可选提示项**:仅提示有无,缺失不阻塞 dsh → 如 Git(非 dsh 前置,仅提示)。
- 探测为后台静默进行,不作"点击才检测"的报告式监测。

### 4.1 网络环境

- 统一管理所有联网操作所依赖的源(nodejs 下载、npm、github 等),处理国内网络访问。
- 订阅式多源:维护一个固定 URL 的公共仓库列表;从固定 URL 读取各仓库的源列表,再拉到本机。
- 对每个源做延迟 + 带宽测试;用户选定,固化保存。
- 将选定源注入 pty 会话环境(GUI 发起的命令与用户自由输入的命令均继承)。
- github 更新源列表时,重新拉取刷新并自动跟新。
- 取代原"原子网络功能"与"网络容错":一个统一网络环境子项(原子功能作为总则仍成立,仅网络原子功能并入此)。

#### GitHub 分类网络依赖

- **HTTP GET 拉数据**(release 下载 / raw / API / 网页资源):用 GitHub CDN 前缀(ghproxy / ghfast)。
- **git 只读**(clone / fetch,git 协议拉取):走 gitclone.com(全量只读镜像)。
- **网页操作 + git push + fork**:需代理网络(完整会话/认证/网页多请求,cdn 前缀与 gitclone 均不覆盖)。fork 属网页操作。

### 4.2 Node.js 部署

dsh 硬前置(必须项,缺了 dsh 不可用)。

1. 选源:从网络环境读 node 下载源,取当前 LTS 版本号(默认最新 LTS),失败回退内置版本号。
2. 版本选项:默认最新 LTS;旁边标注「DeepSeek Harness 建议使用 22 LTS 与 24 LTS 之间的版本」。
3. 下载 MSI:用 Rust HTTP GET(reqwest,不依赖系统浏览器/curl),可显示下载进度。
4. 静默安装:msiexec /quiet,node/npm 默认入 PATH;corepack 装后补执行。
5. 验证:node -v / npm -v。

### 4.3 Git 部署

非 dsh 部署前置条件,仅作提示(参考项)。

1. 选源:从网络环境读 git/github 源。
2. 探测版本:从 GitHub release API(latest)动态取最新版本号,不硬编码。
3. 下载:用 Rust HTTP GET,可显示进度。
4. 静默安装:Git-<ver>-64-bit.exe /VERYSILENT(Inno Setup 参数)。
5. 验证:git --version。

### 4.4 dsh

#### 4.4.1 dsh 部署

- 安装方式定位:
  - 优先 npm 全局安装(Windows 默认):固定路径、好探测、升级/管理简单、免 UAC。
  - npx:缓存残留累积、升级需 latest、无法"只拉不跑"。
  - git clone:有完整源码,适合开发者/高级用户自管,不纳入一键管理。
  - npm 非全局(项目本地):可执行文件在项目内 node_modules/.bin,升级按项目逐个做,管理麻烦。
- 环境探测(部署之前):
  - 后台静默探测云端最新版本 + 本机安装状态(有无安装、安装方式、路径、版本)。
  - 探测到已有安装 → 删除提示 + 删除按钮,用户自选:
    - npx 安装 → 提示"升级后会产生大量系统垃圾,建议删除并改用 npm 方式部署"。
    - git clone 安装 → 提示"此安装方式更适合开发者"。
    - npm 非全局安装 → 提示"执行和管理相对麻烦,建议用 npm 全局部署"。
    - npm 全局安装 → 目标方式,不提示删除。
  - 删除与否、是否保留自定义版本,由用户决定。
- 部署与升级:
  - 未安装 → npm 全局安装 `npm i -g @deepseek-ai/dsh`(装进用户级 `%APPDATA%\npm`,免 UAC,自动入 PATH)。
  - 升级只对 npm 全局安装提供(检测到新版本显示升级按钮,`npm i -g @deepseek-ai/dsh@latest`)。

#### 4.4.2 dsh 管理(运行时)

- 启动方式:
  - 按钮手动启动:后台运行,屏幕内容写入 log,不占用终端。
  - 安装计划任务(自启动),两个选项:随当前用户登录自动启动;用当前用户身份、随系统启动自动启动,不依赖登录过程。
  - GUI 自启动必然随用户登录(GUI 需图形会话)。
- 状态与端口:
  - 实例检测,UI 显示状态与持续时间。
  - 端口默认 3080,可改(提供定制端口选项)。
  - host 禁止 0.0.0.0,只允许 127.0.0.1。
  - 停止与启动方式无关,通用。

#### 4.4.3 打开 WebUI

- 扫描本机浏览器(有几个提供几个按钮),每个按钮用对应浏览器图标+名称。
- 点击用选定浏览器打开 127.0.0.1:port,port 取 dsh web 服务当前运行时端口。

#### 4.4.4 关键技术认知

- 用户数据在 `$DSH_HOME`(默认 `~/.dsh`),与 npm/npx 安装路径(程序本体)相互独立、互不影响。
- 原生模块:依赖树含 node-pty/koffi 等原生包;平台无预编译产物时走 node-gyp 源码编译。win32-x64 预编译已确认存在。
- dsh 在 Windows 上的 shell:dsh 用 pwsh(pwsh-sandbox/本地)栈,Windows 上挂载 pwsh、禁 bash;**不支持 cmd**(所有 agent 都不支持 cmd,功能缺失过多无法补丁);pwsh 优先,可回退 powershell 5.1。

### 4.5 Shell 部署(pwsh)

- dsh 在 Windows 上**优先使用 pwsh**(PowerShell 7);Windows PowerShell 5.1 有局限、可能出错;**cmd 不支持**。
- **非 dsh 硬前置**:pwsh 是 ssh 部署的配套/被依赖,**硬依赖是 ssh**;仅当 ssh 部署需要时才真正装机。
- **探测**:列出本机所有 shell(cmd / Windows PowerShell / pwsh);若已有 pwsh,则本步骤无需额外安装二进制。
- **无 pwsh 才装**:HTTP GET(GitHub CDN 前缀,由网络环境注入 pty)→ 下载官网 MSI(PowerShell-7.x-win-x64.msi)→ msiexec /quiet 静默安装。
- 安装到官网版常规路径(非 store 版),**符合 OpenSSH server 默认 shell 要求**。
- 状态灯:作为参考/配套项,提示其就绪与否(取决于 ssh 是否需要)。

### 4.6 SSH 隧道

仅指远程端口转发,SSH server 属下一版本。

- 点击该项后的 UI 描述:「把远程主机的一个端口通过 ssh 隧道转发到本机,并在本机使用 localhost:port 来访问」。

#### 表单(分远程/本机)

- **远程**:远程主机 IP、远程主机 SSH server 端口、远程主机用户名、远程主机密码 / 远程主机私钥、远程服务端口。
- **本机**:本机端口(仅一项)。
- **连接控制**:点击「建立连接」;显示持续时间、状态、延迟、已用流量;后台运行,不占用 pty。
- **保存与认证**:填写内容可保存;认证方式可选密码 / 指定私钥 / 系统私钥;可为远程主机部署公钥(部署后免密)。

#### 三栏 UI 布局

- **左栏(最左,窄、宽度固定)**:子项列表;收起 = 只显示图标 + 绿灯,展开 = 加功能描述;仅做收起/展开切换,不调宽度。
- **中栏(中,稍宽、可调)**:子项内容(检测状态、执行的操作等)。
- **右栏(最右,最宽、可调)**:pty 终端。
- 中、右两栏宽度可拖动调节。

## 五、通用组件

- pty 终端作为独立于各功能模块的通用 UI 组件,供各子项复用于命令执行与实时输出。

## 六、下一版本(agentbase 中 dsh 当前版所无、后续添入)

- Python 部署(开发三件套补齐)。
- OpenSSH server 部署(含本机/远程部署公私钥、远程执行命令)。
- WSL 部署。
- 多 agent 部署(hermes / opencode / pi 等)。
- Windows Terminal(已含 pwsh 部署,其余窗口环境)补充。
- 检测 git bash。
- 三端适配(macOS / Linux)。
- 产品双模式 B(生成可复制命令清单,脱机执行)。
- 受众留存与 SiteLink/对外协同策略。
