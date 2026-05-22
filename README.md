# DigitalHumanLive

一款 macOS 原生数字人直播系统，支持根据预设场景内容进行自动化直播，自动回复观众弹幕/评论。

## 功能特性

- **多场景配置**：支持带货、聊天、知识分享等多种直播场景
- **数字人直播**：静态照片 + AI 口型驱动，生成实时视频流
- **自动回复**：关键词规则 + LLM 智能回复观众弹幕
- **快手推流**：通过 RTMP 协议直接推流到快手直播间
- **纯本地运行**：零服务器成本，全部 AI 能力本地计算

## 技术栈

| 组件 | 选型 |
|------|------|
| UI | SwiftUI |
| 口型驱动 | 音频特征 + 预置口型等级 |
| TTS | Edge-TTS |
| LLM | Ollama + qwen2.5:7b |
| OCR | macOS Vision 框架 |
| 推流 | FFmpeg RTMP |

## 项目状态

目前处于设计阶段，详见 [设计文档](docs/superpowers/specs/2026-05-22-digital-human-live-design.md)。

## 许可证

MIT
