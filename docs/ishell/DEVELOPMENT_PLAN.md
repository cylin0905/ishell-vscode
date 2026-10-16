# iShell —— 基于 VSCode 二次开发的 Linux 服务器连接工具 · 开发计划（需求基线）

> 本文档为前期三部分规划（分阶段实施、产品原型、AI 集成）的沉淀，是开发执行的唯一需求基线。产品名 iShell 为暂定名。

## 0. 立项定位与前提

- **产品定位**：Windows 桌面 Linux 服务器连接/管理工具（对标 XShell / WindTerm / Termius），底座复用 VSCode 开源源码（MIT 许可）。
- **"原生 Windows"口径**：Electron 桌面应用（x64/arm64 安装包、代码签名、系统通知/任务栏/凭据管理器/跳转列表集成），非 Win32/UWP 重写。
- **合规红线**：
  1. "Visual Studio Code" 名称与 Logo 为微软商标，必须彻底重塑品牌（参考 VSCodium 清理方案）。
  2. 官方 Remote-SSH 扩展为闭源专有软件，不可复用其代码；SSH 能力基于开源 ssh2 库自研（open-remote-ssh 等仅作架构参考，引入前过 license 审查）。
  3. 微软 Marketplace 条款限制非官方 VSCode 使用 → 扩展市场切换 Open VSX 或禁用。
  4. 移除微软遥测 key；更新 URL 指向自建静态 JSON 源或禁用。
- **复用的核心资产**：workbench 框架（布局/主题/快捷键/国际化/设置系统/命令面板）、xterm.js 终端与 addon 体系、Monaco 编辑器、Electron 主进程基建（更新/崩溃/多窗口/协议注册）。

## 1. 总体架构

- **架构决策：轻量 SSH 直连（方案 B）**——终端 = SSH shell 通道 → xterm.js；文件 = SFTP；隧道 = 端口转发；**不在远端部署 vscode-server**。v2 可选"深度模式"（方案 C）。
- **分层**：
  - 渲染层：VSCode workbench + 自研 contrib 模块
  - 服务层：Electron 主进程 + Utility Process（SSH 连接池、SFTP 会话、隧道管理、DPAPI 凭据加密、配置持久化）——渲染层崩溃不影响连接
  - IPC：复用 VSCode 既有主↔渲染基建
- **二次开发隔离原则（最重要工程纪律）**：
  - 自研代码全部收敛到 `src/vs/workbench/contrib/ishell/` 与 `extensions/ishell-*`；
  - 对 VSCode 既有文件的修改控制在最小补丁集（product.json、少量挂载点），便于 cherry-pick 上游安全修复；
  - UI 遵循 VSCode 原生 DOM/Widget 风格与主题 css 变量，**不引入 React**。

## 2. 分阶段计划（M0–M6 + AI 轨）

| 阶段 | 主题 | 里程碑 |
|---|---|---|
| 0 | 预研与架构决策 | M0 可行性放行 |
| 1 | 工程基座与品牌外壳 | M1 可安装的干净外壳 |
| 2 | 连接管理与 SSH 核心 | M2 双击即连、断线自愈 |
| 3 | 远程终端 | M3 日常可用主力终端 |
| 4 | 文件传输（SFTP） | M4 传输与远程编辑闭环 |
| 5 | 配置管理与运维增强 | M5 功能完整 Beta |
| 5A | AI 功能落地 | MA2 |
| AI-T | AI 基础框架（与阶段 3/4 并行） | MA1 |
| 6 | 打磨与发布 | M6 / GA v1.0 |

### 阶段 0：预研与架构决策
- 基线锁定：最新稳定 tag 冻结 commit；上游同步策略（仅 cherry-pick 安全修复）。
- Windows 构建链：Node（版本以仓库要求为准）/ VS Build Tools 2022（C++ workload）/ Python；原生模块按 Electron ABI 编译。
- PoC①：主进程 ssh2 完成密钥认证 + 交互 shell + SFTP，桥接渲染进程 xterm.js。
- PoC②：裁剪评估 ADR（隐藏优先于删除）。
- 验收：干净环境复现构建；PoC 中文/ vim/top/resize 正常；ADR 评审通过。

