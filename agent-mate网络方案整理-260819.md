# agent-mate 网络方案讨论整理（2026-08-19）

> 整理来源：WorkBuddy 会话（项目 agent-mate，2026-08-19，会话文件 `b7e69b4e-ebef-498b-8c9c-b7e4d14bda7f.jsonl`）。
> 整理范围：仅抽取「agent-mate 网络方案」主线——拦截改写模型 / 方法分类 / 通道优先级 / 目标实体表 / 两层声明式规则 / 实测验证 / 包管理器生态清单 / 每类 agent 安装脚本适配补丁。
> 已排除（非主线）：CODEBUDDY.md 注入机制与系统级/项目级规则定位、执行纪律与 SOUL.md 行为基调、本机 python 版本盘点与 3.12 来源排查、AI 内部反思与道歉。
> 关联草案：同目录 `网络层适配想法-0819.md`（AI 草案）。本次整理产物独立成文件，未与之合并、未覆盖。

---

## 0. 术语基座（五层术语链 + 通道优先级铁律）

（定稿于 260819 凌晨，MacMini 家用机会话）

- **五层术语链**：`目标实体（平台 × 访问方式）→ 平台 → 访问方式（协议是派生属性）→ 通道 → 端点`
  - 例：GitHub download = 平台 GitHub × 访问方式 download-release。
- **通道优先级（铁律）**：`1 代理 → 2 直达 → 3 镜像站 → 4 加速器`
  - 1/2 是 universal 固定位，**不入表**；路由表只写第三/第四优先级（镜像站、加速器）。
- **绑死规则**：版本探测 + 下载**同平台同通道**；该规则仅 nodejs / python 适用；git 用 latest、包管理器源不探测。
- **目标实体表（腾讯文档现状）**：列结构 = 目标实体 / 访问方式 / 第三优先级（镜像站）/ 第四优先级（加速器）/ 用到它的 agent。已填 3 组：nodejs（安装包，第四判空）、python（安装包 / pip / uv 安装器 / uv 源 4 行）、git（安装包 / git 操作 2 行）。

---

## 1. 拦截改写模型（核心认知）

### 1.1 拦截点：回车之后、建进程之前，全程内存字符串操作

- agent-mate 本身就是终端：用户敲的字符先进入 agent-mate 进程内存；按回车那一刻，完整命令字符串已在内存里组装完毕，**curl/子进程尚未创建、什么都没执行**。
- 检测/改写发生在「回车后、创建子进程前」这一窗口，数据全程在 agent-mate 自己的内存里，原 URL 从没离开过它 → 这是无痕的关键。
- 与代理/证书/MITM **无关**：改写是命令文本层面的字符串操作，不是网络层截流。拦截（代理在途截流）已被 260818 否决，agent-mate 走命令层（包装器 / PTY 模式）先改后放行。

### 1.2 命令行 vs GUI 按钮：两者都是「意图明确」

- 命令行：意图在用户敲的**文本**里（`curl -L https://github.com/...`），agent-mate 必须先把 URL 从文本里**解析**成结构化意图，再查表改写执行。
- GUI 按钮：意图在程序内是**结构化数据**（按钮 = "安装 git"），无需解析，直接查表构造。关键点：这里从头到尾**不存在原 URL**，agent-mate 自己拼 URL 时就已经是镜像地址。
- 统一模型：`意图明确 → 执行前介入 → 查表选源 → 构造/改写命令 → 执行`。命令行比按钮多一道「文本解析成结构化意图」的工序，其余（拦截点、查表、选源、执行）完全一致。

### 1.3 协议模型：HTTP GET 明文，无 MITM、无证书

