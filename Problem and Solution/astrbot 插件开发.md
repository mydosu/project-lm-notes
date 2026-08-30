> 回到 [[2026.8.30 Main]] · 排障 [[Live2D Kiosk 排障记录]] · 板子部署 [[Kiosk部署完成]]

# astrbot 插件开发（Live2D 屏幕控制）

> 插件是**电脑端 astrbot 生态**的开发，与板子上 kiosk 本身的开发相互独立——板子只提供 HTTP/WebSocket 接口（见 [[Agent接口文档]]），插件负责把 astrbot 的消息/大模型能力翻译成接口调用。

## 架构

```
astrbot ──(LLM 工具/消息)──▶ live2d-kiosk 插件 v2.0（内存队列）──(板子壳 3s 轮询 pending API)──▶ 板子壳 board_shell.py ──POST /api/send──▶ 后台 :8080 ──/api/poll 2s──▶ 屏幕页面
   │                          │
   │                          └── v1.x 旧架构：插件直连 POST /api/send（已废弃）
   └──(对话消息)──────────────────────────────▶ 气泡/表情
```

插件仓库：https://github.com/mydosu/astrbot-live2d-kiosk（独立于主仓库 live2d-kiosk）

## 核心能力：LLM 工具调用（function-calling）

大模型分析对话后**自动调用工具**控制屏幕，不需要指令。注册 3 个工具：

| 工具 | 作用 | 大模型调用时机 |
|---|---|---|
| `live2d_emotion(emotion)` | 切换表情 | 开口回复前——**表情反映助手自己这句话的情感**（开心答应→happy、安慰→sad/shy、被逗笑→surprised），不是转述用户情绪 |
| `live2d_action(action)` | 触发动作 | 说话时配合肢体语言（挥手/拍一下/待机） |
| `live2d_speak(text)` | 屏幕气泡显示 | 准备回复时把完整回复显示到屏幕 |

**实现要点**（astrbot v4，源码 register_llm_tool 确认）：

```python
from astrbot.api.all import *

@llm_tool(name="live2d_emotion")
async def live2d_emotion(self, event: AstrMessageEvent, emotion: str):
    """控制屏幕表情。表情应反映你这句话想表达的情感——不是转述用户的情绪。

    Args:
        emotion(string): 情感词或代号（happy→F01, angry→F03, think→F04, sad→F05,
                         surprised→F06, shy→F07, pout→F08；或 F01~F08 / exp_01~exp_08）
    """
    emo = self._map_emotion(emotion)
    ok, err = await self._send({"type": "emotion", "value": emo})
    return f"已切换表情 {emo}" if ok else f"切换表情失败：{err}"
```

- 装饰器 `@llm_tool(name=...)`（`astrbot.api.all` 导出，`register_llm_tool as llm_tool`）
- **函数必须 async**，首个参数 `event: AstrMessageEvent`（LLM 调用时注入），后面是工具参数
- **docstring 的 `Args:` 段**由 AstrBot 解析生成工具 schema（参数类型 string/number/object/array/boolean）
- 返回 str → 加入下一次 LLM 请求 prompt（让 LLM 知道工具结果）；返回 None → 不加入
- 可用 `yield event.result(...)` 直接发消息

## 情感映射

| 情感词 | 表情代号 | 说明 |
|---|---|---|
| happy / joy / 开心 / 哈哈 | `F01` | 开心（Haru 模型 F 系列） |
| angry / mad / 生气 | `F03` | 生气 |
| think / 思考 | `F04` | 思考 |
| sad / cry / 难过 | `F05` | 难过 |
| surprised / 惊讶 | `F06` | 惊讶 |
| shy / 害羞 | `F07` | 害羞 |
| pout / 不满 | `F08` | 不满 |

Mao 模型用 `exp_01`~`exp_08`；Hiyori 无表情只有动作。模型可在管理后台切换。

## 手动指令（备用）

```
/屏幕 表情 happy      切换表情（情感词或代号）
/屏幕 动作 tapbody_0  触发动作（tapbody_0、tap、idle）
/屏幕 说 你好呀       气泡显示文字
/屏幕 状态            查询屏幕模型/信息源
/屏幕 帮助            帮助
```

## 自动行为（插件配置可关）

- `speak_user_msg`（默认开）：收到消息转发到屏幕气泡（`你：...`）
- `auto_emotion`（默认关）：关键词情感自动切表情（有 LLM 工具后通常不需要）

## 安装与配置

