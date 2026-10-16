# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**SmartLightAsrPro**：智能家居 AI 开灯产品的语音识别固件，运行在 ASR-PRO 语音识别开发板（好好搭搭 / haohaodada「天问Block」平台，`platform:haohaodada`）。整个项目是一个文件：`SmartLight-ASRPRO.hd`。

`.hd` 文件是天问Block 图形化 IDE 生成并回读的 C 源码。本仓库**没有构建系统**，编译与烧录都在天问Block IDE / 好好搭搭 SDK 工具链中完成。

## 与主控项目的分工（双 MCU）

本产品由两块 MCU 协作，本仓库只做「听 + 说」：

- **ASR-PRO（本仓库）**：语音识别 → 经 UART 发指令给 ESP-12F → 收响应 → 喇叭播报。
- **ESP-12F（仓库 `../SmartLight`）**：采集光照/测距、驱动舵机拨开关、维护灯状态、回传结果。

二者唯一耦合点是 UART 协议（9600 8N1，文本行帧，以换行结尾），见本仓库 README 与 `../SmartLight/doc/软件设计.md` 第 6 节。改协议必须两边同步。

## Critical：`//{...}` 注释块是 DSL，不是注释

IDE 解析这些块来生成语音/命令表，编辑它们才能改变行为；删除或破坏语法会静默改变固件。

- `//{speak:思思-知性女声,vol:10,speed:10,platform:haohaodada}` — TTS 音色。
- `//{playid:10003,voice:已打开灯光}` — 一条播报语音（TTS 文本），由 `play_audio(10003)` 播放。
- `//{ID:1,keyword:"命令词",ASR:"打开灯光",ASRTO:""}` — 命令词定义：`ID` 即识别回调里的 `snid`，`ASR` 为要匹配的口语。

文件末尾的 `/** edittype="asr_procode" */` 是 IDE 标记，需保留。include 的头文件由 SDK 提供，不在本仓库。

## 架构（生命周期）

SDK 调用三个固定入口：

- `setup()` — 操作系统启动前初始化；命令词定义（`//{ID:...}` DSL）在此。
- `hardware_init()` — 操作系统启动后初始化：串口（9600）、音量 `vol_set(7)`，随后 `vTaskDelete(NULL)` 结束初始化任务。
- `ASR_CODE()` — 语音识别回调，每次识别到命令词时被调用；读取全局 `snid` 分发。`set_state_enter_wakeup(...)` 必须在此调用才有效。

## 当前行为

- 串口 `Serial` 9600 接 ESP-12F：TX=PB5(13)、RX=PB6(14)。
- 命令 ID：1「打开灯光」/3「开灯」→ 发 `ON`；2「关闭灯光」/4「关灯」→ 发 `OFF`。
- 发送后带 2s 超时读一行响应，按响应播报（`ON_OK`/`OFF_OK`/`ERROR`，见 README）。
- 唤醒词「智能灯光」；`set_state_enter_wakeup(20000)` 使 20 秒无命令后休眠，唤醒后重新说唤醒词。