### 阶段 1：工程基座与品牌外壳
- product.json 品牌重塑（nameShort/nameLong/applicationName 等 → 用户数据目录与注册表自动隔离）；图标/EXE 版本信息/关于页。
- 合规清理：遥测 key、更新源（自建静态 JSON 或禁用）、市场→Open VSX 或禁用、默认设置裁剪。
- CI/CD；Inno Setup 安装器（per-user、升级保数据、卸载可选保留、注册 myproto:// 协议）；中文一等语言（VSCode l10n）。
- 验收：签名包安装-启动-升级-卸载全流程；抓包无 MS 域出站；与官方 VSCode 数据完全隔离；CI 产物含哈希与 SBOM。

### 阶段 2：连接管理与 SSH 核心
- 主机档案 CRUD/分组/标签/搜索；认证矩阵（密码/密钥 ed25519+passphrase/ssh-agent 管道+pageant/keyboard-interactive）；known_hosts 指纹确认+变更硬阻断；连接状态机+keepalive+指数退避重连+**单连接多通道复用**；快速连接 QuickInput（user@host:port 不落盘）。
- 选型：Electron safeStorage（DPAPI）存凭据；JSON+敏感字段加密存档案。
- 模块：connection-core（主进程连接池）/ connection-store / connection-ui（树+对话框）/ auth / host-key。
- 验收：50 并发连接 4h 稳定；断网/sshd 重启/超时三类故障自愈；指纹变更硬阻断；磁盘凭据 100% 密文。

### 阶段 3：远程终端
- **终端即编辑器标签**：自定义 EditorInput 接入 EditorGroup；分屏（1×2/2×2）、拖拽成组、拖出独立窗口；底部面板模式可选。
- 复用 terminal contrib 的 xterm 封装（backend/进程抽象处外挂 SSH shell 通道适配器）；WebGL 渲染、搜索、链接、滚回可配。
- IME 中文输入；右键行为可配；大文本粘贴限速；日志导出；广播输入（多终端同输+醒目横幅）；ZMODEM（rz/sz，可选）；断线覆盖层保留缓冲；本地终端（node-pty/ConPTY）附加。
- 模块：terminal-ssh-backend / terminal-tabs / terminal-addons / terminal-settings。
- 验收：vim/top/htop/tmux 渲染与 resize 正确；24h 无崩溃；10 万行粘贴不冻结；GBK 可选编码；dogfood ≥5 人一周无 P0。

### 阶段 4：文件传输（SFTP）
- SFTP 远端树（懒加载/权限/大小/mtime）；传输引擎（队列化、并发可配、断点续传 offset、限速、size/mtime 校验）；Explorer 拖拽上传（目录递归）/拖出下载。
- 远程文件编辑闭环：双击→临时下载→Monaco 标签（标题带远端标识）→保存回传；远端 mtime 变更检测。
- 冲突弹窗（覆盖/跳过/重命名/比较，可"应用到全部"）；传输中心底部面板；**队列持久化**（强杀可恢复）；目录镜像同步。
- Windows：长路径（>260）/非法字符映射；权限位映射（可执行位）；任务栏进度；完成 Toast。
- 模块：sftp-core / remote-explorer / transfer-center / remote-editor / dnd。
- 验收：1GB 断网续传校验一致；10 并发传输+4 终端不卡顿；强杀后队列恢复；1000+ 文件目录递归上传正确。

### 阶段 5：配置管理与运维增强
- 设置页注册自有配置节（常规/外观/终端/传输/连接/安全/高级）并裁剪无关项；端口转发管理器（L/R/D、随连接启动、失败重试、导入导出）；密钥管理器（生成 ed25519/RSA、公钥幂等部署 authorized_keys）；会话导入（OpenSSH config / XShell 目录 / PuTTY 注册表）；加密备份导出；快捷命令。
- 验收：500 条 XShell 导入保真；L/R/D 隧道 curl/redis-cli 实测；公钥部署幂等；回归全过。

### 阶段 6：打磨与发布
- 性能（参考机冷启动 P50 ≤3s、多会话内存治理）；崩溃收集闭环；72h 压测；空状态引导/首次运行向导；自动更新灰度与回滚；签名（EV 建议）/SmartScreen；SBOM/CVE 跟踪（高危 48h 响应）。
- 验收：crash-free ≥99.5%；Beta→GA 升级演练；文档齐备（快速上手/FAQ/快捷键/迁移指南）。

