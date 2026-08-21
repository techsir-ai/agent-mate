260821.14（此日期是最后修改的时候的日期，使用$data获取，此序号只增加，改一次就加一，永不重置）

# Agent Mate — 需求文档


## 一、产品定位

Agent 伴侣，是为了让国内普通小白用户可以方便快捷的使用各种 Agent 。小白用户在 ai agent 时代最主要的障碍有两个，第一是国内网络环境导致安装和使用不畅（是不畅不是不通），第二是小白用户对命令行操作和后台服务的安装管理的不熟悉，而很多 agent 是需要依赖一些命令行操作，也需要把一部分功能做成后台无头服务来管理，不如把命令行操作和服务管理都做成按钮一键式使用，下面细说。一阶段目标：安装 agent 之前，首先要安装开发三件套 nodejs python git 和 pwsh 及其生态（比如npm、pnpm、pip、uv、git操作、非沙箱路径的pwsh）；二阶段目标是要解决各种不同 agent 的部署和使用便利性的问题，这里主要面对国外开发者的 agent ，这些 agent 的安装脚本经常包含很多依赖，在国内网络环境下都不稳定；三阶段目标是解决使用的便利性问题，比如 ssh client/ssh隧道/ssh server ，还有服务管理的问题；四阶段目标是做一些常用的skill或者 agent 使用的软件的评测，做成社区资讯，提供给小白用户。当然，阶段划分除了一阶段之外，其他都是可以随时变动的，只是一个初期的规划 。
另外一个考虑是收入问题，软件免费才能被广泛使用，所以我的收入只能依靠我提供的一些资讯的广告收入，比如我前面讲过的 agent 软件的评测，就可以部署在我自己的网站上，通过广告联盟来赚取点击费，国内和国外分开，这两个站，既要自己针对搜索引擎做seo，也要嵌入 agent mate 软件中，这其实是通过为用户提供数据和服务的方式来赚钱，而不是卖软件。

## 二、技术栈(总则)

- Rust + Tauri 构 GUI(不用 Node/Python 承载,避免"鸡用蛋"自举困境)。
- 原子化设计为根本:有复用价值的能力独立成可复用模块,接口隔离、保护已测功能。
- 首发 Windows,架构预留三端(三端适配列下一版本)。
- 终端统一用通用组件：
  - **pty 终端**:独立于各功能模块的通用 UI 组件,供各子项复用于命令执行与实时输出；
  - **标签结构**:固定一个 console 标签,专用于 GUI 功能执行的信息展示;其余标签页均为交互终端；
  - **交互终端 shell 分类与优先级**:
    - pwsh/powershell/cmd 为一类,按优先级选择:有 pwsh 用 pwsh,缺失回退 powershell,再无回退 cmd；
    - bash 为一类,两个选项:git bash 与 wsl bash,前提是已安装,缺失则回退不可用；
  - **技术选型**:`xterm.js + portable-pty`；

---版本分割线1---
以下内容的版本是v0.01.xxx，而分割线以上的内容不限于某个版本，和版本号无关

## 三、UI框架

- 三栏总布局，左边栏是功能模块列表和状态显示，中栏是选定的功能模块的细化操作和信息展示，右栏为pty终端，也是信息展示屏，也提供交互式命令行功能；
- 左边栏简略展示是功能模块的图标和状态显示，具备环境是绿灯，不具备环境是空白灯，展开则加上功能文字描述，相当于一级菜单的作用；
- 中间栏相当于二级菜单的作用，是已选中功能模块的展开，子操作，信息展示，均在此处；
- GUI 内的功能都以 shell 命令实现，发送到固定的 console 标签执行，方便不同网络目标的环境注入，执行过程与屏幕信息默认记录到 log 文件；用户自行操作则在交互终端标签页中进行。
- 顶部预留菜单栏，底部预留状态栏。
- 预留我自己的网站链接，我可以把动态配置，agent 适配等信息放在网站上，动态加载实现更新，而不用支持一个 agent 就发布一个版本，我也可以把我的评测文章等资料放在我的网站上，挂靠广告联盟实现收入。


## 四、功能模块

功能模块按下述顺序排列。每个功能模块均设状态指示灯,探测均为后台静默进行:

- **绿灯** 该项已具备。
- **红灯** 该项不具备。
- **参考/可选提示项** 不同版本和安装方式的提示。

### 4.1 网络环境

- 统一管理所有目标为国外站的网络访问问题（几乎所有的开发资源原始站都站国外，国内虽然不墙，但是直接访问慢和不稳定），比如：
  git http downloads/nodejs版本探测+同站下载安装包（python和git在github都有）/ nodejs和python各种包管理和版本管理器的安装包下载/nodejs和python包管理的registry/git 操作，这些都叫网络目标；
- 对以上网络目标的不同访问方式我们叫做访问路径，比如代理、直连（透明代理也叫直连）、国内合法镜像站、加速器（灰色地带），路径优先级固定，1.代理（因为如果设置了代理，证明用户有一定认识，有一定基础，并且既然设置了代理，就默认他的网络需要代理，不然设置了干什么）、2.直连（因为透明代理是默认网关，我们分辨不出来用户的网关是否是透明代理，所以优先级2）、3.国内合法镜像站（1.2优先级的原因解释过了，这个基础上，国内合法镜像站是最稳妥的方法）、4.加速器兜底（兜底就是不优先，但是很重要，无路可走情况下的唯一的道路，倾向于自建x-get节点，仓库在https://github.com/techsir-ai/xget）；
- 国内合法镜像站要有3-4个选择（以网络目标.csv表格内容为准）；
- 优先级和多路径，同路径多源配置做成规则，放在公网，软件内拉取，这样换配置不需要更新软件；
- 对每个路径和源做延迟 + 带宽测试;用户选定,固化保存；
- 将选定源注入 pty 会话环境(GUI 发起的安装和部署命令都是shell命令，在软件内设计成与用户自由输入的命令都继承上述方法)；
- 下面所有需要使用网络的均默认使用本项设置，不再赘述；
- 这是一个极重要的特色功能，用来让普通用户也可以降低国内网络的影响。

#### 网络目标及实现方法
网络目标源表见 `网络目标.csv`（单一事实来源，改源目标、镜像/加速器、执行时机、执行方式均以该表为准）。

改源手段分类（供需求表述统一）：A1 文本层兜底（绝对路径命令）/ A2 PATH 包装器（拦截 curl/wget）/ B1 环境变量注入（npm/pip/uv/git 源）/ B2 解释器钩子（覆盖 cmdlet），相互配合实现内存拦截，不落盘不影响操作系统。


### 4.2 Node.js 部署

dsh 硬前置(红灯必处理,缺了 dsh 不可用)。

1. 探测：静默探测本机是否有nodejs实例，如果有，判断安装方式和版本号，静默探测云端最新版本号，如果有实例且云端版本更新，则显示升级按钮；
2. 版本：从官方json取所有版本列表，以下拉框实现，对于22-24之间的版本用粗体表示推荐，旁边标注「DeepSeek Harness 建议使用 22 LTS 与 24 LTS 之间的版本」；
3. 下载 MSI:用 Rust HTTP GET(reqwest,不依赖系统浏览器/curl),可显示下载进度；
4. 静默安装:msiexec /quiet,node/npm 默认入 PATH;之后执行corepack enable；
5. 验证:node -v / npm -v。

### 4.3 Python 部署

### 4.4 UV 部署

### 4.5 Git 部署

非 dsh 部署前置条件,仅作参考/可选提示项。

1. 探测：静默探测本机是否有git实例，如果有，显示版本号，静默探测云端最新版本号，如果有实例且云端版本更新，则显示升级按钮；
2. 版本:静默探测latest版本，安装按钮，点击执行下载；
3. 下载:用 Rust HTTP GET,可显示进度；
4. 安装:Git-<ver>-64-bit.exe /VERYSILENT(Inno Setup 参数)；
5. 验证:git --version。

### 4.6 Shell 部署(pwsh)

- **dsh支持程度**:`pwsh`(PowerShell 7,首选)> Windows PowerShell 5.1(可回退)> `cmd`(不支持)；
  - pwsh:功能完整；
  - Windows PowerShell 5.1:作回退,有局限、可能出错;不支持 cmd,命令需兼容 5.1 语法。
