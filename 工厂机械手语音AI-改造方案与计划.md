# 工厂机械手语音 AI 改造方案与实施计划

| 项 | 内容 |
|---|---|
| 项目 | 基于开源 nanobot 改造为「语音对话 + 机械手操作」工厂桌面 AI |
| 版本 | v1.0 |
| 日期 | 2026-07-03 |
| 状态 | 方案已锁定,待实施 |
| 交付物 | 一个 Windows exe(内嵌 WebUI,零端口,工厂 EDR 友好) |

---

## 一、项目目标

把 nanobot(轻量 AI agent 框架,v0.2.2)改造为工厂专用的「语音对话 + 机械手操作」桌面 AI 系统,交付一个双击即用的 Windows exe。操作工用语音连续对话下达指令,AI 理解后调用机械手动作并语音播报结果。

核心能力:
- **语音连续对话**:不用按键,持续听、自动断句、自动回复
- **机械手操作**:LLM 调用串口/Modbus tool 控制机械手
- **零端口桌面 app**:pywebview 内嵌,绕开工厂 EDR 对端口监听的拦截
- **云端智能**:LLM/STT/TTS 全走云(工控机无独显,不跑本地模型)

---

## 二、核心约束(工厂环境)

| 约束 | 影响 |
|---|---|
| IT 管制严,EDR/杀毒拦端口监听、未知 exe | → 必须零端口(pywebview),不能开 8765 走浏览器 |
| 网络白名单,能联公网云 API(已确认) | → LLM/STT/TTS 走云成立,但要确认放行的具体域名 |
| 工控机普通配置 / 无独显 | → 不能本地跑模型,LLM 必须云端 |
| 操作工非技术人员 | → 傻瓜式 UI、大按钮、语音确认、实体急停 |
| 机械手是生产设备,误操作=事故 | → 高危动作必须确认门 + 限位 + 急停(物理安全最高优先级) |
| 生产岗位,不能频繁宕机 | → 崩溃自重启、看门狗 |

---

## 三、最终技术选型(锁定)

| 维度 | 选定 | 理由 |
|---|---|---|
| 界面 | WebUI(React/TS) | 现成,语音录制组件已就绪 |
| **进程模型** | **Python SDK + pywebview 内嵌** | 零端口,绕开工厂 EDR |
| 通信 | JS↔Python 桥(`window.pywebview.api`) | 替代 WebSocket,进程内调用 |
| LLM | 云端 API | 工控机不跑模型 |
| STT | 云端(groq / siliconflow) | 现有三层链路复用 |
| TTS | 云端(Edge-TTS / 讯飞) | 新增,照抄 STT 三层结构 |
| 机械手 | 串口 / Modbus tool(pyserial / pymodbus) | 待协议文档 |
| 语音交互 | 连续对话,半双工 VAD(Level 1) | 机械手指令式场景够用且更安全 |
| 打包 | PyInstaller + WebView2 运行时 | 带数据文件,入口用 SDK |

---

## 四、整体架构(零端口)

```
双击 nanobot.exe
  │
  └─ pywebview 启动内嵌窗口(加载 WebUI 静态 HTML,不开任何端口)
       │
       ├─ 前端 React WebUI(几乎不动)
       │     传输层 nanobot-client.ts:
       │       旧: new WebSocket("ws://127.0.0.1:8765")
       │       新: window.pywebview.api.send_message(...)   ← 进程内调用
       │
       └─ Python 侧:Api 类桥接到 SDK + 语音 + 机械手
             class RobotApi:
                 self.bot = Nanobot.from_config()       # 不开端口,不走 gateway
                 send_message(text)        → bot.run / run_streamed
                 transcribe(audio_data_url)→ audio/transcription 复用
                 speak(text)               → TTS(新增)
                 move_arm / gripper / ...  → 机械手 tool
             webview.create_window(html=..., js_api=RobotApi())
```

**关键点**:Python SDK 的 `Nanobot` 类(`nanobot/nanobot.py`)本身就是「不开端口、不走 gateway」的进程内入口,通过 `agent_loop.process_direct()` 直接跑 agent。pywebview 桥正好把它暴露给 WebUI,实现零端口桌面 app。

---

## 五、子系统方案

### 5.1 精简瘦身(阶段 0)

删除无用 chat channel,大幅降低 exe 体积和复杂度。

- **删除 14 个 channel**:`telegram / discord / slack / feishu / matrix / whatsapp / qq / weixin / weecom / dingtalk / email / mochat / msteams / signal / napcat`
- **保留**:`websocket` channel 可保留作调试(生产用 pywebview 桥,不再需要它;可一并删除)
- **删除 gateway service/runtime**(`nanobot/gateway/service.py` 的 systemd/launchd 安装器):工厂不走系统服务托管
- **清理 `pyproject.toml`**:`python-telegram-bot / discord.py / slack-sdk / lark-oapi / qq-botpy / dingtalk-stream / matrix-nio / neonize` 等 20+ 个 SDK 依赖
- **验证**:`python -c "from nanobot.nanobot import Nanobot"` 能 import;`Nanobot.from_config()` 能初始化

