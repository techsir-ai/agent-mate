---
agent: WorkBuddy
model: glm-5.2-a
session_id: 824b2f80-1c31-457d-9952-461b7fb07c35
mode: record
host_os: macOS
hostname: MacMini.lan
ip: 192.168.10.100
source: ~/.workbuddy/projects/Users-changwei-Project-agent-mate/824b2f80-1c31-457d-9952-461b7fb07c35.jsonl
summary: agent-mate 项目 4.1《网络环境》独立成文的前期工作会话（260818 晚～260819 凌晨，MacMini 家用机）。主线：从术语辨析出发（端点→源→目标→最终采纳 xget 的 platform 叫法），定稿五层术语基座「目标实体 Target Entity / 平台 Platform / 访问方式 Access Mode / 通道 Channel / 端点 Endpoint」，确立通道优先级铁律（1代理→2直达→3镜像站→4加速器，1/2 不入表）与版本探测+下载绑死规则（仅 nodejs/python 适用），随后在腾讯文档在线表格《网络源目标实体表》（https://docs.qq.com/sheet/DR1V1eGpPR3ZKTXRY，file_id GUuxjOGvJMtX，sheet_id BB08J2）上人机协同逐行填写，完成 nodejs 生态（nodejs 安装包、npm 源）、python 生态（python 安装包、pip 源、uv 安装器、uv 源）、git 生态（git 安装包、git 操作）三组行。关键核验事实：nodejs/node GitHub release 无预编译 assets；xget platform-catalog 无 node 平台、无 uv 平台（uv 走 /pypi/）；npmmirror 有 node+python 二进制镜像但无 uv（404）；python 第四优先级走 xget /gh/ 加速 astral-sh/python-build-standalone；npm 随 nodejs 捆绑安装无独立 URL。技术坑：腾讯文档写多行必须 \n、zsh 吃转义须用 python heredoc + json.dumps、清空单元格用 delete_range。本文件即终点产物，供办公室电脑次日继续填表并最终汇编成 4.1 独立《网络环境管理》文档（双模式：手动固化 + 自动智能选源）。
prev:
created: 260819
---

# agent-mate网络源目标实体表

> 会话标题（AI 生成）：编写网络功能独立需求文档
> 项目：/Users/changwei/Project/agent-mate（需求文档 agent-mate需求文档.md 的 4.1 网络环境部分要独立成完整文档）
> 承接：办公室电脑讨论 `260818-xget原理与agent-mate网络源管理设计.md`（同目录，Obsidian Sync 同步）
> 产出物：腾讯文档在线表格《网络源目标实体表》——见步骤 10

## 环境与前提

- 机器：MacMini.lan（macOS 26.5.2），家用电脑；办公室电脑为另一台，通过 Obsidian Sync 同步 AIWIKI vault
- 载体决策：两台电脑的 WorkBuddy 不能互联会话（步骤 19 已查官方文档确认），跨设备协同靠**共享外部载体**（腾讯文档在线表格 / 同步盘），这正是目标实体表放腾讯文档的原因
- 输入材料：
  - `agent-mate需求文档.md` 4.1 网络环境（原文只写了手动模式：订阅式多源 + 测速 + 固化 + 注入 pty）
  - `260818-xget原理与agent-mate网络源管理设计.md`（办公室讨论：xget 原理、智能选源双模式、注入机制对照表、端点四态表雏形、xget 只是兜底加速器不是主力）
- 三个 agent 的安装脚本（会话中实际抓取核验）：
  - hermes：`curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash` / `iex (irm https://hermes-agent.nousresearch.com/install.ps1)`
  - openclaw：`curl -fsSL https://openclaw.ai/install.sh | bash` / `iwr -useb https://openclaw.ai/install.ps1 | iex`
  - dsh：需求文档 4.4（非脚本），`npm i -g @deepseek-ai/dsh`，前置 pwsh + nodejs
- 交互规则（老板本会话多次强调）：**行要一个一个加**；老板手改表格后 AI 必须**读回对齐**再动手；AI 回答要**短**；写操作须老板显式指令

