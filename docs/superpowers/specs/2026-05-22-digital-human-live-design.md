# 数字人直播系统 — 系统设计文档

**版本**: 1.0
**日期**: 2026-05-22
**状态**: 待实现

---

## 1. 项目概述

### 1.1 目标

开发一款 macOS 原生数字人直播软件，支持根据预设场景内容进行自动化直播，自动回复观众弹幕/评论，优先支持快手平台，后期可扩展至抖音、小红书。

### 1.2 核心需求

- **多场景配置**：用户可创建多个直播场景（带货、聊天、知识分享等），每个场景独立配置数字人形象、话术脚本、回复规则
- **数字人直播**：静态照片 + AI 口型驱动 + 微动效果，生成实时视频流
- **自动回复**：识别观众弹幕，通过关键词规则或 LLM 生成回复并口播
- **快手推流**：通过 RTMP 协议直接推流到快手直播间
- **低成本优先**：MVP 阶段零服务器成本，全部本地运行

### 1.3 非目标（后续版本）

- 3D 虚拟形象（成本过高，MVP 不做）
- 抖音/小红书平台（快手优先跑通后再扩展）
- 实时换脸/真人克隆（合规风险 + 成本高）
- 多路同时直播（先做单直播间）

---

## 2. 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          数字人直播系统 (macOS App)                        │
├────────────────────────┬────────────────────────┬────────────────────────┤
│      表现层             │       业务层            │       基础设施层         │
│    (SwiftUI)           │    (Swift + Python)    │    (本地/嵌入)          │
│                        │                        │                        │
│ ┌──────────────────┐   │ ┌──────────────────┐   │ ┌──────────────────┐  │
│ │ 场景配置管理面板   │   │ │ 直播编排引擎      │   │ │ Python 运行时    │  │
│ │ • 创建/编辑场景   │◄──┤ │ • 脚本调度        │   │ │ (TTS/LLM/OCR)   │  │
│ │ • 商品话术绑定   │   │ │ • 自动回复决策    │   │ │                  │  │
│ │ • 回复规则设置   │   │ │ • 场景切换        │   │ │ ┌──┐┌──┐┌──┐   │  │
│ └──────────────────┘   │ └────────┬─────────┘   │ │ │TTS││LLM││OCR│   │  │
│ ┌──────────────────┐   │          │             │ │ └──┘└──┘└──┘   │  │
│ │ 直播间控制台      │   │          ▼             │ └──────────────────┘  │
│ │ • 开播/停播      │◄──┤ │   媒体合成管道      │   │                      │
│ │ • 推流状态监控   │   │ │ • 视频帧合成       │   │ ┌──────────────────┐  │
│ │ • 实时弹幕列表   │   │ │ • 音频混合         │   │ │ FFmpeg 推流      │  │
│ └──────────────────┘   │ │ • 字幕叠加         │───┤►│ • RTMP 输出      │  │
│ ┌──────────────────┐   │ └────────┬─────────┘   │ │ • 虚拟摄像头     │  │
│ │ 数字人预览窗口    │◄──┘          │             │ └────────┬─────────┘  │
│ │ • 实时画面预览   │              │             │          │            │
│ │ • 音频波形显示   │              ▼             │          ▼            │
│ └──────────────────┘   │   ┌──────────────┐    │   ┌──────────────┐    │
│                        │   │ 快手直播后台  │    │   │ 快手直播间   │    │
│                        │   │ (浏览器抓屏)  │    │   │ (观众端)    │    │
│                        │   └──────────────┘    │   └──────────────┘    │
└────────────────────────┴────────────────────────┴────────────────────────┘
```

### 2.2 核心数据流

**开播流程：**

```
用户点击开播 → 加载当前场景脚本 → 启动 FFmpeg RTMP 推流
                          ↓
                开始按脚本轮播内容
                          ↓
          语音生成(TTS) → 口型同步 → 视频帧合成 → FFmpeg 编码推流
```

**互动流程：**

```
快手直播后台页面(浏览器) → 定时截图 → Vision OCR 识别弹幕
                          ↓
                弹幕文本 → LLM/关键词匹配 → 生成回复文本
                          ↓
          回复文本 → TTS → 口型同步 → 插入当前直播流
```

**场景切换：**

```
用户切换场景 / 脚本定时器触发 → 保存当前状态 → 加载新场景脚本
                          ↓
                    新内容无缝衔接播出
