# DigitalHumanLive

一款 macOS 原生数字人直播系统，支持根据预设场景内容进行自动化直播，自动回复观众弹幕/评论。

## 功能特性

- **多场景配置**：支持带货、聊天、知识分享等多种直播场景，每个场景独立配置形象、话术、回复规则
- **数字人直播**：静态照片 + AI 口型驱动 + 微动效果，生成实时视频流
- **自动回复**：关键词规则（毫秒级响应）+ LLM 智能回复（兜底），自动处理观众弹幕
- **快手推流**：通过 RTMP 协议直接推流到快手直播间，支持断线自动重连
- **纯本地运行**：零服务器成本，全部 AI 能力（TTS/LLM/OCR）本地计算，隐私安全

---

## 技术栈详解

### 1. 应用层 — SwiftUI

**选型理由**
- macOS 原生框架，性能和用户体验最佳
- 声明式 UI 语法，前端开发者（React/Vue 背景）上手曲线平缓
- 与 macOS 系统能力（Vision OCR、进程管理、电源管理）深度集成

**核心职责**
- 场景配置管理面板（增删改查场景、编辑脚本、设置回复规则）
- 直播间控制台（开播/停播、推流状态监控、实时弹幕列表）
- 数字人预览窗口（实时画面预览、音频波形可视化）
- 通过 `Process` API 管理 FFmpeg 和 Python 子进程

### 2. 数字人渲染 — 音频特征驱动 + 预置口型

**为什么不选 Wav2Lip/SadTalker？**
- Wav2Lip 需要 GPU 推理，Mac Intel 机型无法流畅运行，M 系列芯片也负载较高
- SadTalker 生成速度慢，难以做到实时 25fps
- 两者模型体积大（数百 MB 到数 GB），增大安装包体积

**MVP 方案设计**
```
音频 PCM 数据
    ↓
分帧计算音量 / 频谱能量特征
    ↓
映射到 5 个口型等级：[闭嘴] [微张] [半张] [大张] [O型]
    ↓
根据等级叠加对应口型遮罩到原图
```

用户在配置中标注 5 个口型对应的嘴部区域坐标，运行时实时切换。

**微动效果**
- 呼吸动画：整体缩放 1.0 → 1.02 → 1.0，周期 3-4 秒
- 眨眼动画：间隔 3-6 秒随机触发，150ms 时长，使用预置闭眼图片
- 头部微动：正弦波偏移 2-3 像素，周期 5-8 秒

**升级路径**：MVP 跑通后，可引入 Wav2Lip 作为高配选项，由用户自行选择。

### 3. 语音合成 — Edge-TTS

**选型理由**
- 完全免费，无需 API Key，零调用成本
- 微软 Azure 语音合成引擎，中文效果自然流畅
- 支持 40+ 种中文音色，可调节语速、音调
- Python 库安装简单：`pip install edge-tts`

**用法示例**
```python
import edge_tts

communicate = edge_tts.Communicate("家人们今天活动价只要29块9！", "zh-CN-XiaoxiaoNeural")
await communicate.save("output.mp3")
```

**缓存机制**
- 相同文本的 TTS 结果缓存到 `~/.digital_human/tts_cache/`，避免重复生成
- 缓存键为文本内容的 MD5 哈希
- 缓存文件定期清理（保留最近 7 天）

**升级路径**：可选接入 GPT-SoVITS 进行音色克隆，训练自己的专属主播声音。

### 4. 大语言模型 — Ollama + Qwen2.5:7B

**选型理由**
- 完全本地运行，无需网络，零 API 费用
- Ollama 在 macOS 上安装极简单（一行命令），M 系列芯片有专门优化
- Qwen2.5:7B 中文能力优秀，回复口语化自然，7B 参数量在 Mac 上推理速度可接受
- 支持自定义 System Prompt 定义主播人设

**安装方式**
```bash
# 安装 Ollama
curl -fsSL https://ollama.com/install.sh | sh

# 下载模型
ollama pull qwen2.5:7b
```

**API 调用**
```python
import requests

response = requests.post("http://localhost:11434/api/generate", json={
    "model": "qwen2.5:7b",
    "prompt": "你是一位热情的带货主播。用简短口语化方式回复这条弹幕，控制在30字以内。弹幕：这个好吃吗？",
    "stream": False,
    "options": {"temperature": 0.8, "num_predict": 80}
})
```

**降级策略**
- Ollama 未启动时，降级到纯关键词规则回复
- LLM 响应超时（>3s）时，同样降级到关键词规则

**升级路径**：可配置为调用云端 API（DeepSeek/通义千问/ChatGPT），在本地模型不够用时切换。

### 5. 弹幕识别 — macOS Vision 框架

**选型理由**
- macOS 原生框架，无需下载额外模型，零安装成本
- 中文识别准确率足够应对直播弹幕场景
- 实时性好，单次识别耗时 < 100ms
- 与 Swift 代码无缝集成

**工作流程**
```
快手直播后台（浏览器）
    ↓
用户框选弹幕区域（首次配置，保存坐标）
    ↓
每秒截图 1-2 次 → Vision OCR 识别文字
    ↓
弹幕去重（编辑距离 < 3 视为同一条，3 秒时间窗去重）
    ↓
提取新增弹幕 → 送入回复引擎
```

**去重策略**
- 文本相似度：Levenshtein 编辑距离 < 3 认为是同一条
- 时间窗口：3 秒内相同内容只算一条
- 系统消息过滤：可配置是否处理 "xxx 进入直播间" 等提示