1. 插件目录放 astrbot `data/plugins/`（`git clone https://github.com/mydosu/astrbot-live2d-kiosk.git`）
2. WebUI 启用插件
3. 配置 `board_url`：USB 连接 `http://192.168.137.2:8080`（默认）；局域网 `http://192.168.5.32:8080`

## 验证记录（v1.1.0）

- mock astrbot 环境：3 工具注册、docstring Args、情感映射、失败分支——16 项 PASS
- `py_compile` 通过
- 板子全链路实测：POST `/api/send`（emotion F01 / action tapbody_0 / speak）→ `{"ok":true}` → 页面气泡显示

## 版本历史

- v2.0.0：**队列模式**（插件=消息生产者，板子壳=消费者）——消息/LLM 回复入内存队列，3 Web API 供壳轮询（pending/sessions/ping）+ clear API；消息带会话 origin；屏幕显示助手回复（speak_user_msg 默认 false）；去 ws 直连
- v1.1.0：新增 LLM 工具调用；工具描述改为**反映助手自身情感**；auto_emotion 默认关
- v1.0.0：初始（/屏幕 指令 + 关键词情感）

## v2.0 队列模式（2026.8.30）

### 架构（板子在外面场景）

```
QQ/微信 ──▶ astrbot（家里）──▶ 插件 on_message/LLM 工具（带 origin 入队）
                                    │
                    3 个 Web API（前缀固定 /api/v1/plugins/extensions/live2d-kiosk/）：
                    pending（拉取即清空）/ sessions（会话列表）/ ping / clear
                                    │ 板子壳 board_shell.py 每 3s GET pending（Bearer API Key）
                                    ▼
                     过滤（astrbotSession）→ POST 板子 localhost:8080/api/send
                                    │
                    页面 2s 轮询 /api/poll（拉取即清空）→ 气泡/表情
```

关键点：**板子主动出站**（无公网/隧道），Windows 仅 ICS 网桥；API Key 只勾 **plugin scope**。

### 核心代码（main.py 关键段）

```python
# 消息/LLM 回复统一入队（带会话 origin）
def _enqueue(self, msg: str, origin: str = ""):
    async with self._lock:
        self._queue.append({"type": "speak", "text": msg[:200], "origin": origin})

# 注册 Web API（AStrBot 扩展路由前缀固定）
self.context.register_web_api("live2d-kiosk/pending", self._pending, ["GET"], "拉取待显示消息")
self.context.register_web_api("live2d-kiosk/sessions", self._sessions, ["GET"], "会话列表")
self.context.register_web_api("live2d-kiosk/ping", self._ping, ["GET"], "探活")
self.context.register_web_api("live2d-kiosk/clear", self._clear, ["POST", "GET"], "清空队列")

# LLM 回复提取链（result_chain 空时兜底 raw_completion）
@register_on_llm_response()
async def on_llm_response(self, event, result_chain, raw_completion=None):
    text = ""
    if result_chain is not None:
        try:
            m = result_chain.message() if callable(result_chain.message) else result_chain.message
            text = (m or "").strip()
        except Exception:
            text = ""
    if not text and getattr(result_chain, "_completion_text", None):
        text = result_chain._completion_text.strip()
    if not text and raw_completion:
        try:
            text = raw_completion.choices[0].message.content or raw_completion.choices[0].message.reasoning_content or ""
        except Exception:
            text = ""
```

### 加载失败 4 连修（AStrBot 4.27.4 实测）

| 现象 | 根因 | 解决 |
|---|---|---|
| 插件加载失败 | metadata `name: live2d-kiosk` 含连字符，非法 Python 标识符 | `name: live2d_kiosk`（目录同步改名） |
| `name 'event' is not defined` | all.py 无 `event` 导出，`@event.register` 不可用 | `@event_message_type(EventMessageType.ALL)` |
| `EventMessageType has no attribute 'ALL_MESSAGE'` | 枚举成员是 `ALL` | `EventMessageType.ALL` |
| `__init__() missing config` | 构造参数必填 | `config: AstrBotConfig = None` + `cfg = config or {}` |

### AStrBot API 对齐铁律

- 插件名必须合法 Python 标识符；装饰器用 `@event_message_type(EventMessageType.ALL)`
- `MessageChain.message` 是**方法**（调用兼容属性两种形态）；`unified_msg_origin` 属性真实存在
- LLM 钩子在 `astrbot.core.star.register`（all.py 不导出）；日志用 `from astrbot import logger`（print 在 WebUI 不可见）