```

---

## 3. 模块设计

### 3.1 模块划分

| 模块 | 名称 | 职责 | 技术栈 |
|------|------|------|--------|
| Module 1 | SceneMgr | 场景管理（CRUD、切换） | Swift |
| Module 2 | ScriptEng | 脚本引擎（时间轴调度） | Swift |
| Module 3 | ReplyEng | 回复引擎（弹幕处理、回复决策） | Swift + Python |
| Module 4 | MediaPipe | 媒体合成（口型、字幕、帧合成） | Swift |
| Module 5 | Streamer | 推流器（FFmpeg RTMP） | Swift + FFmpeg |
| Module 6 | ChatOCR | 弹幕识别（截图 + OCR） | Swift + Vision |

### 3.2 Module 1: SceneMgr（场景管理）

**职责**：管理多个直播场景的增删改查，以及场景间的切换。

**核心数据结构：**

```swift
struct LiveScene: Codable, Identifiable {
    let id: UUID
    var name: String
    var type: SceneType        // .product / .chat / .knowledge
    var avatarImagePath: String
    var voiceConfig: VoiceConfig
    var scriptSegments: [ScriptSegment]
    var replyRules: [ReplyRule]
    var bgmPath: String?
}

enum SceneType: String, Codable {
    case product      // 带货
    case chat         // 闲聊
    case knowledge    // 知识分享
}
```

**核心方法：**

- `loadScene(id: UUID) -> LiveScene` — 加载场景到内存
- `switchScene(to sceneId: UUID)` — 切换场景（淡入淡出过渡）
- `exportScene(id: UUID) -> URL` — 导出场景为 JSON 文件（备份/分享）
- `importScene(from url: URL) -> LiveScene` — 从文件导入场景

### 3.3 Module 2: ScriptEng（脚本引擎）

**职责**：按照时间轴和触发器调度直播内容的播放顺序。

**脚本段类型：**

```swift
enum SegmentType: String, Codable {
    case intro              // 开场白
    case productIntro       // 商品介绍
    case productDetail      // 商品详情
    case interaction        // 互动引导（如"想要的扣1"）
    case scheduled          // 定时触发（如每5分钟说一次）
    case keywordTrigger     // 关键词触发（如"多少钱"）
    case loop               // 循环内容
}

struct ScriptSegment: Codable, Identifiable {
    let id: String
    let type: SegmentType
    let content: String          // 口播文本
    let duration: TimeInterval   // 预计时长（秒）
    let productId: String?       // 关联商品ID（可选）
    let repeatInterval: TimeInterval?  // 循环间隔（仅 loop 类型）
    let transition: TransitionType     // 过渡动画
}
```

**调度逻辑：**

1. 按顺序播放：`intro` → `productIntro` → `productDetail` → `loop`
2. 弹幕触发关键词 → 暂停当前段落 → 插入 `keywordTrigger` 回复段落 → 恢复原段落
3. 定时器到期 → 插入 `scheduled` 段落 → 恢复原段落

### 3.4 Module 3: ReplyEng（回复引擎）

**职责**：处理弹幕输入 → 决策是否回复 → 生成回复文本。

**两级回复策略：**

**第一级：关键词规则匹配（优先，延迟 < 100ms）**

规则由用户在场景配置中预设，匹配到关键词立即返回固定回复。

示例规则：
- 关键词 `["价格", "多少钱"]` → 回复 `"今天活动价29块9！"`
- 关键词 `["怎么买", "下单"]` → 回复 `"点击右下角小黄车就能下单哦"`
- 关键词 `["包邮吗"]` → 回复 `"全场包邮，48小时发货"`

每条规则有独立的 `cooldown_seconds`（冷却时间），防止同一内容反复回复。

**第二级：LLM 智能回复（兜底，延迟 1-3s）**

当关键词规则未命中时，启用 LLM：

1. 弹幕文本经过过滤（去除纯表情、重复刷屏、过长文本）
2. 将弹幕内容 + 商品上下文 + 主播人设 Prompt 送入本地 LLM
3. LLM 生成回复文本（限制 20-50 字，适合口播）
4. 回复送入 TTS 合成

**防刷屏机制：**

- 同一用户 10 秒内最多回复 1 次
- 同类问题（关键词相同）30 秒内去重
- 优先回复包含商品关键词的弹幕
- 每分钟最多回复 8 条（`max_reply_per_minute`）

### 3.5 Module 4: MediaPipe（媒体合成管道）

**职责**：将音频 + 数字人照片 + 字幕合成为连续的视频帧序列。

**口型动画 MVP 方案（成本最低）：**

不使用 Wav2Lip（需 GPU + 大模型），采用 **音频特征驱动 + 预置口型等级**：

```
音频 PCM 数据 → 分帧计算音量/频谱特征 → 映射到 5 个口型等级
                                                    ↓
                                        [闭嘴] [微张] [半张] [大张] [O型]
                                                    ↓
                                        根据等级合成口型遮罩 → 叠加到原图