### 5.2 语音 STT(已有,复用)

nanobot 已有三层 STT 架构,直接复用:

| 层 | 文件 | 职责 |
|---|---|---|
| 注册表 | `nanobot/audio/transcription_registry.py` | 7 家 ASR 元数据(groq/openai/siliconflow/xiaomi_mimo/stepfun/assemblyai/openrouter) |
| 编排 | `nanobot/audio/transcription.py` | 配置解析、校验、临时文件、调度,**不碰 HTTP** |
| HTTP adapter | `nanobot/providers/transcription.py` | 各家 ASR 的 HTTP 调用 |

adapter 接口:`async transcribe(file_path) -> str`。工厂中文场景优先 `siliconflow`(SenseVoiceSmall)或 `groq`(whisper-large-v3),取决于工厂白名单放行哪家。

### 5.3 语音 TTS(新增,照抄 STT 结构)

项目当前**无 TTS 模块、无音频播放库**,需新增。照抄 STT 的三层结构:

```
nanobot/audio/tts_registry.py          # provider 元数据(edge_tts/azure/xfyun)
nanobot/audio/tts.py                   # 编排:配置解析、调用、返回音频 bytes
nanobot/providers/tts.py               # adapter:各家 HTTP 调用
```

adapter 接口:`async synthesize(text) -> bytes`(返回 mp3/wav bytes)。起步推荐 **Edge-TTS**(免费、中文好);商业场景可换讯飞/Azure。

音频播放在前端:pywebview 桥返回音频 bytes → 前端 `Audio` 元素播放(不走 HTTP)。

### 5.4 连续对话(半双工 VAD,Level 1)

当前 STT 是「手动整段录音」(`MediaRecorder` 一次性 blob + 整段发 ASR)。要改成连续对话,核心是加 **VAD 自动断句 + 会话状态机**。

**状态机循环:**

```
① 持续监听(开麦 + VAD 跑着)
    │ 检测到语音能量超阈值
    ▼
② 录音中(持续累积音频)
    │ 检测到尾部静音 ~700ms
    ▼
③ 转写中 → 后端 STT(现有链路复用)
    │ 拿到文字
    ▼
④ 思考中 → LLM + 机械手 tool 执行
    │ 产出回复
    ▼
⑤ 播报中 → TTS(麦克风静音,避免自激)
    │ 播报结束
    └─→ 回到 ①
```

**要补的三块:**
1. **VAD 自动断句**:复用 `useVoiceRecorder.ts` 现有 RMS 音量检测(`voiceLevelFromSamples`),改成「检测到语音开始 → 攒音频;尾部静音 ~700ms → 截断送 ASR」。能量阈值起步即可,不必上模型。
2. **会话状态机**:新写一个 `ConversationSession` 控制器,管 ①→⑤ 状态切换、防重入。
3. **半双工抑制**:TTS 播报期间让麦克风「失聪」,否则 AI 听到自己的语音答复形成自言自语死循环。

**两个硬坑:**
- **回声消除(AEC)**:当前 `getUserMedia({ audio: true })` 没开,必须改成
  `getUserMedia({ audio: { echoCancellation: true, noiseSuppression: true, autoGainControl: true } })`,否则车间噪声/回声严重干扰 VAD。
- **VAD 灵敏度可调**:车间环境噪声大,固定阈值会误触发,WebUI 设置里暴露灵敏度滑块。

**后端零改动**:连续对话只是前端把整段录音切成多段话语逐个发,`audio/transcription.py` 的 `transcribe_audio_data_url` 链路原样复用。

### 5.5 机械手 tool + 安全(阶段 1)

新建 `nanobot/agent/tools/robot_arm.py`,封装串口/Modbus,暴露给 LLM:
- `status()` — 只读,查当前位置/状态(默认安全)
- `move_joints(joints)` / `move_xyz(x,y,z)` — 危险,需确认
- `gripper_open()` / `gripper_close()` — 危险,需确认
- `home()` — 回零,需确认
- `estop()` — 软件急停(第二道防线)

**安全护栏(不可省):**
- **高危动作确认门**:连续对话 + 自动执行剥夺了打字模式的「回车前审视」机会,状态机里高危动作要先在 WebUI 显示「即将执行:xxx」0.5–1s,需语音/点击确认;只读查询可直接执行。
- **速度/工作空间限位**:tool 内部硬限制,LLM 给的越界指令直接拒。
- **实体急停按钮**:软件急停不可靠,机械手必须配独立硬件急停。
- **白名单化**:高危动作库白名单,不在白名单一律要确认。