## 3. 产品原型要点（布局与交互基线）

- **主界面五区**：活动栏（连接/文件/隧道/扩展/设置）｜侧边栏（默认连接管理树）｜主区=**终端标签占据编辑器区**（与远程文件 Monaco 标签同层，支持分屏/拖出）｜底部面板=传输中心（传输/输出/日志页签）｜状态栏（焦点会话连接状态/协议/延迟 RTT/编码 | 隧道数/传输数/通知）。
- **交互总原则**：① 一切功能命令面板可达；② 连接状态三处一致（侧边栏圆点/终端标签/状态栏；绿在线/黄重连/灰未连/红失败）；③ 断线不丢上下文（终端缓冲保留、传输队列可恢复）；④ 危险操作二次确认；⑤ 空状态大入口引导（新建连接/从 XShell·PuTTY·ssh config 导入）。
- **连接管理**：分组→主机→会话子节点（终端×N/SFTP/隧道×N）；双击连接并按"连接后动作"开终端；右键菜单（连接/新建终端/SFTP/端口转发/复制/属性/删除）；搜索（别名/IP/标签/拼音首字母）；新建对话框（基本信息/认证三选一：密码|密钥|Agent/高级折叠：跳板机·编码·Keepalive·连接后动作/测试连接按钮）；密码默认不保存，开启时明示 DPAPI 加密；首次连接指纹确认（算法+SHA256 大字展示）。
- **终端**：右键（复制/粘贴/搜索/广播/导出日志/ZMODEM）；Ctrl+F 浮层搜索；超链接 Ctrl+点击；广播模式状态栏横幅；断线覆盖层区分"重连中(第 n 次)"与"失败"，重连后缓冲保留并加分隔线。
- **传输**：侧边栏"文件"视图（远端树+面包屑路径条）；传输中心队列（方向/路径对/进度/速度 ETA/暂停继续取消/失败自动重试）；冲突弹窗并列大小+时间；任务栏总进度+完成 Toast。
- **配置**：设置页分组见阶段 5；隧道面板表格化（类型/本地端口/远端目标/绑定地址/启停/随连接启动）；密钥管理器（列表+生成向导+部署）；首次运行三步向导。

## 4. AI 能力集成模块

### 场景（S1–S7）
| # | 场景 | 触发 | 优先级 |
|---|---|---|---|
| S1 | 智能命令生成（NL→Shell） | 命令输入条 Ctrl+I / 右键 | P0 |
| S2 | 终端错误诊断 | 报错行内"AI 诊断"/选中右键 | P0 |
| S3 | 命令解释 | 选中 Ctrl+E / 右键 | P0 |
| S4 | Ops 问答面板 | 侧边栏 AI 视图 | P1 |
| S5 | 日志分析 | 选中文本/远端日志右键 | P1 |
| S6 | 脚本/配置生成与审查 | 问答面板/命令条 | P2 |
| S7 | 会话主动诊断 | 自动检测 | v2 |

### 边界红线（七条）
1. 永不自动执行：命令必须用户确认后手动插入；2. 凭据永不入 prompt；3. 上下文最小化（默认仅选区，终端缓冲默认关闭）；4. 脱敏可控（IP/主机名默认脱敏，可扩展正则）；5. 降级零影响（AI 不可用时核心功能无感）；6. 输出免责 + 危险命令本地规则库强制二次判定（rm -rf /、mkfs、dd of=、fork 炸弹、chmod -R 777 / 等；模型自评 risk 仅参考）；7. 用量透明 + 预算限额。