## 步骤序列

### 阶段一：术语基座定稿（被推翻的中间结论全部保留）

**1. 读需求文档 4.1，识别缺口。**
4.1 现有：订阅式多源、延迟+带宽测试、固化保存、注入 pty 会话、代理设置、GitHub 分流、Node.js 安装 url + npm registry 纳入订阅管理。缺口：源的数据结构、测速方法学、固化机制、注入机制、代理 vs CDN 组合、失败降级、订阅更新信任模型、UI、跨模块依赖（4.2/4.3/4.4 如何消费）。

**2. 读办公室讨论文档，对齐双模式定位。**
4.1 = "国内网络部署向导"底座；开发三件套（nodejs/python/git）本体无"源"概念只能走 URL；包管理器的源是部署之后才有；xget 只是兜底加速器。独立文档应按"手动固化 + 自动智能"双模式结构写。

**3. 术语辨析（多次被否，演化过程）：**

| 轮次 | 提议 | 老板裁决 | 理由 |
|---|---|---|---|
| 1 | "端点"是否准确（GitHub url、npm 源各算一个端点？） | 不够准，需分层 | GitHub 本身不是单一端点（github.com / raw.githubusercontent.com / codeload / api.github.com 至少 4-5 个地址）；把"逻辑目标"和"具体网络地址"揉在一起了 |
| 2 | 三层：源 Source / 通道 Channel / 端点 Endpoint | **"源"被否** | ① GitHub 只是 url 不是源；② "源"是 npm/pip 的专有名词（registry/index），混叫会把 GitHub 下载和 npm registry 拉平 |
| 3 | 顶层叫"目标 Target"+ 目标类型 5 类（github-http/git/npm/pip/uv） | **"目标"被否；uv 并入 pip 被否** | "目标"太泛；uv 源和 pip 源不同、安装包来源不同，合并后探测和改写逻辑无法区分 |
| 4 | 查 xget 源码定名 → **平台 Platform** | **采纳** | xget 路由核心 `resolve-target.js` 用 `SORTED_PLATFORMS.find(...)`，README 专章 "Supported Platforms"，URL 格式 `https://实例/<platform前缀>/...`（/gh/ /npm/ /pypi/ /cr/ /ip/）。与加速器 xget 术语对齐，可直接复用其 platform 目录 |
| 5 | 平台-协议-通道-端点四层 | **"协议"被否为独立层** | xget 自己的 `src/protocols/` 只有 4 个处理器（docker/git/huggingface/ai），npm/pip/uv/github-http 协议恒为 HTTP，是空层；通道改变传输路径不改协议（git 协议无论直连/镜像/代理始终是 git） |

**最终定稿五层术语基座：**

| 层 | 名称 | 含义 | 例子 |
|---|---|---|---|
| 顶层 | **目标实体 Target Entity** | 平台 × 访问方式的有效组合，要解决的那个单元；平台与访问方式之间有对应关系（非任意配对） | "nodejs 安装包"、"npm 源" |
| 2 | **平台 Platform** | xget 同名概念，一个需打通的网络服务 | GitHub、npm registry、PyPI、nodejs.org、astral.sh |
| 3 | **访问方式 Access Mode** | 平台内的取数动作；**协议是 (平台,访问方式) 的派生属性，不是独立层** | download-release / query-api / raw / clone / install-pkg |
| 4 | **通道 Channel** | 加速选择 | 代理 / 直达 / 镜像 / xget |
| 5 | **端点 Endpoint** | 通道对应的具体探测地址 | `raw.githubusercontent.com` |

（平台,访问方式）有效性对应关系：GitHub/GitLab/Gitea→download-release/query-api/raw/clone/push；npm registry/PyPI/uv index→install-pkg；nodejs.org/python.org/git-scm→download-release(+版本探测)。目标实体是平台无关的问题（"装 uv"），解析时才定平台——uv 在 astral-sh/uv 的 GitHub release 上；万一某工具不在 GitHub 就解析到它真正发布的平台。