```

用户需在配置中标注 5 个口型对应的嘴部区域坐标，运行时根据音频特征选择对应口型进行叠加。

**照片微动效果（解决死板问题）：**

- **呼吸动画**：整体缩放 1.0 → 1.02 → 1.0，周期 3-4 秒
- **眨眼动画**：间隔 3-6 秒随机触发，单次 150ms，使用预置的闭眼图片
- **头部微动**：正弦波偏移，幅度 2-3 像素，周期 5-8 秒

**字幕渲染：**

- 当前口播文本以字幕形式叠加在画面底部
- 支持字体、颜色、背景透明度配置
- 长文本自动换行，最多显示 2 行

**帧合成输出：**

```swift
struct FrameComposer {
    func composeFrame(audioSample: AudioSample, text: String) -> CVPixelBuffer
    // 输出 25fps 的 CVPixelBuffer，供 FFmpeg 编码
}
```

### 3.6 Module 5: Streamer（推流器）

**职责**：将视频帧编码为 H.264、音频编码为 AAC，通过 RTMP 推送到快手服务器。

**配置结构：**

```swift
struct StreamConfig {
    var rtmpUrl: String            // 服务器地址
    var streamKey: String          // 流密钥
    var videoBitrate: Int = 2000   // kbps
    var audioBitrate: Int = 128    // kbps
    var fps: Int = 25
    var resolution: CGSize = CGSize(width: 1080, height: 1920)  // 竖屏 9:16
}
```

**实现方式：**

- Swift 通过 `Process` 启动嵌入的 FFmpeg 子进程
- 视频帧通过 `pipe` 喂给 FFmpeg 的 rawvideo 输入
- 音频通过 `pipe` 喂给 FFmpeg 的 s16le 输入
- FFmpeg 进行 H.264 + AAC 编码，RTMP 封装后推流

**断线重连：**

- 监控 FFmpeg 进程状态和退出码
- 网络异常时 3 秒内自动重试，最多 5 次
- 超过 5 次则自动暂停直播，弹窗提示用户检查网络或推流地址

**快手推流地址获取流程：**

1. 用户在快手 APP 创建直播间
2. 选择"电脑直播"或"第三方推流"
3. 快手返回 RTMP 服务器地址 + 流密钥
4. 用户将地址填入本软件，点击开播

### 3.7 Module 6: ChatOCR（弹幕识别）

**职责**：定时截取快手直播后台页面，OCR 识别弹幕文字。

**工作流程：**

```
用户用浏览器打开快手直播后台（直播中控台）
                          ↓
本软件让用户框选弹幕区域（首次配置，保存坐标到 config）
                          ↓
定时截图（每秒 1-2 次）→ macOS Vision OCR 识别文字
                          ↓
弹幕去重（对比上一帧，只提取新增内容）
                          ↓
解析出发送者昵称 + 弹幕内容 → 送入 ReplyEng
```

**OCR 实现（macOS Vision 框架）：**

```swift
import Vision