- 拦截对象是 HTTP GET（明文），请求行里 URL 直接可见；流程 = 捕获（拿到明文 URL）→ 改写（请求行 + Host）→ 执行（由 agent-mate 发出），全流程无加密参与。
- HTTPS 也无碍：URL 在「建立任何连接之前」就躺在内存的命令字符串里，管它 http 还是 https，文本就是文本、明文就是明文。命令层拦截 http/https 无差别；只有「代理层在途拦截」才因 CONNECT 隧道加密而看不到 URL（那才需要 MITM 证书，而 agent-mate 不走那条路）。
- 与代理/证书/MITM **无关**：改写是内存中的字符串操作。

### 1.4 复杂脚本：拦截点从「一条命令」下沉到「一个进程树」

- 脚本里 URL 多是运行时才拼出来的（`curl -fsSL "$URL"`），执行前文本里只有变量名 → 不能靠「执行前读一遍脚本文本」。
- 机制：agent-mate 执行 `bash install.sh` 时，给脚本进程树注入一个前置 PATH（自己的目录里放 curl/wget/git 包装器）。脚本内 `curl ...` 命中的是**包装器**，此刻变量已展开，包装器拿到的是**完整最终 URL**，再改写后 exec 真 curl。
- 包装器比静态分析可靠：动态拼接、嵌套变量全都能覆盖（静态读文件只看到 `"$URL"`）。

### 1.5 无痕保证（铁律）

- 不装证书；代理设置运行期改、退出恢复（或进程级环境变量注入，只影响自己派生的进程树，系统设置零改动）；规则路由全内存；包装器目录与环境变量只存在于本次进程树，进程退出即无，不落盘。

---

## 2. 拦截改写方法分类（A / B 两大类 + 真盲区）

### 2.1 层级树（最终版，两层无冗余）

```
选源机制（按目标类型分流）
├── A. 拦截改写类（裸 URL 下载，无配置接口）
│   └── A2. PATH wrapper（主，脚本内每个子命令 exec 前接管）
│       └── A1. 文本层兜底（可选，只处理用户直接敲的绝对路径命令，如 /usr/bin/curl）
└── B. 注入类（有配置接口的客户端，agent-mate 不碰 URL，注入后客户端自己消费）
    ├── B1. 环境变量注入（源配置：npm_config_registry / PIP_INDEX_URL / UV_INDEX_URL / UV_PYTHON_INSTALL_MIRROR / git insteadOf 等）
    └── B2. 解释器钩子注入（启动钩子：PYTHONPATH 指向 sitecustomize 打补丁 urllib/requests；PowerShell profile 覆盖 cmdlet）
```

- 划分依据（Level 1）：按「有没有检测+改写动作」分 A / B，互斥。
- A1/A2 关系：A2（PATH wrapper）能覆盖 A1 的典型场景（用户敲的命令和脚本命令最终都命中同一个 wrapper），A1 降级为「可选兜底」，只处理绕过 PATH 的绝对路径命令。

### 2.2 A 类：拦截改写（针对命令与脚本）

| 子类 | 机制 | 覆盖场景 | 备注 |
|---|---|---|---|
| **A2 PATH wrapper（主）** | PATH 前插 agent-mate 目录放 curl/wget 包装器；脚本内子命令 exec 前接管，拿到变量展开后的完整 URL | bash 安装脚本内部所有外部 curl/wget/git 下载 | 主方案，覆盖 90% 脚本场景 |
| **A1 文本层兜底（可选）** | 直接敲的绝对路径命令（如 `/usr/bin/curl`）绕过 PATH | 用户显式绝对路径 | 可选，不做也行 |

### 2.3 B 类：注入（针对进程内部请求与配置口）

| 子类 | 机制 | 覆盖场景 |
|---|---|---|
| **B1 环境变量** | 注入 `npm_config_registry` / `PIP_INDEX_URL` / `UV_INDEX_URL` / `UV_PYTHON_INSTALL_MIRROR` / git `insteadOf` 等 | 包管理器、工具链进程内部请求；A2 拦不到的所有软件自有下载逻辑 |
| **B2 解释器钩子** | PYTHONPATH 注入 sitecustomize.py monkey-patch urllib/requests；PowerShell profile 覆盖 cmdlet | python 脚本内 urllib 下载；pwsh 脚本内 `irm`/`Invoke-WebRequest`（不走 PATH，A2 全盲区） |