**4. 抓 4 个安装脚本，按证据列目标实体清单。**
最大发现：**绝大多数安装包在自有域名不在 GitHub**——nodejs 在 nodejs.org、uv 在 astral.sh、hermes/openclaw 脚本在各自官网域名；只有 agent 源码仓库和少数工具（git-for-windows、gum）在 GitHub。这证明平台抽象必需。清单要点：
- hermes：uv 安装器、python 3.11（uv 托管，python-build-standalone）、hermes-agent 仓库（git clone）、nodejs、git（硬编码版本号避开 api.github.com 60次/小时限速）、npm 源、PyPI 源（uv pip）、ripgrep/ffmpeg（系统包管理器，可选）
- openclaw：nodejs（+NodeSource deb/rpm）、git（用 `api.github.com/repos/git-for-windows/git/releases/latest` 做 MinGit 版本探测）、openclaw 包（npm 默认 / git 方式）、pnpm（仅 git 方式）、gum（charmbracelet/gum release，临时 spinner）、Homebrew 安装脚本（GitHub raw）、ripgrep/cmake/python3（可选）
- dsh：dsh 包（npm）、nodejs（前置）、pwsh（前置，dsh 必需，GitHub PowerShell-7 release）、git（可选）

**5. 老板裁决：砍掉 agent 官网脚本行。**
hermes-agent.nousresearch.com / openclaw.ai 国内**不被墙**，只拉一个脚本，比所有真实目标都顺畅，**不是要解决的问题**（H1、O1 删除）。Homebrew 安装脚本虽也是拉脚本，但在 GitHub raw，国内常慢/不稳，保留待定。

**6. 老板定通道优先级铁律（所有目标实体通用）：**

| 槽位 | 通道 | 说明 |
|---|---|---|
| 1 | 代理 | 若用户有代理，永远第一 |
| 2 | 直达 | 官方 URL 直连 |
| 3 | 镜像站 | 表里"第三优先级"列 |
| 4 | 加速器（xget） | 表里"第四优先级"列 |

1、2 是 universal 固定位**不写入表**，表里只写第三/第四优先级两列。排序按**可信度/权威性**不按速度：直达（registry.npmjs.org 实测 ~200ms）排镜像（npmmirror ~10-30ms）之前，因为官方是 source of truth，镜像是副本（有同步延迟+第三方信任问题）。

**7. 绑死规则（版本探测 + 下载同平台同通道，整实体原子切换）：**
- **适用范围仅 nodejs 和 python**（老板原话：各 agent 往往有版本要求甚至只能用某一个版本，LTS 也常更新，所以要主动探测版本→选满足要求的→同通道下）
- **git 不用**：latest 即可，版本不影响使用
- **包管理器源不用**：源本身无版本，版本是流经的各个包的、由客户端 install 时透明解析；agent-mate 只把 channel 指到正确 registry，不主动探测
- 判据：候选通道必须**同时**提供版本探测和安装包下载且同平台——只满足其一（如 GitHub 对 nodejs 只有探测 API + 源码 tarball）则该候选不成立，直接删掉不占位

### 阶段二：协同表格载体选型与建表

**8. 载体选型（AI 走了弯路被骂，正确路径）：**
HTML 表格（✗ 人没法编辑）→ 问 CSV 行不行 → WorkBuddy 无内置 CSV 编辑器，CSV 只能外部软件改 → **正解：腾讯文档在线表格**（连接器 tencent-docs，个人版即可；当时状态 disconnected，老板手动连上）。

