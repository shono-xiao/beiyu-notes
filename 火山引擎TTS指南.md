# OpenClaw + 火山引擎语音合成

> 给 AI Agent 加语音：用火山引擎 TTS，Agent 自己选音色、配脚本、搞定出声。

---

## 👤 人类（3 步）

**1.** 打开 https://console.volcengine.com/speech/new ，开通服务（赠免费礼包）

**2.** 语音合成 → 服务概览 → API 密钥 → 复制 API Key

**3.** 把 Key 发给你的 AI，描述一下想要什么风格的声音（比如"偏冷的青年男声"）

完事。

---

## 🤖 AI 要做的事

收到 Key 和声线偏好后：

### 1. 选音色

打开 https://console.volcengine.com/speech/service/voices ，根据人类描述匹配最合适的 `voice_type`。

（人类可以提前整理一份音色列表放在桌面/文档里给你参考——比如豆包语音的音色清单。）

### 2. 创建桥接脚本 `tts-volcengine.js`

```js
const https = require('https');
const fs = require('fs');
const KEY = '你的API_KEY';
const VOICE = '你选的voice_type';

const payload = JSON.stringify({
  app: { appid: 'your-app-id', cluster: 'volcano_tts' },
  user: { uid: 'your-user-id' },
  audio: { voice_type: VOICE, encoding: 'mp3', speed_ratio: 0.85 },
  request: { reqid: 'tts-' + Date.now(), text: process.argv[3], text_type: 'plain', operation: 'query' }
});

const req = https.request({
  hostname: 'openspeech.bytedance.com',
  path: '/api/v1/tts',
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'X-Api-Key': KEY }
}, res => {
  const chunks = [];
  res.on('data', c => chunks.push(c));
  res.on('end', () => {
    const json = JSON.parse(Buffer.concat(chunks).toString());
    if (json.code === 3000 && json.data) {
      fs.writeFileSync(process.argv[2], Buffer.from(json.data, 'base64'));
      process.exit(0);
    }
    process.exit(1);
  });
});
req.write(payload); req.end();
```

### 3. 配置 OpenClaw

```json
"messages": {
  "tts": {
    "provider": "tts-local-cli",
    "providers": {
      "tts-local-cli": {
        "command": "node",
        "args": ["路径/tts-volcengine.js", "{{OutputPath}}", "{{Text}}"],
        "outputFormat": "mp3",
        "timeoutMs": 30000
      }
    }
  }
}
```

### 4. 重启

```bash
openclaw gateway restart
```

---

## ⚠️ 常见踩坑

| 问题 | 说明 |
|------|------|
| `voice_type` 格式 | 必须是音色页面的确切名称，通常带 `_bigtts` 后缀 |
| Key 混淆 | 火山语音 Key ≠ Seed Speech Key，端点不同 |
| 鉴权方式 | 2.0 音色用 `X-Api-Key` header（body 不放 token），1.0 音色相反 |
| `speed_ratio` | 建议 0.85-0.95，太快像念经，太慢像树懒 |

---

## 参考

- 火山引擎语音合成控制台：https://console.volcengine.com/speech/new
- 音色列表页面：https://console.volcengine.com/speech/service/voices
- 豆包语音音色清单：见同仓库 `豆包语音音色列表.md`