### 2.4 真盲区（必须明确记录，静态分析不可靠）

- **bash 函数覆盖 curl**：`curl() { ... }` 函数定义在脚本自己的 shell 命名空间，函数优先于 PATH，wrapper 不会被调用；脚本内容 agent-mate 从没读过，A1 文本层也看不到。运行时无解，只能靠 B1 或执行前静态读脚本文本（函数可动态生成、脚本可 source 嵌套，不可靠）。
- **PowerShell cmdlet**：`irm` / `Invoke-WebRequest` / `Invoke-RestMethod` 是内建 cmdlet，不走 PATH → 只能 B2 钩子或改写脚本文本。

---

## 3. 通道优先级与路由表边界

- 铁律：`1 代理 → 2 直达 → 3 镜像站 → 4 加速器`；1/2 固定位不入表，表只写第三/第四优先级。
- 候选约束：改写目标必须自己处理跳转（xget 服务端、ghproxy 都处理 302）。GitHub release 原始响应会 302 跳到 `release-assets.githubusercontent.com` / `objects`，所以第三/第四优先级候选不是随便哪个前缀都行。
- 多域名：GitHub 不是单域名，raw（`raw.githubusercontent.com`）、源码 tarball（`codeload.github.com`）、API（`api.github.com`）、release（`/releases/download/`）是不同前缀，各自匹配各自访问方式。
- 模式匹配 + 按目标重写：xget 是同构替换（`https://{xget}/gh/{owner}/{repo}/releases/download/...`），清华镜像却是不同构（`https://mirrors.tuna.tsinghua.edu.cn/github-release/{owner}/{repo}/{tag}/{file}`，tag 段被提到前面），所以不是简单前缀替换。

---

## 4. 目标实体表设计（五层基座落表）

- 表列结构（腾讯文档 `GUuxjOGvJMtX` / `BB08J2`）：目标实体 / 访问方式 / 第三优先级（镜像站）/ 第四优先级（加速器）/ 用到它的 agent。
- 已完成 3 组：nodejs（安装包，第四判空）、python（安装包 / pip / uv 安装器 / uv 源 4 行）、git（安装包 / git 操作 2 行）。
- **核心空档（两头空，必须 agent-mate 自己解决）**：nodejs.org 安装包下载——chsrc 不管裸 URL，xget 无 node 前缀；python / git / pwsh / agent 本体的 GitHub release 目标，xget 的 `gh` 前缀恰好能填第四优先级（且自带 302 处理）。
- 现有工具对照（chsrc × xget，仅比目标不比机制）：
  - 两边都覆盖：npm / PyPI / conda / Maven / Gradle / Homebrew / RubyGems / CRAN / CPAN / CTAN / Go / crates / Flathub / Debian / Ubuntu / Fedora / Rocky / Arch / openSUSE / Docker Hub。
  - 仅 chsrc：约 30 个其余 Linux 发行版系统源、nvm、rustup、Flutter、Julia、Clojure、Haskell、OCaml、Lua、PHP、Perl、winget、CocoaPods、Nix、Guix、Emacs、TeX Live 等（靠换环境变量/配置，如 `NVM_NODEJS_ORG_MIRROR`、`RUSTUP_DIST_SERVER`）。
  - 仅 xget：GitHub 全场景（clone/release/raw/gist）、GitLab/Gitea/Codeberg/SourceForge/AOSP、HuggingFace/Civitai、AI 推理 API（29 家）、容器镜像 registry（17 个）、NuGet/Packagist/Apache/arXiv/Jenkins/F-Droid。
  - 两边都不覆盖（agent-mate 空白）：nodejs.org 安装包、python.org 安装包、python-build-standalone、Git for Windows 安装包、PowerShell 安装包。