**9. 建表（腾讯文档 skill CLI 路径）：**
```
cd /Users/changwei/.workbuddy/plugins/cache/workbuddy-builtin/tencent-docs-plugin/1.0.0/skills/tencent-docs \
  && python3 tencentdocs.py tdoc_init                      # 票据 READY
  && python3 tencentdocs.py tdoc_call tencent-docs manage.create_file '{"title":"网络源目标实体表","file_type":"sheet"}'
```
- 链接：https://docs.qq.com/sheet/DR1V1eGpPR3ZKTXRY
- file_id：`GUuxjOGvJMtX`，sheet_id：`BB08J2`（由 `get_sheet_info` 取得）
- 写入用 `tdoc_call sheet-mcp set_range_value_by_csv '{"file_id","sheet_id","start_row","start_col","csv_data"}'`，读回用 `get_cell_data`（range 到 end_row/end_col）
- 说明：这条 CLI 是腾讯文档 skill 官方指定入口，内部转发到腾讯自家 sheet-mcp；本会话连接器是中途连上的，`mcp__tencent-docs__*` 直连工具未注入当前会话，**下个新会话若有 MCP 工具则直接用，不再绕 CLI**

### 阶段三：nodejs 生态（row1-3）

**10. 表头 + nodejs 行初版写入，老板手改表头。**
老板改后列结构定稿：**目标实体 / 访问方式 / 第三优先级（镜像站）/ 第四优先级（加速器）/ 用到它的 agent**（删掉了 AI 原写的"平台""真实地址"两列，"通道优先级"拆成第三/第四两列）。

**11. nodejs 关键核验（真实 URL，全部实测）：**
- 直达·版本探测：`https://nodejs.org/dist/index.json`（字段 version/date/files/npm/lts）
- 直达·下载根：`https://nodejs.org/dist/vX.Y.Z/`（`node-vX.Y.Z-{platform}.{ext}`，如 linux-x64.tar.xz、win-x64-msi）
- **nodejs/node GitHub release 无预编译 assets**：实测 `api.github.com/repos/nodejs/node/releases/latest`（v26.7.0）`assets: []`，只有 tarball/zipball 源码（`https://api.github.com/repos/nodejs/node/tarball/vX.Y.Z`）；网页端 releases 也只有 2 项源码
- **xget platform-catalog 无 node 平台**（权威列表覆盖代码站/包管理器/Linux 发行版/AI/容器，唯独没有 node/nodejs/nodejs-dist）→ xget 也加速不了 nodejs 安装包
- 镜像：`https://registry.npmmirror.com/-/binary/node/{version}/node-{version}-{platform}.{ext}`，探测 `https://registry.npmmirror.com/-/binary/node/index.json`
- npmmirror 下载 302 重定向到 `cdn.npmmirror.com`（国内 CDN 边缘节点，所以 ping 好——"国内有节点"正是第三档价值）

**12. nodejs 行定稿（绑死规则首次应用）：**
GitHub 只有版本探测没有安装包，不满足绑死 → **删掉 GitHub，第四优先级判空**。nodejs 只有 1代理/2直达/3镜像 三档有效通道。

**13. 变量约定（老板批评 `vX.Y.Z` 与 `{platform}.{ext}` 两套混用后确立）：**
表内 URL 统一花括号 token：`{version}`（取自 index.json 的 version 字段，nodejs 自带 v 前缀）、`{platform}`（本机 OS/arch 映射，如 linux-x64/darwin-arm64）、`{ext}`（由 platform+包格式推出）、`{pkg}`（包名）、`{xget}`（xget 实例域名）。nodejs 镜像模板定稿：`https://registry.npmmirror.com/-/binary/node/{version}/node-{version}-{platform}.{ext}`。