- **探测**:列出本机所有 shell(cmd / Windows PowerShell / pwsh);已有 pwsh 则无需再装；
- **无 pwsh 时显示安装按钮**:HTTP GET→ 下载官网 MSI(PowerShell-7.最新版-win-x64.msi)→ msiexec /quiet 静默安装；
- **安装路径**:官网版常规路径(非 store 版),符合 OpenSSH server 默认 shell 要求。

### 4.7 SSH 隧道

- **范围**:仅指远程端口转发(客户端转发远端已存在的 ssh server 端口到本机访问),不在本机自装 ssh server(后者属下一版本)；
- 点击该项后的 UI 描述:「把远程主机的一个端口通过 ssh 隧道转发到本机,并在本机使用 localhost:port 来访问」。

#### 表单(分远程/本机)

- **远程**:远程主机 IP、远程主机 SSH server 端口、远程主机用户名、远程主机密码 / 远程主机私钥、远程服务端口；
- **本机**:本机端口(仅一项)；
- **连接控制**:点击「建立连接」;显示持续时间、状态、延迟、已用流量;后台运行,不占用 pty。可点击复制隧道本地ip：port方便粘贴；
- **保存与认证**:填写内容可保存;认证方式可选密码 / 指定私钥 / 系统私钥;可为远程主机部署公钥(部署后免密)。密钥可指定，也有按钮生成，默认两种ed25519和rsa，默认使用id_ed25519和.pub、id_rsa和.pub文件名，默认两个公钥加入authorized_keys文件，也就是默认2个私钥/3个公钥文件；
- 可保存多项纪录，可同时建立多条隧道，每条隧道状态单独显示。






---版本分割线2---
以下内容的版本是v0.02.xxx，而分割线以上的内容则是v0.01.xxx的

## 六、后续版本未经讨论定型版(agentbase 中 dsh 当前版所无、后续添入)

- Python 部署(开发三件套补齐)。
- OpenSSH server 部署(含本机/远程部署公私钥、远程执行命令)。
- WSL 部署。
- 多 agent 部署(hermes / opencode / pi 等)。
- Windows Terminal(已含 pwsh 部署,其余窗口环境)补充。
- 检测 git bash。
- 三端适配(macOS / Linux)。
- 产品双模式 B(生成可复制命令清单,脱机执行)。
- 受众留存与 SiteLink/对外协同策略。

### 4.4 dsh

#### 4.4.1 dsh 部署

- 安装方式定位:
  - 只提供 npm 全局安装这一种方式，(Windows 默认):固定路径、好探测、升级/管理简单、免 UAC；
  - 标注为什么不提供其他方式：如：1.npx:优点最简单，一行命令有则运行无则先安装后运行，缺点：缓存残留累积、升级需 latest、无法"只拉不跑"。2.git clone:有完整源码,适合开发者/高级用户自管,如有需求建议手动安装。3.npm 非全局安装:可执行文件在项目内 node_modules/.bin,升级按项目逐个做,管理麻烦。
- 环境探测(部署之前):
  - 后台静默探测云端最新版本 + 本机安装状态(有无安装、安装方式、路径、版本)；
  - 探测到已有安装 → 删除提示 + 删除按钮,用户自选:
    - npx 安装 → 提示"升级后会产生大量系统垃圾,建议删除并改用 npm 方式部署"；
    - git clone 安装 → 提示"此安装方式更适合开发者"；
    - npm 非全局安装 → 提示"执行和管理相对麻烦,建议用 npm 全局部署"；
    - npm 全局安装 → 目标方式,如云端有新版本则显示升级按钮。
  - 删除与否、是否保留自定义版本,由用户决定。
- 部署与升级:
  - 未安装 → npm 全局安装 `npm i -g @deepseek-ai/dsh`(装进用户级 `%APPDATA%\npm`,免 UAC,自动入 PATH)；npm rebuild避开任何可能出现的问题，升级也是一样
  - 升级只对 npm 全局安装提供(检测到新版本显示升级按钮,`npm i -g @deepseek-ai/dsh@latest`)。

#### 4.4.2 dsh 管理(运行时)

