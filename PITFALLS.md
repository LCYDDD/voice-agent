# 踩坑记录

> 语音智能助手开发过程中遇到的坑，以及解决方案。

---

## 1. MiMo ASR 使用 WebSocket 方式连接失败

### 问题
连接 `wss://asr.mimo.mi.com/v1/asr` 报 WebSocket 连接错误。

### 原因
MiMo ASR v2.5 **已经废弃了 WebSocket 接口**，改为标准 HTTP API。这是 API 版本升级导致的重大变更。

### 解决方案
改用 HTTP POST 请求：

```bash
POST https://api.xiaomimimo.com/v1/chat/completions
model: mimo-v2.5-asr
```

把音频文件转为 base64，通过 `input_audio` 字段传入。

### 参考资料
https://mimo.mi.com/docs/zh-CN/quick-start/usage-guide/audio/Speech-Recognition

---

## 2. MiMo ASR 不支持的音频格式

### 问题
MediaRecorder 默认录制 `audio/webm;codecs=opus`，发送后 ASR 返回 400 错误。

### 原因
MiMo ASR **只支持 wav 和 mp3** 两种格式，不支持 webm/opus。

### 解决方案
方案一：录制时指定支持的 MIME 类型：
```javascript
const mimeTypes = ['audio/mp3', 'audio/mpeg', 'audio/wav', 'audio/wave'];
```

方案二：如果浏览器不支持这些格式（如 Chrome 默认只支持 webm），用 AudioContext 解码 webm 后重新编码为 WAV（PCM 16bit）。

### 关键代码
```javascript
// webm → WAV 转换
async function convertWebmToWavBase64(webmBuffer) {
  const ctx = new AudioContext();
  const audioBuffer = await ctx.decodeAudioData(webmBuffer);
  // ... 编码为 WAV PCM 16bit ...
}
```

---

## 3. MiMo 模型版本已废弃

### 问题
代码中配置了 `mimo-v2-pro` 模型，但调用时报错或无法使用。

### 原因
`mimo-v2-pro`、`mimo-v2-omni`、`mimo-v2-flash` 已于 **2026年6月30日正式废弃**。

### 解决方案
改用新版本模型：
- **文本生成**：`mimo-v2.5-pro`（上下文 1M tokens，支持深度思考/函数调用）
- **语音识别**：`mimo-v2.5-asr`
- **语音合成**：`mimo-v2.5-tts`

### 参考资料
https://mimo.mi.com/docs/zh-CN/quick-start/summary/model

---

## 4. MiMo API 认证方式不同

### 问题
代码中只使用了 `Authorization: Bearer` 认证头，但 MiMo 推荐使用 `api-key` 头。

### 原因
MiMo 兼容 OpenAI 格式，但不是完全一样。MiMo 的 API Key 以 `sk-` 开头，推荐用 `api-key` 请求头，但也支持 `Authorization: Bearer`。

### 解决方案
自动检测 Key 前缀，使用对应的认证方式：
```javascript
if (key.startsWith('sk-') || key.startsWith('tp-')) {
  headers['api-key'] = key;    // MiMo 风格
} else {
  headers['Authorization'] = `Bearer ${key}`;  // OpenAI 风格
}
```

---

## 5. MiMo TTS 返回 base64 音频，不是直接播放

### 问题
MiMo TTS 的响应中音频是 base64 编码的字符串，不能直接播放。

### 原因
MiMo TTS API 返回 `choices[0].message.audio.data` 字段，内容是 base64 编码的 WAV 数据。

### 解决方案
解码 base64 → Blob → Object URL → Audio 播放：
```javascript
const binaryStr = atob(audioData);
const bytes = new Uint8Array(binaryStr.length);
for (let i = 0; i < binaryStr.length; i++) {
  bytes[i] = binaryStr.charCodeAt(i);
}
const audioBlob = new Blob([bytes], { type: 'audio/wav' });
const audioUrl = URL.createObjectURL(audioBlob);
const audio = new Audio(audioUrl);
await audio.play();
```

---

## 6. 事件绑定时机错误导致按钮无响应

### 问题
设置按钮（⚙️）点击没有反应，无法打开配置面板。

### 原因
事件绑定代码写在模块顶层，但 `cacheDom()` 在 `init()` 中才执行。事件绑定执行时 DOM 元素还没缓存到 `dom` 对象，所有 `dom.settingsBtn` 都是 `undefined`。

### 解决方案
将所有事件绑定放到 `bindEvents()` 函数中，在 `cacheDom()` 之后调用：
```javascript
function init() {
  cacheDom();    // 1️⃣ 先获取 DOM
  loadConfig();
  bindEvents();  // 2️⃣ 再绑定事件
  setState(State.IDLE);
}
```

