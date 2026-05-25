# 网页端 RTSP 极低延迟播放快速测试指南

本文档将指导你如何基于本仓库的源码，跑通 RTSP 视频流在网页端的极低延迟播放测试。

## 1. 架构原理回顾
本仓库不直接在前端解析 RTSP（难度大且延迟高），而是基于以下链路实现：
`RTSP 摄像机` -> `ZLMediaKit (协议转换)` -> `WebSocket (FLV/fMP4)` -> `Jessibuca 前端播放器` -> `浏览器 (WASM 软解 / WebCodecs 硬解)`。

---

## 2. 准备工作
进行测试前，你需要具备：
1. **测试摄像机**：一路可访问的 RTSP 流地址（建议先使用 **H.264** 编码进行基础验证）。
2. **流媒体服务器**：一个正在运行的 **ZLMediaKit** 实例，并记住它的 HTTP 端口和 API `secret`。
3. **前端代码**：确保当前工作区下已有 `jessibuca.js`, `decoder.js`, `decoder.wasm` 三个核心库文件（由于已上传本仓库，clone 下来即有）。

---

## 3. 核心测试流程

### 第一步：让 ZLMediaKit 拉取摄像机流
打开任何支持 curl 的终端，向你的 ZLMediaKit 服务器发送 `addStreamProxy` 请求，让其后台拉取摄像机的 RTSP 流：

```bash
# 请将下面方挂号 [ ] 内的参数替换为你自己的真实参数
curl "http://[ZLM_IP]:[ZLM_Port]/index/api/addStreamProxy?vhost=__defaultVhost__&app=live&stream=test&url=[RTSP_URL]&secret=[ZLM_Secret]"
```
*执行成功后，API 会返回 `"code" : 0`。此时，ZLM 已经自动生成了低延迟流。*

我们统一使用 **WS-FLV** 地址流作前端测试（实测最稳），流地址格式为：
👉 `ws://[ZLM_IP]:[ZLM_Port]/live/test.live.flv`

### 第二步：启动本地静态服务器
**🚨 严禁直接双击打开 `.html` 文件！** 浏览器严格的安全策略会导致 WASM 文件跨域或 MIME 类型丢失而报错。

在当前项目目录（`index_wasm.html` 所在的目录），打开终端运行以下 Python 命令启动本地服务器：
```bash
python -m http.server 8080
```

### 第三步：进行页面播放测试
打开浏览器（推荐 Chrome 或 Edge），进行以下两种技术路线的测试。

#### 方案 A：WASM 软解测试（高兼容性）
1. 访问 `http://127.0.0.1:8080/index_wasm.html`
2. 在输入框填入第一步获取的 WS-FLV 地址。
3. 点击播放。
4. **效果预期**：延迟在 200ms - 500ms 左右，观看任务管理器 CPU 使用率会有所上升。

#### 方案 B：WebCodecs GPU硬解测试（极低功耗，推荐！）
1. **必须访问** `http://127.0.0.1:8080/index_webcodecs.html` 或者 `http://localhost:8080/index_webcodecs.html`。
2. 在输入框同样填入 **WS-FLV** 地址（*注：Jessibuca 能够完美解析 FLV 容器并剥离出 H.264 数据塞给硬件，实测比 fMP4 格式容错率更高*）。
3. 点击播放。
4. **效果预期**：延迟可降至 100ms - 300ms，CPU 占用率极低，显卡 (GPU) 的 Video Decode 负担上升。

---

## 4. 常见问题排查与避坑指南 (Troubleshooting)

如果点击播放后没有画面，请按以下步骤排查（按 `F12` 打开控制台 Network 和 Console面板）：

* **坑点 1：环境安全限制（对于方案 B 极为重要）**
  WebCodecs API `VideoDecoder` 对象受到浏览器极其严格的安全限制。**页面 URL 必须处于 `https://` 或者 `127.0.0.1`/`localhost` 环境下**。如果你用局域网 IP (`http://192.168.x.x:8080`) 去访问，浏览器会直接隐藏这个 API，导致初始化失败。

* **坑点 2：浏览器插件的拦截**
  如果你在 Console 看到奇怪的网络阻断报错（比如 `Failed to fetch`），很大可能是被浏览器的翻译插件（如“沉浸式翻译”）、广告拦截插件所拦截。
  👉 **解决办法**：开启浏览器的**无痕窗口 / 隐私模式 (Incognito)** 重新打开网址测试。

* **坑点 3：ZLMediaKit 源流断开**
  如果 WebSocket (`WS`) 连接提示成功（状态码 101），但没有数据往下传，请检查摄像机网线或使用 VLC 播放器验证原始 RTSP流是否健康。可调用 ZLM 接口 `http://<IP>:<Port>/index/api/getMediaList?secret=xxx` 检查流是否还在线。