**14. npm 源行（row2→后成 row3）及概念辨析（老板逐问，均定论）：**
- **npm 安装 vs npm 源是两个概念**：npm 安装是把 npm CLI 弄到机器上；npm 源是 `npm install <pkg>` 读的 registry
- **npm 随 nodejs 捆绑**：官方 nodejs 安装包（pkg/msi/tar.xz）默认含 npm，一次下载同时交付 node+npm，**npm 安装无独立 URL**，与 nodejs 安装包同 URL、绑死；这与 agent 无关，是 nodejs 分发的客观事实（边界：apt 等 OS 包管理器会拆包，但那不在本目标实体范围）
- **npm/pnpm/yarn 是三个独立包管理器**：不同命令/锁文件/安装算法，**只共用同一个默认源** `registry.npmjs.org`（"源共用、程序独立"）；默认**仅 npm 随 nodejs 到场**，pnpm/yarn 按需另装；openclaw 的 git 安装方式才需要 pnpm
- registry 是半专有术语：各生态都有（pip→PyPI、cargo→crates.io、Java→Maven Central），在我们表里是访问方式的一个类别（registry = 从包索引服务器装包，与 release 下二进制并列）
- install-pkg 过程：查 registry 元数据定版本→拉 tarball `<pkg>/-/<pkg>-<version>.tgz`→解包 node_modules→递归依赖→生命周期脚本→写锁文件；版本解析由客户端 install 时自动完成（指定版本号或 latest），agent-mate 不主动探测
- **AI 认错记录**："npm 源绑死成立所以有 xget"是错误逻辑——正确原因是 xget 有 `/npm/` 平台能代理 registry，与绑死无关（绑死根本不适用于包管理器源）
- npm 源行内容：第三优先级 = 探测 `https://registry.npmmirror.com/{pkg}` ｜ 下载 `https://registry.npmmirror.com/{pkg}/-/{pkg}-{version}.tgz`；第四优先级 = xget `/npm/`（`https://{xget}/npm/{pkg}`）
- xget 包管理器类平台还有：pypi、conda、maven、cran、crates、dotnet、julia、flutter、packagist（之前只写 npm 是因为只填到那一行）

### 阶段四：python 生态（row4-8）

**15. python 侧核验（真实 URL，curl 实测）：**
- npmmirror **有** python 二进制镜像（HTTP 200，`https://registry.npmmirror.com/-/binary/python/`，版本探测靠解析目录数组的 name 字段，无 index.json）；阿里云 `https://mirrors.aliyun.com/python-release/`、华为云 `https://mirrors.huaweicloud.com/python/` 均 200
- **npmmirror 无 uv**（`/-/binary/uv/` 404）→ uv 安装器第三优先级空缺
- python-build-standalone（astral-sh）确认有预编译 CPython 二进制，且是 hermes 实际经 uv 使用的 python 来源
- **xget catalog 有 pypi/pypi-files，但没有 python 官方安装包平台** → python 安装包第四优先级走 **xget /gh/ 加速 astral-sh/python-build-standalone**（预编译、满足绑死），不能走 /pypi/
- **uv 无专门 xget 平台**：uv 索引源底层就是 PyPI，走 xget 复用 `/pypi/`

**16. python 生态 4 行写入（含写入翻车与修复，详见避坑 P2-P5）：**
- python 安装包（运行时本体）：download-release + 版本探测（绑死适用）；第三优先级 = npmmirror `/-/binary/python/`（解析目录数组 name）+ 阿里 python-release；第四优先级 = xget /gh/ 加速 `https://github.com/astral-sh/python-build-standalone`
- pip 源（PyPI）：install-pkg，版本客户端 install 时解析；第三优先级 = 清华 `https://pypi.tuna.tsinghua.edu.cn/simple`｜阿里 `https://mirrors.aliyun.com/pypi/simple`｜腾讯 `https://mirrors.cloud.tencent.com/pypi/simple`；第四优先级 = xget `/pypi/`（`https://{xget}/pypi/{pkg}`）
- uv 安装器（独立二进制）：download-release（预编译非源码）；第三优先级 = **无**（npmmirror 无 uv 镜像）；第四优先级 = xget /gh/ 加速 `https://github.com/astral-sh/uv/releases/download/{version}/uv-{platform}.tar.gz`
- uv 源：install-pkg（uv registry）；第三优先级 = 复用 PyPI 镜像；第四优先级 = xget /pypi/（底层 PyPI）。**uv 单列不并入 pip**（老板早前裁决），但底层同源 PyPI，仅注入变量不同：`UV_INDEX_URL` vs `PIP_INDEX_URL`
- hermes 依赖链结论：python3.11 + uv，uv 源底层 PyPI，注入 UV_INDEX_URL 与 PIP_INDEX_URL 区分