- 结论：现存软件无等价物——DevSidecar/FastGithub 有动态静默切换但换路不换源；chsrc/cmirror 有选源但手动写死配置；三件套本体（nodejs.org/python-build-standalone/git 二进制）没有任何现成工具碰过。agent-mate 4.1 做两者都没覆盖的空档。

---

## 5. 两层声明式规则架构（定稿）

### 5.1 两层设计

- **第一层（global）**：开发三件套（nodejs / python / git）安装包 + 所有包管理源 + uv + GitHub CDN 基础设施。任何 agent 的基础，变更频率低。
- **第二层（agent:&lt;name&gt;）**：单个 agent 安装脚本在第一层覆盖不到处的补丁/加固（astral.sh 改写、UV_PYTHON_INSTALL_MIRROR、pwsh B2 钩子等）。用户可选、新 agent 持续加入，变更频率高。
- 运行时同一引擎处理两级规则，**第二层优先级高于第一层**（per-agent 覆盖全局）。新增 agent 成本 = 分析安装脚本 → 清单划归已覆盖项 → 只写没覆盖的补丁。

### 5.2 统一 schema + scope 字段

- 两层**同一 schema**，靠 `scope` 字段区分：`global` vs `agent:<name>`。
- 规则形态 = **声明式规则数据**（目标实体表：目标 → 改写规则 → 路线表 → 校验），引擎解释执行。
- **禁任意可执行脚本**：公共站被攻破最多改错 URL，不允许 RCE（这是 5.1 设计的关键安全修正——动态注入必须是声明式规则，不是脚本）。
- 适配粒度：`agent名@版本号` → 规则集；agent 升级 → 新适配版本 → 客户端下次启动自动更新。

### 5.3 纯引擎本体 + 出厂兜底规则

```
软件本体 = 纯引擎（解析 / 校验 / 同步 / 改写 / 注入）+ 出厂兜底规则（最小集）
```

- 第一层也声明式化后，**软件本体退化为纯引擎**，两层统一成一套数据模型，所有可变化的东西全部外置成数据。
- 出厂兜底规则 = 三件套默认源改写（npmmirror / PyPI 镜像 / GitHub CDN），保证首次启动、断网、公共站全挂时最基本安装可用。
- 软件本体只在引擎修 bug 时发版；一切规则变化走动态更新。

### 5.4 启动时增量同步机制

```
软件启动
  └─ 拉公共站 manifest.json（版本清单，很小）
       ├─ 对比本地缓存版本
       │    ├─ 无变化 → 直接用本地缓存（离线可用）
       │    └─ 有新版本 → 只拉变更的规则集 → sha256 校验 → 原子替换
       └─ 公共站不可达 → 静默降级
```

| 组件 | 作用 | 要求 |
|---|---|---|
| manifest.json | 版本清单（两层分开版本号：agent 名 + 适配版本 + 校验值） | **增量拉取**，启动只拉 manifest 不拉全量 |
| sha256 校验 | 防公共站篡改 | **必须**，校验不过不加载 |
| 本地缓存 | 公共站挂也能用旧规则 | 启动先本地后远端 |
| 版本化文件名 | 沿用 `260816` 式版本命名 | 备份后覆盖流程（规则 5.5/5.6） |

**降级链路**：公共站 → 最近一次成功缓存 → 出厂兜底规则。

### 5.5 发布策略（机制统一，策略分层）

| 通道 | 适用 | 门槛 | 出错影响 |
|---|---|---|---|
| **stable** | 第一层（global） | 人工审 + 版本门禁 + 可回滚 | 全量安装失败 |
| **fast** | 第二层（agent） | 快速迭代 | 单 agent 适配失败 |

两层机制完全一样，**唯独发布门槛不同**（第一层改坏全局崩，所以 stable；第二层错了只影响一个 agent，所以 fast）。这是 manifest.json 里两层要**分开版本号**的原因。