### 5.6 打包 PyInstaller(阶段 3)

- **入口**:新写 `main.py`,启动 pywebview 窗口 + RobotApi(不调 `_run_gateway`)
- **数据文件**:`--add-data` 打入 `nanobot/web/dist`(WebUI)、`nanobot/templates/`、`nanobot/skills/`
- **hidden imports**:动态发现的 provider/tool 需显式声明;重点处理 `tiktoken`(Rust 扩展)、`lxml`、`mcp`
- **WebView2 运行时**:老款工控机(Win10 LTSC/精简版)可能没预装,exe 要自带运行时(+几十 MB)
- **体积预估**:200MB+

---

## 六、分阶段实施计划

| 阶段 | 内容 | 依赖机械手协议? | 工作量 | 验证标准 |
|---|---|---|---|---|
| **0. 精简** | 删 14 个 chat channel + 清 `pyproject.toml` + 删 gateway service | 否 | 0.5–1 天 | SDK 能 import + 初始化;`Nanobot.from_config()` 不报错 |
| **1. 机械手 tool** | `robot_arm.py`(串口/Modbus)+ 安全护栏 | **是** | 2–4 天 | 对话能查状态、发受限动作、越界被拒、高危要确认 |
| **2. 语音 + pywebview 桥** | STT 复用 + TTS 新增 + 连续对话状态机 + WebUI 传输层改桥 + Python `Api` 类 | 否 | 3–5 天 | 语音输入→对话→TTS 播报闭环;连续对话循环跑通 |
| **3. 打包** | PyInstaller + pywebview + WebView2 + 数据文件,入口换 SDK | 否 | 1–3 天 | 干净 Windows 机双击 exe → 内嵌窗口 → 控机械手 |
| **4. 打磨** | 崩溃自重启、灵敏度调参、急停硬件接驳、延迟优化 | — | 按需 | — |

**原型合计约 1.5–2.5 周。** 阶段 0/2/3 不依赖机械手,可先做;阶段 1 等协议文档。

**最大新工作量在阶段 2 的 pywebview 桥**:把 `webui/src/lib/nanobot-client.ts` 的 WebSocket 传输换成 `window.pywebview.api.*`,Python 侧写 `Api` 类桥接 SDK + STT + TTS + 机械手。

---

## 七、工厂专项清单

| 项 | 要求 |
|---|---|
| WebView2 运行时 | 老 Win10 可能没预装,exe 自带 |
| 离线激活 | exe 不依赖在线登录/激活 |
| 崩溃自重启 | 看门狗/托盘,exe 挂了自动拉起 |
| 操作工友好 | 大按钮、语音确认、实体急停 |
| API key 安全 | 不硬编码;建议内网取号服务或 key 轮换 + 用量监控 |
| 物理安全 | 高危动作确认门 + 限位 + 实体急停(最高优先级) |

---

## 八、风险与待确认项

| 项 | 状态 | 应对 |
|---|---|---|
| **工厂白名单具体放行哪些域名** | ⚠️ 待确认 | 选 LLM/STT/TTS provider 前先问 IT;直接决定能用哪家云 |
| **API key 嵌 exe 泄露风险** | ⚠️ 待设计 | 内网取号服务 / key 轮换 / 用量监控 |
| **机械手协议文档** | ⚠️ 待提供 | 阶段 1 阻塞项;型号 + 串口波特率/Modbus 寄存器表 |
| **工厂网络延迟** | ⚠️ 待实测 | 云 ASR/LLM 一来回约 0.5–2s,影响对话响应感 |
| **WebView2 是否预装** | ⚠️ 待确认 | 工控机 Win 版本;无则自带运行时 |
| **生产数据上云合规** | ⚠️ 待确认 | 即使技术能联,语音指令上云是否合规(用户已确认能联) |

---

## 九、关键文件清单

### 要新建
| 文件 | 用途 |
|---|---|
| `main.py`(项目根) | exe 入口,启动 pywebview + RobotApi |
| `nanobot/audio/tts_registry.py` | TTS provider 注册表(照抄 STT) |
| `nanobot/audio/tts.py` | TTS 编排 |
| `nanobot/providers/tts.py` | TTS HTTP adapter |
| `nanobot/agent/tools/robot_arm.py` | 机械手串口/Modbus tool + 安全护栏 |
| `webui/src/hooks/useConversationSession.ts` | 连续对话状态机 |
| `robot_arm.spec.py`(PyInstaller) | 打包配置 |

### 要改
| 文件 | 改动 |
|---|---|
| `webui/src/lib/nanobot-client.ts` | 传输层:WebSocket → `window.pywebview.api.*` |
| `webui/src/hooks/useVoiceRecorder.ts` | 加 AEC constraint + VAD 自动断句 + 灵敏度参数 |
| `pyproject.toml` | 删 20+ channel SDK 依赖,加 pywebview/pyserial/pymodbus/edge-tts |