### 技术方案
- 选型：**OpenAI 兼容协议 + 自研薄适配层**（Node 原生 fetch + AbortController + SSE 手工解析，零运行时依赖）；Provider：openai-compatible / azure / ollama / 自定义 baseURL（企业网关）；BYOK（Key 存 DPAPI）；双档模型 fast/strong。openai npm SDK 排除；内置本地模型 v1 排除（Ollama 外挂覆盖离线）。
- 接口契约：`POST {baseURL}/v1/chat/completions`；`{model, messages, stream:true, temperature, response_format?}`；SSE delta 增量；取消全链路 AbortController ≤100ms。
- 模块架构：
  - 渲染层 contrib：ai-inline-actions / ai-chat-panel / ai-command-bar / 结果渲染（Markdown+命令卡片，卡片可插入终端）
  - 主进程：ai-provider-registry / ai-orchestrator（全局并发≤3、同会话串行、首 token 15s 总 120s 超时、网络/5xx/429 退避重试≤3、流式出 token 后不重试、token 预算告警）/ context-builder+guard（Schema：sceneId 白名单、片段≤8 段×4000 字符、历史≤10 轮；注入防护：终端输出置 data 域+定界+模式检测；脱敏规则库：密码提示行/PEM 私钥/AK·SK·token 模式）/ prompt-templates（版本化）/ response-parser（SSE 分帧、JSON mode 校验、fenced bash 命令块提取）/ ai-store
  - 结构化契约：`{analysis, commands:[{cmd, explanation, risk: safe|caution|danger}]}`，risk 本地规则库重判；解析失败降级纯文本。
- 错误矩阵：网络/超时→重试+提示；401/403→跳设置；429→Retry-After 退避；内容过滤→建议改写；上下文超限→自动截断重发一次；5xx→重试+切换建议；解析失败→降级纯文本；未捕获→禁用 AI 入口+遥测。错误仅 toast/会话内卡片，不弹系统模态。
- 编排：AI-T 并行基础轨（阶段 3/4 期间；MA1=单 Provider 沙箱全链路）→ 阶段 5A（M5 后；MA2=P0 场景+联调+安全用例）→ 阶段 6 GA（MA3=指标+文档）。
- 配置键：ai.enabled / ai.provider / ai.endpoint / ai.apiKey(DPAPI) / ai.model.fast|strong / ai.streaming / ai.timeout.firstTokenMs|totalMs / ai.maxConcurrency / ai.context.terminalBuffer(off|selection|window) / ai.privacy.maskHosts / ai.privacy.customMaskRules / ai.usage.monthlyTokenBudget / ai.history.retentionDays / ai.dangerousCommandRules。
- 性能准入：首 token P50≤3s / P95≤8s；流式主线程阻塞单次 <50ms；取消 ≤100ms；空闲内存增量 ≤50MB、100 轮会话 ≤150MB。
- 安全专项：敏感泄露用例集拦截 100%；注入样本集（≥25 条）无越权；危险命令 danger 级 100% 红色确认。
- 文档：《AI 助手使用指南》（用户）/《AI 服务接入指南》（私有网关/合规/数据出境/Ollama）/《AI 模块开发指南》（适配器/模板/ schema/新场景步骤）。

## 5. 风险清单

| 风险 | 等级 | 缓解 |
|---|---|---|
| VSCode 上游同步冲突 | 高 | 隔离原则+基线冻结+安全补丁 cherry-pick |
| 裁剪引发隐性依赖断裂 | 高 | 阶段 0 裁剪 ADR；隐藏优先于删除 |
| ssh2 与老旧/国产化 sshd 互操作 | 中 | 算法协商日志+可配算法集+外部 ssh.exe 兜底 |
| Electron 内存预期 | 中 | 内存治理+发布口径管理 |
| 证书/签名周期 | 低 | 阶段 1 启动采购 |
| AI 危险命令误标 | 高 | 本地规则库强制重判+红色二次确认 |
| Prompt 注入（终端输出藏指令） | 高 | data/instruction 分离+定界+检测+输出白名单 |
| 数据合规 | 中 | 默认脱敏+私有网关指引+Ollama 离线 |
| API 成本失控 | 中 | 用量统计+预算告警+上下文长度控制 |

## 6. 当前执行任务（2026-10-16 开发指令）

1. 从官方仓库下载最新版 VSCode 完整源码，按官方编译指南完成本地环境配置与源码编译，确保可正常运行开发构建版本
2. 基于本文档对 VSCode 前端页面模块定制化改造（品牌、连接管理、终端等），保留原有架构兼容性
3. 按第 2 节阶段清单顺序逐一开发各功能模块，每完成一步立即功能验证与单元测试
4. 整体集成测试，确保改造后页面与各模块无兼容性问题，编译产物可正常运行
5. 输出可正常运行的定制化构建版本，git 提交所有修改的源码文件

**执行策略**：以阶段 0→1→2→3→… 为主线；因体量巨大，按里程碑分批交付，每批完成后向用户简报；每阶段完成即 git commit；安装缺失工具链（git、VS Build Tools）前须征得用户同意。