func recognizeText(from image: CGImage, completion: @escaping ([String]) -> Void) {
    let request = VNRecognizeTextRequest { request, error in
        guard let observations = request.results as? [VNRecognizedTextObservation] else { return }
        let texts = observations.compactMap { $0.topCandidates(1).first?.string }
        completion(texts)
    }
    request.recognitionLevel = .fast
    request.recognitionLanguages = ["zh-Hans"]

    let handler = VNImageRequestHandler(cgImage: image)
    try? handler.perform([request])
}
```

**去重策略：**

- **文本相似度去重**：编辑距离 < 3 认为是同一条弹幕
- **时间窗口去重**：3 秒内相同内容只算一条
- **系统消息过滤**：可配置是否处理 "xxx 进入直播间" 等系统提示

---

## 4. 数据模型与配置

### 4.1 场景配置文件（JSON）

```json
{
  "version": "1.0",
  "scene": {
    "id": "scene_001",
    "name": "早餐面包带货",
    "type": "product",
    "avatar": {
      "image_path": "avatars/host_01.jpg",
      "mouth_regions": {
        "mouth_closed": [120, 280, 160, 300],
        "mouth_slight": [118, 278, 164, 302],
        "mouth_half": [116, 276, 168, 306],
        "mouth_open": [114, 274, 172, 310],
        "mouth_o": [118, 278, 168, 308]
      },
      "blink_image": "avatars/host_01_blink.jpg"
    },
    "voice": {
      "engine": "edge_tts",
      "voice_name": "zh-CN-XiaoxiaoNeural",
      "speed": 1.15,
      "pitch": 0
    },
    "background": {
      "type": "image",
      "path": "backgrounds/kitchen.jpg"
    },
    "script": [
      {
        "id": "seg_01",
        "type": "intro",
        "content": "家人们早上好！今天给大家推荐一款超好吃的全麦面包！",
        "duration": 8,
        "transition": "fade_in"
      },
      {
        "id": "seg_02",
        "type": "product_intro",
        "content": "这款面包用的是100%全麦粉，无添加蔗糖，特别适合减脂期的姐妹！",
        "duration": 12,
        "product_id": "p_001"
      },
      {
        "id": "seg_03",
        "type": "loop",
        "content": "喜欢的家人们点击右下角小黄车，今天活动价只要29块9，还包邮！",
        "duration": 10,
        "repeat_interval": 60
      }
    ],
    "reply_rules": [
      {
        "id": "rule_01",
        "priority": 1,
        "keywords": ["价格", "多少钱", "怎么卖"],
        "response": "今天活动价29块9一箱，平时要49的！",
        "cooldown_seconds": 30
      },
      {
        "id": "rule_02",
        "priority": 1,
        "keywords": ["热量", "卡路里", "胖"],
        "response": "这款热量超低，一片只有80大卡，减脂期也能吃！",
        "cooldown_seconds": 20
      },
      {
        "id": "rule_03",
        "priority": 2,
        "keywords": ["包邮", "运费"],
        "response": "全场包邮，48小时内发货！",
        "cooldown_seconds": 30
      },
      {
        "id": "rule_fallback",
        "priority": 99,
        "keywords": ["*"],
        "use_llm": true,
        "llm_prompt": "你是一位热情的带货主播，正在卖全麦面包。用简短口语化方式回复这条弹幕，控制在30字以内。弹幕：{message}",
        "cooldown_seconds": 5
      }
    ],
    "settings": {
      "reply_enabled": true,
      "reply_delay_range": [1, 3],
      "auto_thank_gift": true,
      "auto_welcome": true,
      "max_reply_per_minute": 8
    }
  }
}
```

### 4.2 全局应用配置

```json
{
  "app": {
    "stream": {
      "default_bitrate": 2000,
      "default_fps": 25,
      "resolution": "1080x1920"
    },
    "ocr": {
      "capture_interval_ms": 800,
      "screen_region": {
        "x": 100,
        "y": 400,
        "width": 400,
        "height": 500
      }
    },
    "llm": {
      "engine": "ollama",
      "model": "qwen2.5:7b",
      "api_url": "http://localhost:11434",
      "max_tokens": 80,
      "temperature": 0.8
    },
    "tts": {
      "engine": "edge_tts",
      "cache_dir": "~/.digital_human/tts_cache"
    }
  }
}
```

### 4.3 目录结构

**App Bundle 内部：**

```
DigitalHumanLive.app/
├── Contents/
│   ├── MacOS/
│   │   └── DigitalHumanLive          # Swift 主程序
│   ├── Frameworks/
│   │   ├── Python3.11/
│   │   │   └── ...                   # 嵌入的 Python 运行时
│   │   └── FFmpeg/
│   │       └── ffmpeg                # 嵌入的 FFmpeg 可执行文件
│   └── Resources/
│       ├── Assets.xcassets/
│       └── python_scripts/           # Python 脚本
│           ├── tts_engine.py
│           ├── llm_client.py
│           └── audio_analyzer.py
```

**用户数据目录（`~/.digital_human/`）：**

```
~/.digital_human/
├── scenes/                           # 场景配置 JSON
│   ├── scene_001.json
│   └── scene_002.json
├── avatars/                          # 数字人图片
│   ├── host_01.jpg
│   └── host_01_blink.jpg
├── backgrounds/                      # 背景图
├── tts_cache/                        # TTS 缓存（避免重复生成）
│   └── [hash].mp3
├── logs/                             # 运行日志
│   └── 2026-05-22.log
└── config.json                       # 全局配置
```

---

## 5. 技术选型

| 组件 | 选型 | 理由 |
|------|------|------|
| UI 框架 | SwiftUI | macOS 原生，前端开发者可快速上手 |
| Python 嵌入 | 内嵌 Python 3.11 + 依赖 | 同 App Bundle 分发，用户零配置 |
| Swift-Python 通信 | Process stdin/stdout JSON | 简单可靠，无需复杂桥接 |
| TTS | Edge-TTS | 免费、无需 API Key、中文效果好 |
| LLM | Ollama + qwen2.5:7b | 免费离线、Mac M 系列优化、中文优秀 |
| OCR | macOS Vision 框架 | 原生支持、无需额外模型、实时性好 |
| 口型驱动 | 音频特征 + 预置口型等级 | 零 GPU 依赖，所有 Mac 都能跑 |
| 推流 | FFmpeg RTMP | 成熟稳定、编码效率高 |
| 音频分析 | Python librosa / numpy | 提取音量/频谱特征驱动口型 |

---

## 6. 错误处理与边界情况

### 6.1 故障场景应对表

| 故障场景 | 影响 | 应对策略 |
|---------|------|---------|
| 网络断开 | 推流中断 | 3秒内自动重连，最多5次；超过则暂停直播，保存状态 |
| 快手 RTMP 地址过期 | 推流被拒绝 | 检测 FFmpeg 错误码，提示用户重新获取推流地址 |
| TTS 生成失败 | 当前段落无声 | 跳过该段落，记录日志，继续播下一段 |
| LLM 回复超时 | 弹幕无回复 | 降级到关键词规则回复；若规则也没有，静默跳过 |
| OCR 识别不到文字 | 错过弹幕 | 正常跳过，下一帧继续；连续10帧无结果提示检查框选区域 |
| CPU/内存过高 | 卡顿、掉帧 | 动态降码率、降帧率；提示关闭其他应用 |
| 场景切换时弹幕涌入 | 回复积压 | 队列最多保留5条待回复，超限则丢弃最早的 |
| 用户发送恶意长文本 | 占用资源 | 弹幕截断（最多50字），超长直接丢弃 |
| Mac 睡眠/锁屏 | 直播中断 | 开播时阻止系统睡眠（`IOPMAssertion`） |

### 6.2 直播状态机

```
         ┌─────────┐
         │  IDLE   │  ← 空闲，未加载场景
         └────┬────┘
              │ 用户选择场景
              ▼
         ┌─────────┐
         │ LOADED  │  ← 场景已加载，可以预览
         └────┬────┘
              │ 点击开播
              ▼
         ┌─────────┐     推流失败     ┌─────────┐
         │CONNECTING│ ─────────────► │  ERROR  │
         └────┬────┘                  └────┬────┘
              │ 连接成功                   │ 可恢复错误：返回 CONNECTING
              ▼                            │ 不可恢复：返回 IDLE
         ┌─────────┐                      └─────────┘
         │  LIVE   │  ← 正常直播中
         │(PLAYING)│
         └────┬────┘
              │ 点击停播 / 推流断开
              ▼
         ┌─────────┐
         │ STOPPED │
         └────┬────┘
              │ 回到 IDLE
              ▼
         ┌─────────┐
         │  IDLE   │
         └─────────┘
