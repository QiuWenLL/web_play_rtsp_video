# 网页端 RTSP 零延迟播放技术验证方案（PRD）

## 1. 概述
### 1.1 目的
解决当前 HLS 方案延迟过高（十多秒）的问题。基于现有硬件生态（RK3576/RK3588 + ZLMediaKit），在网页端实现 **H.264/H.265 编码下 <500ms 的极致低延迟** 视频播放。

### 1.2 核心技术路线
*   **后端（统一）**：ZLMediaKit 负责将 IPC 的 RTSP 流转换为低延迟的 WebSocket 流（WS-FLV 或 WS-fMP4），**不进行视频重编码**。
*   **前端方案 A（WASM 软解）**：通过 WebAssembly（`libffmpeg.wasm` 机制）在前端由 CPU 软解视频流，兼容性极好。
*   **前端方案 B（WebCodecs 硬解）**：通过浏览器原生 WebCodecs API 调用客户端 GPU 硬解，性能极高。

---

## 2. 后端基建：ZLMediaKit 配置（方案 A、B 共用）

在进行前端验证前，必须确保 ZLMediaKit 已成功拉取 RTSP 流并开启对应的低延迟协议。

### 2.1 验证步骤
1.  **流媒体拉流**：通过 ZLMediaKit 的 RESTful API（`addStreamProxy`）或配置文件，拉取摄像机的 RTSP 流。
2.  **获取对应流地址**：
    假设你在 ZLMediaKit 中定义的 `vhost` 为 `__defaultVhost__`，`app` 为 `live`，`stream` 为 `test`。
    请确保以下两个低延迟播放地址在局域网内可访问：
    *   **WS-FLV 地址**（用于方案 A）：`ws://<ZLMediaKit_IP>:<http_port>/live/test.live.flv`
    *   **WS-fMP4 地址**（用于方案 B）：`ws://<ZLMediaKit_IP>:<http_port>/live/test.live.mp4`

---

## 3. 方案 A 实现指南：WASM 软解（Jessibuca 方案）

该方案原理与你截图中的 `libffmpeg.wasm` 完全一致。我们采用业界成熟的开源播放器 `Jessibuca` 进行快速实现。