### 阶段五：git 生态（row9-11）与收尾

**17. 老板重整表格式后填 git 生态。**
老板亲自整理了格式（分组行只留生态名、访问方式极简），AI 读回对齐后按新格式写。核验：清华 `https://mirrors.tuna.tsinghua.edu.cn/github-release/git-for-windows/git/`、华为云 `https://mirrors.huaweicloud.com/git-for-windows/` 均 curl 200；gitclone.com 是 clone 前缀非网页，根路径 404 属正常。两行：
- git 安装包：download-release（**latest 无需版本探测**，绑死不适用）；镜像 = 清华 github-release/git-for-windows + 华为云；加速器 = xget /gh/ 下 `https://github.com/git-for-windows/git/releases/download/{version}/Git-{version}-64-bit.exe`（预编译 Windows）
- git 操作（clone/fetch/push）：git 协议，注入 insteadOf/GIT_CONFIG（进程级零痕）；镜像 = gitclone.com（只读 clone）`https://gitclone.com/github.com/{owner}/{repo}.git` + ghproxy/ghfast 前缀（https clone）；加速器 = xget /gh/（支持 git 协议）`https://{xget}/gh/{owner}/{repo}.git`；**push 只能走代理**（镜像均只读）

**18. 跨设备 WorkBuddy 问题（查官方文档确认）：**
两台电脑的 WorkBuddy 互相独立，**不能**连接/进入另一台机器 WorkBuddy 的指定对话。唯一跨设备动作是"分享任务"（公开链接，只读）。有效跨设备协同 = 共享外部载体（腾讯文档/Obsidian Sync/云记忆只读）。因此本会话按老板指令用整理子 agent 按记录模式落盘本文件，供办公室电脑次日继续。

## 当前表格最终状态（跨设备继续工作的载体）

腾讯文档在线表格《网络源目标实体表》 https://docs.qq.com/sheet/DR1V1eGpPR3ZKTXRY （file_id `GUuxjOGvJMtX`，sheet_id `BB08J2`）

列结构（老板手改定稿）：**目标实体 / 访问方式 / 第三优先级（镜像站）/ 第四优先级（加速器）/ 用到它的 agent**

已填（按行序）：
- 表头（row0）
- **nodejs 生态** 分组：nodejs 安装包（第三=npmmirror node 二进制；第四=无，绑死判空）、npm 源（第三=npmmirror registry；第四=xget /npm/）
- **python 生态** 分组：python 安装包（第三=npmmirror python+阿里；第四=xget /gh/ python-build-standalone）、pip 源（第三=清华/阿里/腾讯 pypi simple；第四=xget /pypi/）、uv 安装器（第三=无，npmmirror 404；第四=xget /gh/ astral-sh/uv）、uv 源（第三=复用 PyPI 镜像；第四=xget /pypi/）
- **git 生态** 分组：git 安装包（第三=清华/华为云 git-for-windows；第四=xget /gh/）、git 操作（第三=gitclone.com/ghproxy；第四=xget /gh/ git 协议；push 只能代理）

项目日志已记入 `/Users/changwei/Project/agent-mate/.workbuddy/memory/2026-08-19.md`。

## 复现检查

- 打开腾讯文档链接或以 file_id `GUuxjOGvJMtX` + sheet_id `BB08J2` 用 `get_cell_data`（range row0-11, col0-4）读回，应与上文"当前表格最终状态"一致
- 核验命令样例（HTTP 状态码）：
  ```
  curl -s -o /dev/null -w "%{http_code}" https://registry.npmmirror.com/-/binary/python/   # 200
  curl -s -o /dev/null -w "%{http_code}" https://registry.npmmirror.com/-/binary/uv/       # 404
  curl -s https://api.github.com/repos/nodejs/node/releases/latest | jq '.assets'          # []
  ```