### 5.6 公共站选址

约束：国内可达 + 静态托管。候选：GitHub Pages（需 CDN 加速）、Gitee Pages、腾讯云 COS 静态站、CloudStudio 静态托管。manifest + 规则集全是静态文件，任何静态托管都能干。

### 5.7 安全边界（硬性）

1. 第二层必须是声明式规则，不是任意脚本——公共站被攻破 ≠ RCE。
2. sha256 校验必须，校验不过不加载。
3. 无痕铁律：不装证书、不写用户 shell 配置（`UV_NO_MODIFY_PATH=1`）、规则路由全内存、代理设置运行期改退出恢复。
4. 真盲区明确记录：bash 函数覆盖 curl、pwsh cmdlet，静态分析不可靠，只能 B1/B2。

---

## 6. 实测验证（hermes 官方安装脚本 + uv + cnrio.cn）

> 样本：NousResearch/hermes-agent `scripts/install.sh`（3541 行 / 152KB）+ `scripts/install.ps1`（4834 行 / 235KB），GitHub main 分支原版。

### 6.1 结论

- 两个脚本均**无 python 直接下载**（Python 一律经 `uv python install`）、**无函数覆盖 curl**。
- bash 版下载 **100% 走外部 curl** → **A2 全覆盖**。
- pwsh 版下载 **100% 走 cmdlet**（irm / Invoke-WebRequest / Invoke-RestMethod）→ **A2 全盲区，必须 B2**。

### 6.2 hermes 脚本内网络请求清单（全部有国内镜像）

| 目标 | bash 方式 | pwsh 方式 | 拦截手段 | 国内镜像 |
|---|---|---|---|---|
| nodejs.org 二进制 | 外部 curl（L920/922/945） | Invoke-WebRequest（L1624/1633） | A2 / B2 | npmmirror.com/mirrors/node（阿里，nodejs 官方认可镜像） |
| github.com git clone（hermes 仓库） | 外部 git（L1370/1375，先 SSH 后 HTTPS） | — | A2 git wrapper 或 B1 insteadOf | GitHub CDN 加速（ghproxy 类） |
| raw.githubusercontent.com（cua-driver） | 外部 curl（L2575） | Invoke-RestMethod（L3635） | A2 / B2 | GitHub CDN raw 加速 |
| uv 安装器 | curl astral.sh/uv/install.sh（L584） | irm astral.sh/uv/install.ps1（L775/783） | A2 / B2 | cnrio.cn（重定向 cnb.cool） |
| uv pip install 本体依赖 | uv pip install -e '.[all]'（L1697） | — | B1 UV_INDEX_URL | PyPI 国内镜像 |

### 6.3 uv 三层（唯一特殊目标）

| 层 | 请求位置 | 拦截手段 | 国内镜像 |
|---|---|---|---|
| ① 安装 uv（下载安装器脚本） | **脚本里**外部 curl/irm | A2 / B2，改写 astral.sh | **有：cnrio.cn/install.sh → cnb.cool** |
| ② uv python install（python 二进制） | **uv 进程内**下载 python-build-standalone | 拦不到，只能 B1 `UV_PYTHON_INSTALL_MIRROR` | **有：cnb.cool**（腾讯云） |
| ③ uv pip / 自更新 | uv 内部访问 PyPI / astral 资源 | 只能 B1 `UV_INDEX_URL` / `UV_ASTRAL_MIRROR_URL` / `UV_DOWNLOAD_URL` | PyPI 清华/阿里/腾讯镜像；astral 资源 cnb.cool |

- 第 1 层（安装 uv）不是问题——发生在脚本里，走 curl/irm，A2/B2 照常拦。
- 第 2、3 层（uv 内部下载）A2 拦不到，但 uv 官方预留环境变量后门，注入即可，不需要拦截。

### 6.4 cnrio.cn/install.sh 判定（uv 0.12.5 定制安装器）