---

## 7. Web Speech API 兼容性差

### 问题
在 Firefox、微信内置浏览器、部分手机自带浏览器上，语音识别完全不能用。

### 原因
Web Speech API 的 `SpeechRecognition` 仅 Chrome/Safari/Edge 支持，Firefox 和微信浏览器不支持。

### 解决方案
默认使用 MiMo ASR（HTTP API），任何浏览器都能用。Web Speech API 作为备选方案。

| 浏览器 | Web Speech 识别 | Web Speech 合成 | MiMo API |
|--------|:--------------:|:--------------:|:--------:|
| Chrome / Edge | ✅ | ✅ | ✅ |
| Safari (iOS) | ✅ | ✅ | ✅ |
| Firefox | ❌ | ✅ | ✅ |
| 微信浏览器 | ❌ | ❌ | ✅ |
| 手机自带浏览器 | ❌ | ❌ | ✅ |

---

## 8. 用户输入的 API Key 存储位置

### 问题
用户输入的 API Key 存在哪里？会不会泄露？

### 原因
纯前端应用没有后端服务器，无法安全地存储敏感信息。

### 解决方案
使用浏览器 `localStorage` 存储：
- Key **只存在用户自己的浏览器里**
- 关闭浏览器不会丢失
- 部署到 Vercel/GitHub 也不会泄露
- 但要注意：如果别人拿到手机并打开 DevTools，可以看到 Key

### 存储位置
```
浏览器 → Application → Local Storage → voice_agent_config
```

---

## 9. MiMo API 超时处理

### 问题
MiMo ASR 请求偶尔卡住无响应，页面一直停留在"处理中"状态。

### 原因
没有设置请求超时，网络问题或服务端延迟导致请求永远挂起。

### 解决方案
使用 `AbortController` 设置超时：
```javascript
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 30000);

try {
  const response = await fetch(url, { signal: controller.signal });
  clearTimeout(timeoutId);
} catch (err) {
  clearTimeout(timeoutId);
  if (err.name === 'AbortError') {
    // 超时处理
  }
}
```

超时时间设置：
- LLM 请求：30 秒
- TTS 请求：15 秒
- ASR 请求：30 秒
- 文本修正：8 秒

---

## 10. MiMo 模型推荐（截至 2026-07）

### 当前可用模型

| 用途 | 模型 ID | 说明 |
|------|---------|------|
| 文本生成（推荐） | `mimo-v2.5-pro` | 1M 上下文，支持深度思考/函数调用 |
| 文本生成 | `mimo-v2.5` | 全模态理解（图/音/视频） |
| 语音识别 | `mimo-v2.5-asr` | 支持中英文、方言 |
| 语音合成 | `mimo-v2.5-tts` | 内置高品质音色 |
| 语音合成（克隆） | `mimo-v2.5-tts-voiceclone` | 上传音频样本克隆音色 |
| 语音合成（设计） | `mimo-v2.5-tts-voicedesign` | 文字描述生成音色 |

### 已废弃模型（2026-06-30 起）
`mimo-v2-pro`、`mimo-v2-omni`、`mimo-v2-flash`、`mimo-v2-tts`

---

## 11. MiMo API 端点

| 服务 | 端点 |
|------|------|
| OpenAI 兼容（按量计费） | `https://api.xiaomimimo.com/v1` |
| OpenAI 兼容（Token Plan） | `https://token-plan-cn.xiaomimimo.com/v1` |
| Anthropic 兼容（按量计费） | `https://api.xiaomimimo.com/anthropic` |
| Anthropic 兼容（Token Plan） | `https://token-plan-cn.xiaomimimo.com/anthropic` |

### API Key 格式
- 按量计费：`sk-xxxxx`
- Token Plan：`tp-xxxxx`

### 参考资料
https://mimo.mi.com/docs/zh-CN/quick-start/summary/first-api-call

---

## 12. `file://` 协议无法使用麦克风

### 问题
在手机上直接用文件管理器打开 `index.html`，麦克风不可用。

### 原因
浏览器的安全策略：`file://` 不被视为"安全上下文"，所有敏感 API（麦克风、摄像头、地理位置）全部被禁止。

### 支持的协议
| 协议 | 麦克风 | 说明 |
|------|:------:|------|
| `https://` | ✅ | 全部支持 |
| `http://localhost` | ✅ | 本机测试 |
| `http://127.0.0.1` | ✅ | 本机测试 |
| `http://<局域网IP>` | ✅ | 同一 WiFi |
| `file://` | ❌ | 不支持 |

### 解决方案
使用 Python 启动本地服务器：
```bash
cd voice-agent/
python -m http.server 8080
```
然后访问 `http://localhost:8080`