- 写表复现：python heredoc + json.dumps 构造 csv_data 后调 `tdoc_call sheet-mcp set_range_value_by_csv`（见避坑 P2/P3）

## 避坑与易错点

- **P1（步骤 8）协同载体**：HTML 表格不是人可编辑的协同载体；WorkBuddy 无内置 CSV 编辑器。人机协同编辑的正解是腾讯文档在线表格。
- **P2（步骤 16）zsh 吃转义**：向腾讯文档写多行，csv_data 的行分隔必须用 `\n`；zsh 命令行会把 `\n` 转义吃掉导致 JSON 断行报错。**固定解法：python heredoc，内部用 `json.dumps(payload, ensure_ascii=False)` 构造参数再 subprocess 调 tencentdocs.py**。
- **P3（步骤 16）`<br>` 不分行**：误用 `<br>` 作行分隔，腾讯文档不认，5 行数据全塞进单行多列（连错两次），脏数据挤到 F-K 列，需读回确认污染范围后重写+清空。
- **P4（步骤 16）空字符串不覆盖**：`set_range_value_by_csv` 写空字符串不会清掉旧值，清残留必须用清空方法。
- **P5（步骤 16）清空方法名**：`clear_range_value` 不存在（该 MCP 未暴露）；枚举候选后确认 **`delete_range` 可用**（参数同 get_cell_data 式 range），用它清 row4 col5-10 成功。
- **P6（步骤 15）WebFetch 误报**：WebFetch 对 npmmirror python 二进制路径报 NOT_FOUND，curl 实测 HTTP 200——镜像可达性核验以 curl 为准。
- **P7（步骤 17）gitclone.com 根路径 404 属正常**：它是 clone URL 前缀（`https://gitclone.com/github.com/{owner}/{repo}.git`），不是网页，别当不可达判死。
- **概念误区（阶段一/三，已纠入主线）**：① "源"与 npm/pip 专有名词冲突；② "目标"太泛；③ uv 不能并入 pip 类；④ "协议"不是所有平台都有的独立层（是访问方式的派生属性）；⑤ 绑死规则超范围套用到包管理器源是错的（仅 nodejs/python）；⑥ "npm 源绑死成立所以有 xget"是错误逻辑；⑦ AI 空口排"GitHub 第 3"被骂——候选通道必须先把真实地址查出来再进表。
- **交互规则**：行一个一个加；老板手改表格后必须读回对齐；回答要短；写操作须老板显式指令。

## 待办 / 下一步（办公室电脑继续）

1. 填 **pwsh 安装包** 行（dsh 必需，GitHub PowerShell-7 release，需核 xget /gh/ 下 exe/msi）
2. **各 agent 本体**：hermes-agent 仓库（git clone）、openclaw 包（npm）/ openclaw 仓库（git 方式）、dsh 包（npm）
3. **小依赖**：gum（GitHub release）、Homebrew 安装脚本（GitHub raw）
4. **系统包管理器依赖**（ripgrep/ffmpeg/cmake/python3）：标"委托系统"，v1 不纳核心
5. **pi agent 安装脚本还没读**——需抓脚本确认其目标实体（不能猜）
6. 回读三个 agent 安装脚本，提取 **node 最低版本约束**精确填进 nodejs 行版本探测逻辑（此前未逐字记录）
7. 最终把表汇编成 4.1 的独立《网络环境管理》文档，双模式结构：**手动固化 + 自动智能选源**（智能选源技术内核见 260818 办公室讨论：进程级临时注入对照、git insteadOf 只认 config 的坑、惰性探测、滑动窗口快照、捕获边界三方案）

## 相关条目

- `260818-xget原理与agent-mate网络源管理设计.md`（办公室电脑前置讨论，本会话的直接输入）
- 腾讯文档《网络源目标实体表》（本会话产出物，见上）
- `/Users/changwei/Project/agent-mate/agent-mate需求文档.md` 4.1（待独立成文的对象）
- `/Users/changwei/Project/agent-mate/.workbuddy/memory/2026-08-19.md`（项目日志）