- 与官方 astral-sh/uv 0.12.5 **同模板**（cargo-dist 0.31.0，L69 元数据），仅两处改动：默认下载源 → cnb.cool（L39）+ 尾部 CNB 段落（L2196+）。
- **版本/功能零修改**，下载的 uv 二进制带 sha256 校验（L240-324），校验通过即官方原版，不会被篡改。
- **完整替代**官方 install.sh，对 hermes 安装脚本无负面影响。
- 副作用（CNB 私货）：若 `UV_PYTHON_INSTALL_MIRROR` 未设置且未禁用，自动往 `~/.profile`/`~/.bashrc`/`~/.zshrc` 写 `UV_PYTHON_INSTALL_MIRROR` 和 `UV_ASTRAL_MIRROR_URL`（指向 cnb.cool）。
- **无痕用法**：`UV_NO_MODIFY_PATH=1 UV_DISABLE_UPDATE=1` —— 前者禁掉 CNB 私货写 profile，后者禁 uv 自更新检查。hermes 脚本执行 uv 安装器时只传了 `UV_UNMANAGED_INSTALL`，未传 `UV_NO_MODIFY_PATH`，所以 agent-mate 拦截改写时需额外注入这两个变量。
- uv 安装器自带 5 级下载源优先级（`UV_DOWNLOAD_URL` > `INSTALLER_DOWNLOAD_URL` > `UV_INSTALLER_GHE_BASE_URL` > `UV_INSTALLER_GITHUB_BASE_URL` > 默认 cnb.cool），`UV_DOWNLOAD_URL` 支持空格分隔多源 fallback，对多路线选源直接可用。

---

## 7. nodejs / python 生态：包管理器与版本管理器清单 + 配置口

### 7.1 nodejs 生态

- **真·包管理器（5 个）**：npm（2010，Node 自带）、yarn classic（2016，维护模式）、yarn berry（2020，活跃，PnP 无 node_modules）、pnpm（2017，活跃，内容寻址 store+硬链接）、bun（2022，一站式 runtime+包管理+bundler+test）。
  - 注意：yarn classic 与 yarn berry 同名但配置体系完全不同（.npmrc vs .yarnrc.yml），规则表必须当两个目标。
- **版本管理器（管 node 本体）**：nvm / fnm / volta / n（会下载 node 二进制 = 第一层「nodejs 安装包」目标）。
- **易混淆但非包管理器**：corepack（Node 16.9+ 内置版本启动器，按 package.json 的 `packageManager` 自动下载 yarn/pnpm 二进制——隐藏拦截点，走 npm registry）、ni（命令代理）、npx/dlx/bunx（执行器）、deno（runtime）。
- **第一层 nodejs 配置口（实际只有 4 个）**：

  | 包管理器 | 配置口 | 注入 |
  |---|---|---|
  | npm / yarn classic / pnpm | `.npmrc` 的 `registry=` | **同一口**：B1 `npm_config_registry` 三者通吃 |
  | yarn berry | `.yarnrc.yml` 的 `npmRegistryServer` | B1 `YARN_NPM_REGISTRY_SERVER` |
  | bun | `bunfig.toml` 的 `[install] registry` | B1 `BUN_CONFIG_REGISTRY` |
  | corepack（下载 yarn/pnpm 本体） | 走 npm registry | B1 `COREPACK_NPM_REGISTRY` |

  - 第一层 nodejs 部分 = **4 个环境变量 + nodejs.org 安装包 URL 改写**，规则表很小。corepack 那条最容易漏。

### 7.2 python 生态（结构同构 nodejs，多一层 venv 隔离）

| 角色 | nodejs | python |
|---|---|---|
| 运行时本体 | Node.js | CPython |
| 版本管理器 | nvm / fnm / volta / n | pyenv / pyenv-win / asdf / mise / conda / **uv python** |
| 包管理器 | npm / yarn / pnpm / bun | pip / poetry / pdm / pipx / conda / **uv** |
| 包仓库 | npmjs.org | **PyPI** |
| 环境隔离 | node_modules（无需激活） | **venv（独立环境 + 激活概念）** |
| 执行器 | npx / dlx / bunx | uvx / pipx run |

