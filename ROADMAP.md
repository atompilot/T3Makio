# T3Makio ROADMAP

更新：2026-11-27（项目初始化）。

## 项目定位

**T3Makio** 是独立于冻结版 Makio 桌面端的下一代 AI 对话交互式 3D 快速建模产品，基于开源 [`pingdotgg/t3code`](https://github.com/pingdotgg/t3code)（MIT 协议）作为跨端底座演进：

- **产品目标**：提供桌面端与移动端（iOS / Android）同源的 AI 对话快速建模体验——点选 3D 零件即可直接向 Agent 提修改意见，原地实时预览重建结果，并具备完整账号体系、云端中继与原生推送提醒。
- **当前运行模式**：自托管 / 连接用户本机桌面算力（运行 OpenSCAD、本机 Coding Agent CLI 与 Makio 引擎），从第一天即按完整产品级底座建设，不做临时过渡方案。
- **原仓边界**：原 Makio 桌面端仓库保持**完全冻结**，继续支撑日常建模使用；T3Makio 所有开发均在本仓 `main` 分支及后续特性分支独立推进。

## 已对齐架构决策

1. **底座与外壳（复用 T3 Code）**
   - 直接复用 `apps/server`（多 Provider 适配与线程编排）、`apps/desktop`（Electron）、`apps/mobile`（Expo / React Native）与 `infra/relay`（账号、托管隧道控制面、APNs / FCM 推送）。
   - 上游 `upstream` 指向 `pingdotgg/t3code`（已禁止误推），保留 MIT 许可声明；UI 裁剪尽量通过独立模式/配置开关实现，降低后续参考或合并上游的冲突成本。
2. **3D 业务引擎（独立 Sidecar + MCP 自动注入）**
   - 从冻结版 Makio 迁移并独立维护 3D 核心服务（OpenSCAD 构建、项目/模型/版本管理、46 个 `makio_*` MCP 工具、参数化预设、结构配合、模型检查、排盘/切片与设备读取）。
   - 作为随主机服务启动的 Sidecar 运行，并挂入 `apps/server/src/mcp/`，使所有 Provider（Claude Code、Codex 等）开箱自带 Makio 建模工具。
3. **3D 视口与双向闭环**
   - 复用 Makio 现有 React + Three.js 审查视口，输出可嵌入的轻量 3D 审查页面：
     - **桌面端**：嵌入审查工作台，与同源对话侧栏联动。
     - **移动端**：直接复用 `apps/mobile/src/features/browser/TunnelWebView.tsx`（原生 `T3PreviewTunnel` 模块），透明走加密隧道访问主机本机 Makio 端口。
   - **双向联动**：点选组件 `C/I` 码自动作为上下文标签注入对话输入框；Agent 调用 Makio MCP 完成模型修改与构建后，自动触发 3D 画布原地重载。
   - **并发规则**：同一模型对应一条主对话线程，同一时刻只允许 1 个活跃 Turn，双端实时同步流式输出与停止控制。
4. **云端中继与发布基础设施**
   - **目标域名**：`makio.splaz.app`（如 `relay.makio.splaz.app` 与隧道子域，基于 Cloudflare Workers + Managed Tunnel）。
   - **数据库**：`infra/relay` 当前为约 10 张标准 Drizzle Postgres 表，经 Cloudflare Hyperdrive 连接；研发时替换为轻量 Postgres 托管或自建库（具体选型在 M1 落地时敲定），尽量不改表结构与查询层。
   - **移动端推送与分发**：已备好 Apple Developer 账号与 Cloudflare 基础设施；所有账号、私钥、令牌统一放本机私有环境变量与部署 Secrets，**严禁写入 Git**。

## 里程碑规划（初稿）

### M0 · 底座跑通与品牌/基建隔离
- 验证本仓 `apps/server`、`apps/web`、`apps/desktop`、`apps/mobile` 本地开发链路可正常启动。
- 完成 `T3Makio` 应用标识（Bundle ID、App Name、本地数据目录）隔离，避免与本机日常使用的原版 T3 Code 和原版 Makio 产生端口、数据目录或配置冲突。

### M1 · 云端 Relay 部署（`makio.splaz.app`）与移动推送链路打通
- 替换 `infra/relay/src/db.ts` 中的 PlanetScale 绑定为轻量数据库方案。
- 部署 `makio.splaz.app` 对应的 Relay Worker、隧道域名、Clerk 认证与 APNs 推送配置，完成手机端与 Mac 本机的外网直连及后台推送验证。

### M2 · Makio 引擎接入与桌面/移动同源 3D 对话闭环
- 迁入 Makio 核心服务为独立包，随 `apps/server` 自动拉起并把 `makio_*` 工具注册进 T3 MCP 暴露层。
- 构建移动端友好的可嵌入 3D 审查视图（项目/模型切换、全屏 3D 点选、版本切换），接入桌面审查页与移动端 `TunnelWebView`。
- 打通 `点选组件 → 注入 @C12 上下文 → Agent 改模构建 → 自动刷新 3D 视口` 全链路。

### M3 · 3D 建模工作台心智裁剪与体验打磨
- 隐藏或折叠面向纯编程的 Git / Worktree / Diff / PR / Terminal 入口，把「模型列表 + 3D 视口 + 对话抽屉」设为默认主工作区。
- 真机打磨移动端手势（3D 旋转缩放 vs 抽屉拖动）、大 STL 内存控制、构建完成推送点击直达对应模型视口。
