# 网页端 RTSP 零延迟播放技术验证项目 (PRD & 任务看板)

## 1. 项目概述 (Project Overview)
- **项目目标**：替代现有高延迟 HLS 方案，在终端（RK3576/RK3588等环境）网页端实现 H.264/H.265 编码的极低延迟流媒体播放（延迟控制在 500ms 以内）。
- **技术路线**：
  - **后端**：利用 ZLMediaKit 作为流媒体网关，进行协议转换（RTSP -> WebSocket），**不进行视频重编码**。提供 WS-FLV 和 WS-fMP4 媒体流。
  - **前端**：采用 Jessibuca 开源播放器套件作为播放器内核，设计两套平行验证方案：
    - **方案A（软解）**：基于 WebAssembly (`libffmpeg.wasm`) 进行 CPU 软解。
    - **方案B（硬解）**：基于浏览器原生 WebCodecs API 调用 GPU 硬解（极低功耗）。

---

## 2. 约束与风险评估 (Risk & Constraints)
⚠️ **实施前必读，请在开发过程中严格规避以下坑点**：
1. **WebCodecs 安全上下文限制 (高)**：方案B 必须在 `https://` 或 `http://localhost` (127.0.0.1) 下运行，不允许在配置了普通局域网 IPv4 的 HTTP 地址下运行。后续起本地测试服务器时务必注意。
2. **流协议匹配性 (中)**：方案A (WASM) 推荐配合 `WS-FLV`；方案B (WebCodecs) 推荐配合 `WS-fMP4`，切勿混用导致解码失败。
3. **前端资源跨域及路径 (中)**：WASM 文件 (`decoder.wasm`) 的加载存在相对路径限制，需确保 Web 服务将其和 JS 暴露在同一层级，且设置正确的 MIME Type (`application/wasm`)。

---

## 3. Agent 自动化执行任务看板 (Task Board)

*(Agent 可根据以下 CheckList 逐步认领并执行任务)*

### 阶段一：项目基础搭建 (Phase 1: Initialization) 🟢 
- [x] 1.1 初始化 Git 仓库 (`git init`)
- [x] 1.2 配置项目忽略文件 (`.gitignore`)
- [x] 1.3 根据架构编写前端 HTML 模板 (`index_wasm.html`, `index_webcodecs.html`)
- [x] 1.4 创建自动化管理规划文档 (即本文档)

### 阶段二：前端核心依赖准备 (Phase 2: Dependencies) � 完成
- [x] 2.1 引入 `Jessibuca` 核心静态资源（可通过脚本自动下载或手动放入）。
  - 需要落实的文件：`jessibuca.js`, `decoder.js`, `decoder.wasm`。
- [x] 2.2 验证引入路径是否在 HTML 模板中正确匹配。

### 阶段三：测试环境与基建拉起 (Phase 3: Environment Setup) 🔴 待认领
- [ ] 3.1 编写并启动一个简易本地 Web 服务器（推荐使用 Python `http.server` 或 Node.js `live-server`），解决后续 WASM 跨域和资源加载问题。
- [ ] 3.2 【需人工配合】在局域网内拉起 ZLMediaKit 服务。
- [ ] 3.3 【需人工配合】推送/拉取一路测试用的 RTSP 流到 ZLMediaKit，最终暴露得到 WebSocket 测试地址。

### 阶段四：方案 A 联调与验收 (Phase 4: Scheme A Validation) � 完成
- [x] 4.1 在本地 Web 服务中访问 `index_wasm.html`。
- [x] 4.2 填入获取的 ZLMediaKit `WS-FLV` 地址，进行播放测试。
- [x] 4.3 验收指标测试：
  - [x] 画面正常输出无花屏。
  - [x] 通过“挥手测试”，确认端到端延迟介于 **200ms - 500ms** 之间。
  - [x] 记录播放 H.265 时的平均 CPU 占用率。

### 阶段五：方案 B 联调与验收 (Phase 5: Scheme B Validation) 🔴 待认领
- [ ] 5.1 在本地 Web 服务 (`http://localhost` 或 `http://127.0.0.1`) 访问 `index_webcodecs.html`。
- [ ] 5.2 填入获取的 ZLMediaKit `WS-fMP4` 地址，进行播放测试。
- [ ] 5.3 验收指标测试：
  - [ ] 确认控制台打印启用 WebCodecs 成功，无降级 WASM。
  - [ ] 确认端到端延迟低至 **100ms - 300ms**。
  - [ ] 确认 CPU 占用率极低，GPU 视频解码模块占有率上升。

### 阶段六：项目复盘与交付 (Phase 6: Retrospective & Delivery) 🔴 待认领
- [ ] 6.1 整理两套方案的延迟与性能数据。
- [ ] 6.2 输出最终对比复盘报告（可直接补充到 `readme.md` 底部）。
- [ ] 6.3 提交流程：使用 “代码同步助手” 提交所有代码定案。
