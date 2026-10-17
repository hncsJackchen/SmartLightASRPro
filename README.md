# SmartLightAsrPro

智能家居 AI 开灯产品的**语音识别固件**，运行在 ASR-PRO 开发板（好好搭搭 / haohaodada「天问Block」平台），配合 ESP-12F 主控（见 [`../SmartLight`](../SmartLight)）实现语音开关灯。

## 工作原理（双 MCU）

ASR-PRO 只做「听 → 说」：识别到命令词后经 UART 发指令给 ESP-12F，ESP-12F 执行舵机拨开关，再把结果回传，ASR-PRO 用喇叭播报。舵机、光照、测距、灯状态全部在 ESP-12F 侧，本固件不直接控制 GPIO。

## 命令词

| 命令词 | ID (`snid`) | 发送帧 |
|--------|:---:|------|
| 「智能灯光」 | 0 | 唤醒词 |
| 「打开灯光」 | 1 | `ON\n` |
| 「关闭灯光」 | 2 | `OFF\n` |
| 「开灯」 | 3 | `ON\n` |
| 「关灯」 | 4 | `OFF\n` |
| 「退出」 | 5 | —（本地休眠） |

## UART 协议（与 ESP-12F 的接口契约）

物理层：UART 9600 8N1，文本帧，以换行结尾（兼容 `\n`/`\r\n`）。

ASR-PRO → ESP-12F：

| 帧 | 含义 |
|----|------|
| `ON\n` | 打开灯光 |
| `OFF\n` | 关闭灯光 |

ESP-12F → ASR-PRO（响应）及播报语音：

| 帧 | 含义 | 播报语 |
|----|------|------|
| `ON_OK\n` | 已执行开灯 | 已打开灯光 |
| `OFF_OK\n` | 已执行关灯 | 已关闭灯光 |
| `ERROR\n` | 执行出错 | 操作失败 |

> ASR-PRO 仅在主动发送指令后才等待一条对应响应；超过 2s 无响应按 `ERROR` 播报「操作失败」。

## 硬件接线

| ASR-PRO | ESP-12F | 说明 |
|---------|---------|------|
| TX（PB5，引脚 13） | RX（GPIO3，D9） | ASR → ESP |
| RX（PB6，引脚 14） | TX（GPIO1，D10） | ESP → ASR |
| GND | GND | 共地 |

> ASR-PRO 供电与 ESP-12F 同为 3.3V，共地即可。

## 文件说明

- [SmartLight-ASRPRO.hd](SmartLight-ASRPRO.hd) — 核心固件（单文件，天问Block 项目）。
- [CLAUDE.md](CLAUDE.md) — 本仓库的 Claude Code 使用说明。

## 构建与烧录

本仓库**不包含构建系统**，编译与烧录都在天问Block IDE / 好好搭搭 SDK 工具链中完成：

1. 用天问Block IDE 打开 `SmartLight-ASRPRO.hd`。
2. 在 IDE 中编译并烧录到 ASR-PRO 开发板。

## 重要提示

- 源码中的 `//{...}` 注释块是 IDE 解析的 **DSL**（用于生成语音/命令表），编辑它们才能改命令词/播报语；删除或破坏语法会静默改变固件。
- 文件末尾的 `/** edittype="asr_procode" */` 是 IDE 标记，须保留。
- 协议改动需与 [`../SmartLight/src/config.h`](../SmartLight/src/config.h) 及 [`../SmartLight/doc/软件设计.md`](../SmartLight/doc/软件设计.md) 第 6 节同步。