### 要删
| 文件 | 原因 |
|---|---|
| `nanobot/channels/{telegram,discord,slack,feishu,matrix,whatsapp,qq,weixin,wecom,dingtalk,email,mochat,msteams,signal,napcat}.py` | 工厂不需要这些 chat 平台 |
| `nanobot/gateway/service.py` 的系统服务安装部分 | 工厂不走 systemd/launchd 托管 |

---

## 十、技术选型决策记录(为什么这么选)

1. **为什么零端口(pywebview)而不是 gateway+浏览器**:工厂 EDR/杀毒会拦「进程监听端口」行为,系统浏览器访问 127.0.0.1 也可能触发授权弹窗。pywebview 内嵌 + 进程内 JS↔Python 桥完全绕开网络栈,工厂 EDR 友好。
2. **为什么云端而不是本地模型**:工控机普通配置/无独显,本地跑 7B 模型不现实;工厂已确认能稳定联云。云端是唯一可行。
3. **为什么半双工连续对话而不是全双工**:机械手指令式场景,用户发指令、机械手执行,本就不该同时说话。半双工更安全(AI 播报/执行时用户不插嘴造成混乱),且实现成本远低于全双工(全双工需 AEC + 流式 ASR)。
4. **为什么保留 WebUI 而不是重写桌面 GUI**:语音录制、对话、markdown 渲染等组件已就绪,重写 PyQt/Tkinter 等于推倒重来。pywebview 保留 WebUI,只换传输层。
5. **为什么不走 gateway**:gateway 在本项目里就是「启动所有子系统的主进程」,但它的 WebUI 依赖 WebSocket 端口。pywebview 方案用 SDK 进程内调用替代了 gateway 的端口通信,所以不再需要 gateway。

---

*本文档随实施进度更新。每阶段完成后补充实际遇到的问题与偏差。*



# 实施计划:工厂机械手语音 AI 改造 —— 阶段 0(精简瘦身)

## Context(为什么做这个)

用户要把开源 nanobot 改造成「工厂机械手语音对话 + 操作」桌面 AI,交付零端口 Windows exe(完整方案见仓库根 `工厂机械手语音AI-改造方案与计划.md`)。改造分 4 阶段,其中:

- **阶段 1(机械手 tool)** 阻塞于用户尚未提供的机械手协议文档
- **阶段 2(语音 + pywebview 桥)、阶段 3(打包)** 是大工程,且阶段 2 需深挖前端传输层
- **阶段 0(精简瘦身)** 是唯一条件完全具备、能立刻做、风险最低,且能让后续 exe 体积和复杂度显著下降的第一步

本计划**聚焦阶段 0 的可执行步骤**;阶段 1/2/3 作为 roadmap 列出,待阻塞项解决后另行细化。

阶段 0 的目标:删除 14 个无用 chat channel + 清理对应依赖与测试,**验证精简后 SDK 仍能独立 import 和初始化**(为后续 pywebview 入口铺路)。

---

## 阶段 0 实施步骤

### Step 0 — 落档(执行首步)

将本实施计划保存到项目根 `工厂机械手语音AI-实施计划-阶段0.md`,与 `工厂机械手语音AI-改造方案与计划.md` 并列,方便团队查阅。
(当前 plan 仅存在于 plan mode 工作文件 `~/.claude/plans/sleepy-giggling-sketch.md`,不在项目内;此步将其复制到项目根。)

### Step 1 — 删除 14 个 chat channel 文件

删除 `nanobot/channels/` 下(代表性路径,模式即「文件名 = channel 名」):
`telegram.py`、`discord.py`、`slack.py`、`feishu.py`、`dingtalk.py`、`qq.py`、`matrix.py`、`msteams.py`、`wecom.py`、`weixin.py`、`whatsapp.py`、`napcat.py`、`signal.py`、`mochat.py`

**必须保留**:
- `base.py`、`manager.py`、`registry.py`、`__init__.py`(框架)
- `websocket.py` —— webui gateway 承载,`manager.py:127` 有 `name == "websocket"` 硬编码特例,删除会留死分支
- `email.py` —— 无第三方依赖,保留无害(可选删)

**机制保证**:channel 发现是 `registry.py:13-24` 的 `pkgutil.iter_modules` 扫包,排除集合 `{base, manager, registry}`。**删 .py 文件 = 自动从发现列表消失,无需改任何名单。** 已由探索确认:核心 `loop.py`/`runner.py`/`nanobot.py`/`sdk/*` 零 import 任何具体 channel。

### Step 2 — 清理 `pyproject.toml` 依赖