**局限性**
- 依赖用户正确框选弹幕区域
- 快手后台界面改版可能导致区域失效（需重新配置）
- 密集弹幕时可能遗漏（可接受，MVP 阶段够用）

**升级路径**：官方开放弹幕 API 后，可替换为 API 直连方案，延迟和准确率都会大幅提升。

### 6. 直播推流 — FFmpeg RTMP

**选型理由**
- 音视频处理领域的事实标准，成熟稳定
- 支持 H.264 视频编码 + AAC 音频编码，兼容所有直播平台
- 可通过命令行参数精细控制码率、帧率、分辨率
- 可嵌入 App Bundle 分发，用户无需单独安装

**推流参数**
```bash
ffmpeg -f rawvideo -pix_fmt bgr24 -s 1080x1920 -r 25 -i video_pipe \
       -f s16le -ar 44100 -ac 1 -i audio_pipe \
       -c:v libx264 -preset fast -b:v 2000k \
       -c:a aac -b:a 128k \
       -f flv rtmp://live-push.kuaishou.com/xxx/stream_key
```

**断线重连**
- 监控 FFmpeg 进程退出码
- 网络异常 3 秒内自动重试，最多 5 次
- 超过 5 次暂停直播，弹窗提示用户

**快手推流地址获取**
1. 快手 APP → 创建直播间 → 选择「电脑直播/第三方推流」
2. 获得 RTMP 服务器地址 + 流密钥
3. 填入本软件 → 开播

### 7. Swift-Python 桥接 — Process + JSON

**为什么不选 PythonKit？**
- PythonKit 需要精确配置 Python 路径，嵌入 Python 时路径处理复杂
- 单进程内运行 Python，崩溃会影响整个 App
- 调试困难，GIL 可能阻塞主线程

**方案设计**
- Swift 通过 `Process` 启动嵌入的 Python 子进程
- 双方通过 `stdin/stdout` 交换 JSON 消息
- Python 进程崩溃不影响主 App，Swift 可自动重启

**通信协议**
```swift
// Swift 发送
let request: [String: Any] = [
    "cmd": "tts",
    "text": "今天活动价29块9！",
    "voice": "zh-CN-XiaoxiaoNeural"
]
pythonProcess.stdin?.write(jsonData)

// Python 返回
{
    "status": "ok",
    "audio_path": "~/.digital_human/tts_cache/a1b2c3.mp3",
    "duration": 2.5
}
```

---

## 项目结构

```
DigitalHumanLive/
├── README.md
├── .gitignore
├── docs/
│   └── superpowers/
│       └── specs/
│           └── 2026-05-22-digital-human-live-design.md
│
├── DigitalHumanLive/               # SwiftUI App 主工程
│   ├── App/
│   │   └── DigitalHumanLiveApp.swift
│   ├── Views/
│   │   ├── SceneConfigView.swift   # 场景配置面板
│   │   ├── LiveControlView.swift   # 直播间控制台
│   │   └── AvatarPreviewView.swift # 数字人预览窗口
│   ├── Models/
│   │   ├── LiveScene.swift         # 场景数据模型
│   │   ├── ScriptSegment.swift     # 脚本段落模型
│   │   └── ReplyRule.swift         # 回复规则模型
│   ├── Modules/
│   │   ├── SceneMgr.swift          # 场景管理器
│   │   ├── ScriptEng.swift         # 脚本引擎
│   │   ├── ReplyEng.swift          # 回复引擎
│   │   ├── MediaPipe.swift         # 媒体合成管道
│   │   ├── Streamer.swift          # 推流器
│   │   └── ChatOCR.swift           # 弹幕识别
│   └── Utils/
│       └── PythonBridge.swift      # Swift-Python 通信桥
│
├── PythonScripts/                  # Python 脚本
│   ├── tts_engine.py               # TTS 引擎封装
│   ├── llm_client.py               # LLM 客户端
│   ├── audio_analyzer.py           # 音频特征分析
│   └── main.py                     # Python 服务入口
│
└── Resources/                      # 内置资源
    ├── Python3.11/                 # 嵌入的 Python 运行时
    └── FFmpeg/                     # 嵌入的 FFmpeg
```

---

## 系统要求

| 项目 | 最低配置 | 推荐配置 |
|------|---------|---------|
| 系统 | macOS 13 (Ventura) | macOS 14+ (Sonoma) |
| 芯片 | Intel / Apple Silicon | Apple Silicon (M1+) |
| 内存 | 8 GB | 16 GB |
| 存储 | 2 GB 可用空间 | 5 GB 可用空间（含模型） |
| 网络 | 上传带宽 ≥ 5 Mbps | 上传带宽 ≥ 10 Mbps |

---

## 项目状态

目前处于设计阶段，详见 [设计文档](docs/superpowers/specs/2026-05-22-digital-human-live-design.md)。

**里程碑规划**

| 阶段 | 目标 | 预计周期 |
|------|------|---------|
| MVP v0.1 | 单场景直播、预设脚本轮播、基础口型、FFmpeg 推流 | 3-4 周 |
| v0.2 | 多场景管理、弹幕 OCR、关键词自动回复 | 2-3 周 |
| v0.3 | LLM 智能回复、TTS 音色克隆、字幕样式 | 2-3 周 |
| v0.4 | 抖音/小红书平台适配、数据分析看板 | 3-4 周 |

---

## 许可证

MIT