- **python 版本管理器**：pyenv（python.org/python-build，镜像口 `PYTHON_BUILD_MIRROR_URL`）、asdf/mise、conda（repo.anaconda.com，镜像清华/中科大）、**uv python install**（python-build-standalone，镜像口 `UV_PYTHON_INSTALL_MIRROR`）。
- **python 包管理器**：pip（`PIP_INDEX_URL`）、pipx（走 pip 配置）、poetry（pyproject `[[tool.poetry.source]]`，各项目各配无全局 env）、pdm（`pdm config pypi.url`）、conda（`.condarc` channels）、**uv**（`UV_INDEX_URL` 等一组 env）、pipenv（已弃用）、rye（已并入 uv）。
- **第一层 python 配置口**：python 没有统一配置口，比 nodejs 复杂一倍——

  | 子目标 | 配置口 | 形态 |
  |---|---|---|
  | pip / pipx | `PIP_INDEX_URL` | B1 env |
  | uv 装包 | `UV_INDEX_URL` | B1 env |
  | uv 装 python | `UV_PYTHON_INSTALL_MIRROR` | B1 env |
  | pyenv | `PYTHON_BUILD_MIRROR_URL` | B1 env |
  | poetry | pyproject source 表 | 需文本改写（无全局 env） |
  | conda | `.condarc` | 需文件改写 |
  | python.org 安装包 | URL 改写 | A2/A1 |

  - = **5 个 env + 2 个文件改写 + 1 个 URL 改写**。**uv 进第一层价值巨大**——用 uv 的项目越多，poetry/pdm/conda 文件改写规则越少被触发。

### 7.3 uv 的本质：python 生态的「标准生态整合器」

- uv = pyenv + venv + pip + pipx + poetry/pdm 五个工具的合体，单 Rust 二进制。子命令对应：`uv python install`=pyenv、`uv venv`=venv、`uv pip`/`uv add`/`uv lock`=pip+poetry/pdm、`uv tool install`=pipx、`uv run`=执行器。
- **「全合一」不是 uv 首创，conda 才是鼻祖（2012）**。真正区别在哲学：

  | 维度 | conda（独立生态） | uv（标准生态整合器） |
  |---|---|---|
  | 包来源 | 自有频道 conda-forge | **PyPI**（和 pip 同源，wheel 格式） |
  | 虚拟环境 | 自有环境格式 | **标准 venv** |
  | python 本体 | conda 自编译 CPython | 官方 CPython（python-build-standalone） |
  | 关系 | 可完全不碰 pip/PyPI，自成闭环 | 完全依赖标准生态，只是加速合并 |

  - uv 独特点：①零新格式（venv 标准、包 PyPI wheel、python 官方二进制，发明的是「调度」不是「格式」）②兼容 pip 接口（老项目零迁移）③单二进制极快。
  - 顺带影响第一层：uv 走 PyPI wheel + 标准 venv，规则就是纯 env 注入，不像 conda 要碰 `.condarc` 文件改写。

### 7.4 PyPI 定义

- **PyPI = Python Package Index，python 官方软件包仓库**——pip / uv / poetry 默认去「下载包」的地方。类比：PyPI 之于 python = npmjs.org 之于 nodejs。
- 任何人可发布（有 typosquatting 风险）；国内「镜像」本质是 PyPI 的只读完整复制（清华 tuna / 阿里 / 腾讯 / 中科大），即第一层 `PIP_INDEX_URL` / `UV_INDEX_URL` 指向的东西。
- conda 连包来源都不和 PyPI 共用（走 conda-forge），这是「conda 是独立生态」最直接体现。

---

## 8. 开发三件套网络环境改写设计哲学 + 每类 agent 安装脚本适配补丁