- 启动方式:
  - 按钮手动启动:后台运行,屏幕内容写入 专有log（和正常屏幕内容log不同）,不占用终端；
  - 安装计划任务(自启动),两个选项:随当前用户登录自动启动;用当前用户身份、随系统启动自动启动,不依赖登录过程；
  - GUI 自启动必然随用户登录(GUI 需图形会话)。
- 状态与端口:
  - 实例检测,UI 显示状态与持续时间；
  - 端口默认 3080,可改(提供定制端口选项)；
  - 停止与启动方式无关,通用。可以停止其他方式启动的实例。

#### 4.4.3 打开 WebUI

- 扫描本机浏览器(有几个提供几个按钮),每个按钮用对应浏览器图标+名称；
- 点击用选定浏览器打开 127.0.0.1:port,port 取 dsh web 服务当前运行时端口。

#### 4.4.4 关键技术认知

- 用户数据在 `$DSH_HOME`(默认 `~/.dsh`),与 npm/npx 安装路径(程序本体)相互独立、互不影响；
- 原生模块:依赖树含 node-pty/koffi 等原生包;平台无预编译产物时走 node-gyp 源码编译。win32-x64 预编译已确认存在；
- dsh 依赖 pwsh 作为默认 shell(要求见 4.5)。

## dsh 部署需要考虑的问题：
PS C:\Users\changwei> npm install -g @deepseek-ai/dsh@0.1.0-rc.8
npm warn deprecated node-domexception@1.0.0: Use your platform's native DOMException instead

added 1 package, and changed 449 packages in 1m

64 packages are looking for funding
  run `npm fund` for details
npm warn allow-scripts 5 packages have install scripts not yet covered by allowScripts:
npm warn allow-scripts   @deepseek-ai/dsh-subprocess-local@0.1.0-rc.8 (postinstall: node scripts/ensure-spawn-helper.mjs)
npm warn allow-scripts   koffi@3.1.5 (install: node ./cnoke.cjs -P . -D src/koffi --prebuild --release)
npm warn allow-scripts   node-pty@1.2.0-beta.15 (install: node scripts/prebuild.js || node-gyp rebuild; postinstall: node scripts/post-install.js)
npm warn allow-scripts   @google/genai@1.52.0 (preinstall: echo 'preinstall: no-op')
npm warn allow-scripts   protobufjs@7.6.5 (postinstall: node scripts/postinstall)
npm warn allow-scripts
npm warn allow-scripts Run `npm install -g --allow-scripts=@deepseek-ai/dsh-subprocess-local,koffi,node-pty,@google/genai,protobufjs` to allow these scripts once, or `npm config set allow-scripts=@deepseek-ai/dsh-subprocess-local,koffi,node-pty,@google/genai,protobufjs --location=user` to allow them for all global installs.
PS C:\Users\changwei> dsh --version
0.1.0-rc.8
PS C:\Users\changwei> npm install -g --allow-scripts=@deepseek-ai/dsh-subprocess-local,koffi,node-pty,@google/genai,protobufjs
npm error Cannot destructure property 'name' of '.for' as it is undefined.
npm error A complete log of this run can be found in: C:\Users\changwei\AppData\Local\npm-cache\_logs\2026-08-20T05_04_32_067Z-debug-0.log
PS C:\Users\changwei> dsh web
dsh web: http://127.0.0.1:3080
dsh web: opening the default browser; pass --no-open to disable
PS C:\Users\changwei>
PS C:\Users\changwei>
PS C:\Users\changwei>
PS C:\Users\changwei> npm config set allow-scripts=@deepseek-ai/dsh-subprocess-local,koffi,node-pty,@google/genai,protobufjs --location=user
PS C:\Users\changwei> npm install -g @deepseek-ai/dsh@0.1.0-rc.8
npm warn deprecated node-domexception@1.0.0: Use your platform's native DOMException instead

changed 450 packages in 1m

64 packages are looking for funding
  run `npm fund` for details
PS C:\Users\changwei> dsh --version
0.1.0-rc.8
PS C:\Users\changwei> dsh web
dsh web: http://127.0.0.1:3080
dsh web: opening the default browser; pass --no-open to disable