**顶层 `[project.dependencies]` 删行**(行号供定位,以实际内容为准):
`dingtalk-stream`、`python-telegram-bot`、`lark-oapi`、`socksio`、`slack-sdk`、`slackify-markdown`、`qq-botpy`、`python-socks`

**`[project.optional-dependencies]` 删整组**:`wecom`、`weixin`、`msteams`、`matrix`、`discord`、`whatsapp`

**必须保留的核心依赖**:`anthropic`、`openai`、`httpx`、`pydantic`、`pydantic-settings`、`loguru`、`mcp`、`tiktoken`、`typer`、`rich`、`croniter`、`ddgs`、`readability-lxml`、`lxml-html-clean`、`jinja2`、`json-repair`、`websockets`+`websocket-client`(webui)、`python-socketio`+`msgpack`、`prompt-toolkit`+`questionary`(CLI)等。

> `aiohttp` 被 `api`/`matrix`/`qq`/`napcat` 共用 —— 顶层无此依赖(仅在 optional `api`/`matrix` 里),删 matrix/qq 后若保留 `api` 服务则不动。

**本步不加新依赖**(pywebview/pyserial/pymodbus/edge-tts 留到对应阶段再加,避免阶段 0 引入未用依赖)。

### Step 3 — 清理测试

**删** `tests/channels/` 下针对 14 个 channel 的测试(模式 `test_{channel}_*.py`,约 38 个),以及 `tests/test_msteams.py`。

**评估保留**(框架自测,`manager.py`/`websocket.py` 保留即可留):`tests/channels/test_channel_plugins.py`、`test_channel_manager_reasoning.py`、`test_channel_manager_delta_coalescing.py`、`test_base_channel.py`、`test_websocket_*.py`

**改**:`tests/agent/test_loop_direct_websocket_status.py`(把 `channel="websocket"` 标签语义问题留待阶段 2 处理,本步可暂不动)

### Step 4 — 验证(关键,逐步执行)

1. **import 检查**:`python -c "from nanobot.nanobot import Nanobot; print('import ok')"`
2. **lint**:`ruff check nanobot/`(不应有新错误;注意 gotchas:`ruff check` 可用,**不要用 `ruff format`**,会破坏 git blame)
3. **核心测试**:`pytest tests/ -x --ignore=tests/channels`(排除已删的 channel 测试目录,确认核心 agent/sdk/provider 测试仍绿)
4. **(可选,需 provider key)CLI 冒烟**:`nanobot agent -m "你好"` 走单次对话,确认 agent 核心 + provider 链路通

---

## 复用的现有机制(无需新写)

| 用途 | 现有实现 | 说明 |
|---|---|---|
| channel 自动发现 | `nanobot/channels/registry.py:13-24` | pkgutil 扫包,删文件即失活 |
| SDK 进程内入口 | `nanobot/nanobot.py:63` `Nanobot` 类 | `from_config()` + `run()`,不开端口 |
| tool 自动发现 | `nanobot/agent/tools/loader.py:30-60` | pkgutil 扫包,阶段 1 新 tool 落地即注册 |
| tool 模板 | `nanobot/agent/tools/cron.py` | `@tool_parameters` + `tool_parameters_schema` 现代风格,阶段 1 照抄 |
| STT 三层结构 | `audio/transcription_registry.py` + `audio/transcription.py` + `providers/transcription.py` | 阶段 2 TTS 照抄此结构 |

---

## 后续阶段 Roadmap(本次不实现)

**阶段 1 — 机械手 tool**(阻塞于机械手协议文档)
- 新建 `nanobot/agent/tools/robot_arm.py`,照 `cron.py` 模板;落在 tools 目录即被 `loader.py` 自动注册
- 安全护栏:高危动作确认门、速度/工作空间限位、`estop`

**阶段 2 — 语音 + pywebview 桥**
- TTS 新增:照抄 STT 三层(`audio/tts_registry.py` + `audio/tts.py` + `providers/tts.py`)
- 连续对话状态机:新建 `webui/src/hooks/useConversationSession.ts`(听→录→转写→思考→播报循环)
- pywebview 桥:**前置**先读 `webui/src/lib/nanobot-client.ts` 摸清全部接口(发消息/收流/transcribe/settings/session),逐一替换 WebSocket 为 `window.pywebview.api.*`;Python 侧写 `Api` 类桥接 `Nanobot` SDK + STT + TTS + 机械手
- 改 `webui/src/hooks/useVoiceRecorder.ts:190` 开 AEC(`echoCancellation/noiseSuppression/autoGainControl`)

**阶段 3 — 打包**
- 前置:`cd webui && bun install && bun run build`(产物到 `nanobot/web/dist/`,当前不存在)
- PyInstaller spec:`--add-data` 打入 `nanobot/web/dist`、`nanobot/templates`、`nanobot/skills`;hidden imports 处理 `tiktoken`/`lxml`/`mcp` 及动态发现的 provider/tool
- 入口 `main.py`:启动 pywebview + `Api` 类(用 SDK,不调 `_run_gateway`);带 WebView2 运行时