### 8.1 设计哲学：一套引擎 + 两级规则

> **纯引擎本体 + 两层统一 schema 的声明式规则（scope 区分 global / agent）→ 一套同步器（manifest + sha256 + 原子替换 + 三级降级）→ 仅发布策略分层（stable / fast）**

- 第一层（三件套）打底；第二层按 agent 生态补对应配置口。GitHub CDN 改写作为两层共用的基础设施规则（引擎内置，不属于任何一层）。
- ffmpeg/ripgrep 等 agent 公共依赖不在三件套内，归第二层每个 agent 自己的补丁（各 agent 获取方式不同：brew/apt/官方二进制），不强行统一。

### 8.2 agent 按「构建生态」三分（决定补丁形态）

| 类型 | agent 例子 | 本体安装要什么 | python 的角色 |
|---|---|---|---|
| **node 本体型** | Claude Code / Codex / Gemini CLI / Continue | npm 装 node 包 | 无（开发期用户自己装） |
| **python 本体型** | **hermes** / aider | **uv + python 3.11 + pip 装本体依赖** | **本体运行依赖，脚本指定版本** |
| **单二进制型** | opencode | curl 下载二进制 | 无 |

- 归因链：agent 用什么语言写 → 决定安装脚本用什么工具链 → 决定第一层哪些配置口被命中。
- 关键区分（hermes 特殊性）：hermes 的 python 是**本体安装期锁版本的硬依赖**（L584 装 uv、L642 `uv python find`、L650 `uv python install 3.11`、L1697 `uv pip install -e '.[all]'`），而其他主流 agent 的 python 是「安装后用户自选的开发工具」。hermes 也下载 nodejs（L920-945），但 nodejs 在 hermes 里是辅助角色，python 才是主体。

### 8.3 每类 agent 的补丁形态（第二层规则表分组依据）

- **node 本体型**：补丁 = `npm_config_registry` + 下载 URL 改写；python 相关规则永不触发。
- **python 本体型（hermes 类）**：补丁 = uv 装 python（`UV_PYTHON_INSTALL_MIRROR`，版本已知 = 3.11，目标路径可精确预测）+ astral.sh 改写 + `UV_INDEX_URL` 装本体依赖——**三连补丁，缺一不可**。因 hermes 的 python 版本脚本硬编码，第二层规则可按版本号精确命中：`python@3.11 → python-build-standalone 对应版本 → cnb.cool 镜像`，不用探测。
- **单二进制型（opencode）**：补丁 = GitHub CDN 下载 URL 改写（只有下载 URL，无包管理器）。
- hermes 实测印证：五成请求被第一层白嫖（nodejs / git clone / uv pip→PyPI），第二层只补三处（astral.sh 改写、UV_PYTHON_INSTALL_MIRROR、raw.githubusercontent cua-driver CDN 改写、pwsh B2 钩子）。

### 8.4 开发三件套网络环境改写的设计哲学小结

- 开发三件套（nodejs / python / git）是任何 agent 的基础，不管用什么 agent 都要装 → 第一层全局规则。
- 第二层（per-agent）不是基础，用户可选、新 agent 持续加入 → 动态订阅、可频繁增删、快速迭代。
- 每个外部 agent 安装脚本做适配补丁：hermes 是 python 本体型、Claude Code/Codex 是 node 本体型、opencode 是单二进制型——**分类决定补丁形态**。

---

## 附：关联文件

- AIWIKI：`04sessions/260819-agent-mate网络源目标实体表.md`（五层术语基座、目标实体表设计）
- 实测样本：`agent-mate/install.sh`（cnrio.cn uv 0.12.5 定制安装器，72729 字节）、`agent-mate/hermes-install.sh` / `hermes-install.ps1`（hermes 官方原版，GitHub main）
- 同目录草案：`网络层适配想法-0819.md`（AI 草案，独立保留，本次未合并）