```

### 6.3 资源不足时的降级策略

- **CPU > 80%** → 降视频码率（2000 → 1500 → 1000 kbps）
- **内存 > 80%** → 清空 TTS 缓存，释放历史帧缓冲区
- **磁盘 < 1GB** → 停止日志写入，仅保留错误日志
- **温度 > 90°C** → 降帧率（25 → 20 → 15 fps）

---

## 7. 里程碑规划

| 阶段 | 目标 | 预计周期 |
|------|------|---------|
| **MVP v0.1** | 单场景直播、预设脚本轮播、基础口型、FFmpeg 推流 | 3-4 周 |
| **v0.2** | 多场景管理、弹幕 OCR、关键词自动回复 | 2-3 周 |
| **v0.3** | LLM 智能回复、TTS 音色克隆、字幕样式 | 2-3 周 |
| **v0.4** | 抖音/小红书平台适配、数据分析看板 | 3-4 周 |
| **v1.0** | 3D 形象支持、多路直播、商业化发布 | 待定 |

---

## 8. 风险与限制

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 快手调整推流协议 | 推流功能失效 | 关注快手开放平台公告，预留协议适配接口 |
| 平台限制非真人直播 | 账号风控 | 使用真人照片、控制回复频率模拟真人行为 |
| OCR 识别准确率不足 | 错过弹幕 | 允许用户手动校正区域，支持多区域识别 |
| 口型效果不够自然 | 观感差 | MVP 先跑通流程，后续迭代 Wav2Lip 方案 |
| Mac Intel 机型性能不足 | 卡顿 | 提供最低配置检测，M 系列芯片优先推荐 |