---

## 阻塞项(阶段 1/2/3 动手前需用户提供)

1. **工厂白名单放行哪些域名** → 决定能用哪家云 LLM/STT/TTS provider
2. **机械手型号 + 串口波特率 / Modbus 寄存器表** → 阶段 1 必需

---

## 阶段 0 验证清单(完成标准)

- [ ] 14 个 channel 文件已删,`websocket.py` 保留
- [ ] `pyproject.toml` 对应依赖行/组已删,核心依赖保留
- [ ] 对应测试已删,核心测试目录保留
- [ ] `python -c "from nanobot.nanobot import Nanobot"` 成功
- [ ] `ruff check nanobot/` 无新错误
- [ ] `pytest tests/ -x --ignore=tests/channels` 核心测试通过




# 工厂机械手语音 AI 改造 —— 经验总结与实现记录

| 项 | 内容 |
|---|---|
| 日期 | 2026-07-03 |
| 基线 | 开源 nanobot v0.2.2(Python 3.11+,React/TS WebUI) |
| 目标 | 工厂机械手「语音对话 + 操作」桌面 AI,交付零端口 Windows exe |
| 当前进度 | 阶段 0 精简 ✅ / 阶段 1 机械手(Modbus 模拟)✅ / 桌面 exe(mac)✅ / 阶段 2 语音 ⏸ 搁置 / 阶段 3 Windows 打包 ⏳ |

配套文档:
- [`工厂机械手语音AI-改造方案与计划.md`](工厂机械手语音AI-改造方案与计划.md) —— 完整方案与选型
- [`工厂机械手语音AI-实施计划-阶段0.md`](工厂机械手语音AI-实施计划-阶段0.md) —— 阶段 0 逐步执行
- 本文 —— 经验沉淀与实现记录

---

## 一、项目演进脉络

1. **分析 nanobot**:理清核心架构(Channel→MessageBus→AgentLoop→AgentRunner→Provider/Tools),定位它「轻量、边缘可扩展」的特质。
2. **转向改造**:用户要把它改成工厂机械手语音 AI 桌面 exe。
3. **约束逐步浮现**(每一条都改变了方案):
   - 工厂 EDR 拦端口监听 → 放弃 gateway+浏览器,走 **零端口 pywebview**
   - 工控机无独显 → 放弃本地模型,**LLM 走云端**
   - 工厂网络白名单 → provider 受限(对话要联云)
   - 物理设备误操作=事故 → **安全护栏**提到第一优先级

> 经验:**约束驱动的选型**。每一步都不是「最好」,而是「在工厂约束下唯一可行」。提前把约束问清楚(平台/网络/硬件/安全),能省掉大量返工。

---

## 二、关键技术决策(锁定)

| 维度 | 选择 | 理由 |
|---|---|---|
| 界面 | 保留 React WebUI | 语音/对话组件已就绪 |
| 进程模型 | **pywebview 内嵌 + Python SDK** | 零端口,绕开工厂 EDR 对「进程监听端口」的拦截 |
| 通信 | JS↔Python 桥(`window.pywebview.api`) | 替代 WebSocket,进程内调用 |
| LLM | 云端(智谱 glm-4.5-air) | 工控机跑不动本地模型;工厂确认能联云 |
| STT | 复用现有(`audio/transcription*`) | 项目已有三层 STT,7 家 ASR |
| TTS | 新增(照抄 STT 三层)| 项目无 TTS(搁置) |
| 机械手 | Modbus,**先模拟** | 不阻塞于真实硬件/协议文档 |
| 语音交互 | 连续对话半双工 VAD(搁置)| 指令式场景够用且更安全 |
| 打包 | PyInstaller | 标准 Python 打包 |

---

## 三、各阶段实现

### 阶段 0:精简瘦身 ✅

**做了什么**:删 14 个 chat channel + 清 pyproject 依赖 + 清测试。

**怎么实现**:
- channel 是 [`registry.py`](nanobot/channels/registry.py) 的 `pkgutil.iter_modules` 自动发现(排除 `{base,manager,registry}`),**删 .py 文件即自动失活,无硬编码名单要改**。
- 已确认核心 `loop.py`/`runner.py`/`nanobot.py`/`sdk/*` **零 import** 任何具体 channel → 删除零风险。
- pyproject 删 8 行顶层依赖(dingtalk-stream/python-telegram-bot/lark-oapi/socksio/slack-sdk/slackify-markdown/qq-botpy/python-socks)+ 6 个 optional 组(wecom/weixin/msteams/matrix/discord/whatsapp)。
- 删 29 个对应测试。

**验证**:`from nanobot.nanobot import Nanobot` ✓ / `ruff check` ✓ / 自动发现只剩 `['email','websocket']` ✓ / 全库 grep 无遗留 import ✓。