### 3.1 准备工作
1.  前往 [Jessibuca 码云/GitHub 仓库](https://github.com) 下载最新版的 Release 包。
2.  将以下核心文件拷贝到你的前端项目的**静态资源目录**（确保浏览器能直接通过 URL 访问到它们）：
    *   `jessibuca.js` (播放器主逻辑)
    *   `decoder.js` 和 `decoder.wasm` (WASM 解码核心，对应你截图中的 `libffmpeg.wasm`)

### 3.2 编译器代码实现 (index_wasm.html)
创建一纯静态 HTML 文件，填入以下代码并运行：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>方案A测试：WASM低延迟播放</title>
    <!-- 引入播放器核心 JS -->
    <script src="./jessibuca.js"></script>
    <style>
        #video-container { width: 800px; height: 450px; background: #000; }
        .controls { margin-top: 10px; }
    </style>
</head>
<body>
    <h2>方案 A：WASM 软解测试 (<500ms)</h2>
    <div id="video-container"></div>
    
    <div class="controls">
        <input type="text" id="stream-url" value="ws://127.0.0.1:80/live/test.live.flv" style="width: 400px;">
        <button onclick="playVideo()">播放</button>
        <button onclick="pauseVideo()">停止</button>
    </div>

    <script>
        let jessibucaPlayer = null;

        function initPlayer() {
            jessibucaPlayer = new Jessibuca({
                container: document.getElementById('video-container'),
                videoBuffer: 0,                // 核心：设置缓冲时间为 0 秒，实现极致低延迟
                isResize: false,
                text: "",
                loadingText: "视频加载中...",
                useWasm: true,                 // 强制开启 WASM 解码
                useMSE: false,                 // 关闭 MSE 以保证低延迟
                // 必须指定 decoder.js 的正确相对/绝对路径
                decoder: './decoder.js' 
            });

            // 监听错误日志，便于调试
            jessibucaPlayer.on('error', (error) => {
                console.error('WASM Player Error:', error);
            });
        }

        function playVideo() {
            const url = document.getElementById('stream-url').value;
            if (!jessibucaPlayer) initPlayer();
            jessibucaPlayer.play(url);
        }

        function pauseVideo() {
            if (jessibucaPlayer) {
                jessibucaPlayer.pause();
                jessibucaPlayer.destroy();
                jessibucaPlayer = null;
            }
        }

        // 初始化
        initPlayer();
    </script>
</body>
</html>
```

### 3.3 方案 A 验收指标
*   **延迟测试**：对着摄像机挥手，观察网页画面，延迟应在 **200ms - 500ms** 之间。
*   **性能观察**：打开任务管理器，播放 H.265 视频时，该浏览器页面的 **CPU 使用率**会有明显上升（因为是 CPU 软解）。

---

## 4. 方案 B 实现指南：WebCodecs 硬解（极低功耗方案）

该方案利用浏览器原生的硬件解码能力。Jessibuca 同样支持此配置，我们将通过参数切换直接转化为方案 B。

### 4.1 注意事项（避坑指南）
*   **安全上下文限制**：WebCodecs API 必须在 **HTTPS** 环境或者 **http://localhost**（或 127.0.0.1）下才能启用，普通局域网 HTTP IP 无法使用。
*   **流格式**：配合 ZLMediaKit 时，推荐使用 **WS-fMP4** 流。

### 4.2 编译器代码实现 (index_webcodecs.html)

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>方案B测试：WebCodecs硬解低延迟</title>
    <script src="./jessibuca.js"></script>
    <style>
        #video-container { width: 800px; height: 450px; background: #000; }
        .controls { margin-top: 10px; }
    </style>
</head>
<body>
    <h2>方案 B：WebCodecs GPU硬解测试 (更低延迟/极低功耗)</h2>
    <div id="video-container"></div>
    
    <div class="controls">
        <!-- 方案B推荐使用 fMP4 格式流 -->
        <input type="text" id="stream-url" value="ws://127.0.0.1:80/live/test.live.mp4" style="width: 400px;">
        <button onclick="playVideo()">播放</button>
        <button onclick="pauseVideo()">停止</button>
    </div>

    <script>
        let jessibucaPlayer = null;

        function initPlayer() {
            // 检测当前浏览器是否支持 WebCodecs
            if (typeof MediaStreamTrackProcessor === 'undefined' && !window.VideoDecoder) {
                alert("当前浏览器不支持 WebCodecs 硬解！请使用 Chrome 94+ 并确保在localhost或HTTPS环境下运行。");
            }

            jessibucaPlayer = new Jessibuca({
                container: document.getElementById('video-container'),
                videoBuffer: 0,                 // 核心：0 缓冲区
                isResize: false,
                text: "",
                loadingText: "硬解加载中...",
                // --- 方案 B 核心参数配置 ---
                forceNoOffscreen: true,         
                useWebCodecs: true,             // 强制开启 WebCodecs 硬解码
                useWasm: false,                 // 关闭 WASM 软解
                useMSE: false                   // 关闭 MSE
            });

            jessibucaPlayer.on('error', (error) => {
                console.error('WebCodecs Player Error:', error);
            });
        }

        function playVideo() {
            const url = document.getElementById('stream-url').value;
            if (!jessibucaPlayer) initPlayer();
            jessibucaPlayer.play(url);
        }

        function pauseVideo() {
            if (jessibucaPlayer) {
                jessibucaPlayer.pause();
                jessibucaPlayer.destroy();
                jessibucaPlayer = null;
            }
        }

        initPlayer();
    </script>
</body>
</html>
```

### 4.3 方案 B 验收指标
*   **延迟测试**：延迟通常能压减到 **100ms - 300ms**。
*   **性能观察**：播放 H.265 视频时，浏览器 **CPU 占用极低**，而在任务管理器的性能页中，**GPU（Video Decode/专用解码模块）**会有明显的利用率波动。

---

## 5. 两套方案对比复盘矩阵（测试后填写）

在编译器中完成两个 `.html` 文件的编写和测试后，请根据实际效果填写下表，以决定最终上产线的技术方案：


| 评估维度 | 方案 A (WASM 软解) | 方案 B (WebCodecs 硬解) |
| :--- | :--- | :--- |
| **实测延迟 (ms)** | *例如: 350ms* | *例如: 180ms* |
| **H.264 播放稳定性** | | |
| **H.265 播放稳定性** | | |
| **客户端电脑 CPU 占用**| *高（多路播放易卡顿）* | *极低（支持多路高码率）* |
| **老旧浏览器兼容性** | *极好（只要支持WASM即可）* | *一般（依赖现代内核与安全上下文）* |
| **结论归宿** | **可作为低配/老旧浏览器的降级预案** | **推荐作为现代 PC 浏览器的主力方案** |