> 保留 `websocket.py`(webui gateway 承载,`manager.py` 有硬编码特例)和 `email.py`(无第三方依赖)。

### 阶段 1:机械手 tool(Modbus 模拟)✅

**文件**:[`robot_arm.py`](nanobot/agent/tools/robot_arm.py)(325 行)

**三层分层**(接真机只换底层):
```
RobotArmTool (LLM 接口,7 个 action)
   └─ RobotArmController (语义 + 安全护栏)
        └─ ArmBackend (Protocol)
             ├─ SimulatedModbusBackend   ← 当前:内存模拟 Modbus 寄存器
             └─ RealModbusBackend         ← 将来:pymodbus 读写真寄存器
```

**7 个 action**:`status` / `move_joints` / `move_xyz` / `gripper` / `home` / `estop` / `release_estop`

**安全护栏**(模拟阶段已生效):
- 关节 6-DOF 硬限位 ±170°、笛卡尔工作空间 box、速度上限
- 急停锁存(必须 `release_estop` 解除)
- `exclusive=True`(机械手动作不与其他 tool 并发)

**怎么自动注册**:落在 `nanobot/agent/tools/` 即被 [`loader.py`](nanobot/agent/tools/loader.py) 的 pkgutil 扫描自动发现(参照 [`cron.py`](nanobot/agent/tools/cron.py) 模板:`@tool_parameters` + `tool_parameters_schema`)。

**验证**:12 项实跑全对(合法/越界/急停门/解除/数不足等)。

### 桌面 exe(robot_desktop.py)✅

**文件**:[`robot_desktop.py`](robot_desktop.py)(208 行)

**结构**:
- **后台 event loop 线程**跑 SDK(不冻结 UI)
- `RobotApi` 类暴露给 JS:
  - 机械手方法 → 直接调 controller(零延迟,不依赖网络,总能演示)
  - `chat(text)` → `Nanobot.run()`(LLM 可自动调用 `robot_arm` tool)
- 内嵌极简 HTML(对话框 + 机械手按钮),`pywebview` 加载

**验证结果**(从启动日志):
- ✓ Nanobot SDK 初始化成功,注册 19 个 tool **含 `robot_arm`**
- ✓ pywebview 窗口弹出,机械手控制(move_xyz 等)正常
- ⚠️ 对话 `connection error` —— **当前开发机到智谱的网络不通**(非代码问题;工厂白名单配好或换 provider)

### 打包(mac)✅

**命令**:
```bash
pyinstaller --onefile robot_desktop.py \
  --name robot-ai \
  --paths . \
  --collect-all nanobot \
  --collect-all pywebview \
  --collect-all tiktoken \
  --add-data "nanobot/templates:nanobot/templates" \
  --add-data "nanobot/skills:nanobot/skills" \
  --hidden-import webview.platforms.cocoa \
  --noconfirm
```

**产物**:`dist/robot-ai`(mac arm64,394MB)

---

## 四、踩坑与经验(重点)

| # | 坑 | 解法 / 经验 |
|---|---|---|
| 1 | **旧版 nanobot 干扰**:系统装了 `nanobot-ai 0.1.4` editable 在**另一目录**(`web项目/nanobot-0.1.4.post4`),import/打包会抓到它而非当前源码 | `robot_desktop.py` 用 `sys.path.insert(0, 脚本目录)` 让当前源码优先;打包用 `--paths .`;**干净做法是 venv** |
| 2 | **开发机到智谱 API 网络不通**:LLM connection error | 非代码问题;工厂白名单放行后或换 provider。机械手控制不走 LLM 不受影响 |
| 3 | **PyInstaller 动态发现项目**:nanobot 的 provider/channel/tool 是 pkgutil 运行时发现,PyInstaller 静态分析会漏 | `--collect-all nanobot` 抓全包+数据;`--collect-all pywebview/tiktoken`;`--hidden-import webview.platforms.cocoa` |
| 4 | **exe 体积 394MB 偏大**:`--collect-all nanobot` 把系统装的 scipy/numpy/matplotlib 等无关包拖进来了(scipy 还有 macOS SDK 签名警告) | 加 `--exclude scipy --exclude numpy --exclude matplotlib --exclude pandas` 瘦身(机械手/对话用不到) |
| 5 | **ruff**:gotchas 明确——用 `ruff check`,**不要用 `ruff format`**(会破坏 git blame) | 只 check |
| 6 | **macOS 没有 `timeout` 命令**(GNU coretools) | 直接跑 + 工具 timeout,或 `gtimeout` |
| 7 | **工厂 EDR 拦端口**:gateway+浏览器方案在工厂大概率被拦 | 零端口 pywebview 是正解,不是妥协 |
| 8 | **机械手物理安全**:软件急停不可靠 | 必须**独立硬件急停按钮** + 高危动作二次确认门;软件急停/限位只是第二道防线 |
| 9 | **pywebview JS↔Python 桥是同步的**:LLM 响应慢会冻结 UI | 用后台 event loop 线程 + `run_coroutine_threadsafe`;机械手方法同步快不冻结 |
| 10 | **PyInstaller 跨平台限制**:mac 打不了 Windows .exe | 要 Windows .exe 必须在 Windows 机器/VM/CI 打 |

---

## 五、关键文件清单

| 文件 | 作用 | 状态 |
|---|---|---|
| [`工厂机械手语音AI-改造方案与计划.md`](工厂机械手语音AI-改造方案与计划.md) | 完整方案与选型 | 已存 |
| [`工厂机械手语音AI-实施计划-阶段0.md`](工厂机械手语音AI-实施计划-阶段0.md) | 阶段 0 逐步执行 | 已存 |
| `工厂机械手语音AI-经验总结与实现记录.md` | 本文 | 已存 |
| [`nanobot/agent/tools/robot_arm.py`](nanobot/agent/tools/robot_arm.py) | 机械手 tool(Modbus 模拟+安全) | 新增 ✅ |
| [`robot_desktop.py`](robot_desktop.py) | pywebview 桌面入口 + Api 桥 | 新增 ✅ |
| `dist/robot-ai` | mac 可执行(394MB,待瘦身) | 新增 ✅ |
| `nanobot/channels/{14个chat}.py` | 已删 | 删除 ✅ |
| `tests/channels/test_{14个}_*.py` + `tests/test_msteams.py` | 已删 29 个 | 删除 ✅ |
| `pyproject.toml` | 删 8 顶层依赖 + 6 optional 组 | 修改 ✅ |

---

## 六、当前产物与运行

**开发运行**(改代码后即时看效果):
```bash
python3 robot_desktop.py      # 弹出窗口
```

**分发的 mac 可执行**:`dist/robot-ai` 双击运行。

**能用的**:机械手控制(查状态/关节移动/坐标移动/夹爪/回零/急停,全模拟,含安全护栏)。
**待联网**:对话(需到智谱 glm-4.5-air 的网络通)。

---

## 七、后续待办

1. **阶段 1 接真机**:拿到机械手型号 + Modbus 寄存器表 → 实现 `RealModbusBackend`(pymodbus),寄存器地址/缩放按手册调,替换 `SimulatedModbusBackend`。Controller/Tool 不变。
2. **阶段 2 语音**(用户暂搁置):
   - TTS 新增:照抄 STT 三层(`audio/tts_registry.py` + `audio/tts.py` + `providers/tts.py`)
   - 连续对话状态机:`webui/src/hooks/useConversationSession.ts`(听→录→转写→思考→播报)
   - pywebview 桥改 [`webui/src/lib/nanobot-client.ts`](webui/src/lib/nanobot-client.ts) 的 WebSocket → `window.pywebview.api`
   - AEC:`useVoiceRecorder.ts` 的 `getUserMedia` 加 `echoCancellation/noiseSuppression/autoGainControl`
3. **阶段 3 Windows exe**:在 Windows 环境跑下面 spec(mac 打不了);带 WebView2 运行时。
4. **工厂白名单域名**:确认 IT 放行哪些云 → 定 LLM/STT/TTS provider。
5. **exe 瘦身**:`--exclude scipy/numpy/matplotlib/pandas` 等。
6. **接真机前补**:高危动作二次确认门 + 实体急停按钮接驳。

### Windows 打包命令模板(到 Windows 机器跑)

```bat
pyinstaller --onefile robot_desktop.py ^
  --name robot-ai ^
  --paths . ^
  --collect-all nanobot ^
  --collect-all pywebview ^
  --collect-all tiktoken ^
  --add-data "nanobot/templates;nanobot/templates" ^
  --add-data "nanobot/skills;nanobot/skills" ^
  --exclude scipy --exclude numpy --exclude matplotlib --exclude pandas ^
  --noconfirm
```
> 注意:Windows 的 `--add-data` 分隔符是 `;`(mac 是 `:`);pywebview 在 Windows 用 WebView2(Win10 1803+ 自带,老系统要带运行时)。

---

## 八、一句话总结

从「分析一个开源 agent 框架」到「产出工厂可双击运行的机械手 AI exe」,核心是:**用 pywebview+SDK 绕开工厂端口限制,用 Modbus 模拟层让机械手 tool 不阻塞于真实硬件,把安全护栏(限位/急停/exclusive)做在模拟阶段就到位**。当前 mac 上窗口+机械手全链路跑通并打成可执行;接真机、语音、Windows 打包是三个待解锁的后续动作,各自有清晰的解锁条件(协议文档/启动语音/Windows 环境)。
