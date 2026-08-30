# Live2D 终端 Agent 对接接口文档

> 版本：2026-08-30（v2，模式 B 网络架构） · 适用：Orange Pi Zero 2W Live2D 伪全息终端（Project LM）
> 本文档说明 astrbot 等 AI Agent 如何控制板子上的 Live2D 模型（表情 / 动作 / 对话气泡 / 时间天气），实现"**收到感情 → 控制模型表情与动作**"。

---

## 1. 系统架构与开机自启

**v2（当前，模式 B——板子在外面场景）**：

```text
┌─────────────┐  消息/LLM回复入队   ┌──────────────────┐  3s 轮询 pending   ┌──────────────┐  POST   ┌─────────────┐
│ astrbot 插件 │ ─────────────────▶ │ 板子壳             │ ────────────────▶ │ 板子管理后台   │ ──────▶ │ Chromium 页面 │
│ live2d-kiosk │  (内存队列)         │ board_shell.py    │  (Bearer API Key)  │ Flask :8080   │ /api/send│ (kiosk 全屏)  │
└─────────────┘                    └──────────────────┘                    └──────────────┘         │ easy-live2d  │
     │                                                                                             └─────────────┘
     └── Agent 无需关心板子网络：消息发到 QQ/微信 → 插件入队 → 板子壳拉取控屏
         页面侧：2s 轮询 /api/poll（拉取即清空），无 WebSocket
```

**v1（旧架构，已废弃）**：Agent 直连 `POST :8080/api/send` + `ws://:9000` 实时通道——仅保留直连 POST 作为局域网备选（见第 2 节方式 B）。

**开机自启（已配置，无需人工干预）**：

| 服务 | 说明 | 状态 |
|---|---|---|
| `live2d-web` | 页面静态服务（:80） | ✅ enabled |
| `live2d-kiosk` | Chromium 全屏 kiosk（Xorg+GPU 硬解+帧率保持） | ✅ enabled |
| `live2d-admin` | 管理后台 Flask :8080（配置/消息中转/轮询队列） | ✅ enabled |
| `board-shell` | 板子壳：每 3s 轮询 astrbot 插件队列 → 控屏 | ✅ enabled |
| `wifi-autoconnect` | 开机自动连接已保存的 WiFi（产品无网线） | ✅ enabled |
| NTP | 时间自动同步 | ✅ active |

> 开机流程：WiFi 自动连接 → 服务启动 → 页面加载并应用 `config.json` 中的全部配置（模型/布局/缩放/信息源）。**所有设置开机即恢复**。

**板子地址**：
| 场景 | 地址 |
|---|---|
| USB 连接（RNDIS） | `192.168.137.2` |
| 局域网（WiFi/开发） | `192.168.5.32`（DHCP，可能变化） |
| 域名 | `<板子DDNS域名>` |

**开发文件备份**：完整项目源码保存在板子 `/home/myduso/live2d-project/`（demo 前端 / admin 后台 / scripts / docs）。

---

## 2. 消息协议总览

所有消息为 **JSON 对象**，`type` 字段区分类型：

| type | 作用 | 关键字段 |
|---|---|---|
| `emotion` | **控制模型表情**（感情） | `value`: 表情名/代号 |
| `action` | **控制模型动作** | `value`: 动作组名或 `组_编号` |
| `speak` | 显示对话气泡 | `text`: 消息文本 |
| `timeinfo` | RNDIS 模式推送时间/天气/位置 | `time`/`date`/`weather`/`location` |

**发送方式**：

```text
方式 A（推荐，模式 B）：消息发到 astrbot 的 QQ/微信 → 插件 live2d-kiosk 入队 →
  板子壳 board_shell.py 每 3s 拉取 → 控屏
  优点：板子无需公网/可被寻址，Agent 侧零配置（就是正常聊天）

方式 B（备选，局域网直连）：HTTP POST 板子 /api/send（消息格式相同）
```

```bash
# 方式 B 示例：直连板子（USB 192.168.137.2 / 局域网 192.168.5.32）
curl -X POST http://192.168.137.2:8080/api/send \
  -H "Content-Type: application/json" \
  -d '{"type":"emotion","value":"F01"}'
```

HTTP 返回 `{"ok": true}` 表示已接收并转发（板子壳也走这个入口）。

---

## 3. 感情 → 表情控制（核心）

### 3.1 消息格式

```json
{ "type": "emotion", "value": "F01" }
```

`value` 是**表情名**。页面采用**模糊匹配**（精确 → 子串，大小写不敏感），命中即切换表情。

### 3.2 各模型可用表情（当前内置 8 个模型）

| 模型 | 表情 | 说明 |
|---|---|---|
| **Haru** | `F01` ~ `F08` | ✅ 8 个表情（推荐用于表情控制） |
| **Mao** | `exp_01` ~ `exp_08` | ✅ 8 个表情 |
| **Hiyori** | （无） | ⚠️ 仅动作，无表情 |
| **Mark / Natori / Ren / Rice / Wanko** | （多数无） | ⚠️ 以实际模型为准 |

> **用 Haru 或 Mao 才能完整体验表情控制**。模型可在管理后台切换，或通过 `POST /api/model` 切换。

### 3.3 示例（已实测）

```json
{"type":"emotion","value":"F01"}     // Haru 表情 1（实测 ✅ → [agent] emotion: F01）
{"type":"emotion","value":"f02"}     // 大小写不敏感，等价 F02
{"type":"emotion","value":"exp_05"}  // Mao 表情 5
```

**agent 集成建议**：把大模型的"情感倾向"映射到表情代号（happy→F01、angry→F03、sad→F05），再发 emotion 消息。

---

## 4. 感情 → 动作控制

### 4.1 消息格式

```json
{ "type": "action", "value": "tapbody_0" }
```

`value` 匹配规则（按优先级）：
1. **`组_编号`** 精确匹配，如 `tapbody_0`、`idle_0`
2. **组名子串**，如 `tap`（命中 TapBody 组第 1 个）、`idle`（命中 Idle 组第 1 个）
3. **动作文件名字串**

### 4.2 动作组说明

| 组名 | 含义 | 示例模型 |
|---|---|---|
| `Idle` | 待机动作（循环） | Haru×2 / Hiyori×9 / Mao×2 |
| `TapBody` | 点击/交互动作（一次） | Haru×4 / Mao×6 / Hiyori×1 |

### 4.3 示例（已实测）

```json
{"type":"action","value":"tapbody_0"}   // 实测 ✅ → [agent] action: TapBody_0
{"type":"action","value":"tap"}         // 组名子串 → TapBody 第 1 个
{"type":"action","value":"idle_1"}      // 指定 Idle 第 2 个
```

---

## 5. 对话气泡

```json
{ "type": "speak", "text": "你好呀，我是你的 AI 助手" }
```

- 左侧面板显示，**超长自动循环滚动**（速度后台可调 5~100px/s，默认 20）
- **滞留时间**：后台 `bubbleHold` 可配（0 = 一直显示到下一条消息，>0 = 秒数后恢复占位；默认 0）
- 后台「清空消息」按钮可立即清空屏幕（同时清插件队列积压）

---

## 6. RNDIS 模式：时间 / 天气 / 位置

产品无网卡时（或 WiFi 未连），管理后台信息源切到 **RNDIS 电脑推送**，由电脑端 agent 推送：

```json
{
  "type": "timeinfo",
  "time": "14:35",
  "date": "8月9日 星期日",
  "weather": "上海 · 29°C · 晴",
  "location": "上海"
}
```

> 字段可缺省。无推送时时间回退本地、天气显示"等待电脑推送天气…"。
> 信息源切换：管理后台"🌐 信息获取方式"（WiFi 网络获取 / RNDIS 电脑推送），保存后生效。

---

## 7. 管理 API（后台 :8080）

| API | 方法 | 说明 |
|---|---|---|
| `/api/send` | POST | 发送上述任意消息（方式 B 直连入口） |
| `/api/poll` | GET | 页面轮询队列（拉取即清空，页面 2s 调用） |
| `/api/clear` | POST | 清空屏幕消息（清插件队列 + 本地队列 + 页面占位） |
| `/api/astrbot/sessions` | GET | 插件会话列表（后台会话切换用） |
| `/api/astrbot/test` | GET | 测试 astrbot 插件连通（代理调 ping） |
| `/api/shell/status` | GET | 板子壳运行状态（读 /tmp/board_shell.status） |
| `/api/models` | GET | 模型列表 |
| `/api/model` | POST `{"model":"Haru"}` | 切换模型（页面热切换） |
| `/api/config` | GET/POST | 读/写配置（布局/开关/缩放/信息源等） |
| `/api/config/defaults` | GET | 默认配置（后台单项重置用） |
| `/api/config/reset` | POST | 重置所有设置为默认值 |
| `/api/upload` | POST | 上传新模型（zip） |
| `/api/bg/upload` `/api/bg` | POST/DELETE | 上传/移除自定义背景图 |
| `/api/wifi/scan` `connect` `status` `disconnect` | — | WiFi 管理（连接会写入自启配置） |
| `/api/poweroff` | POST | 关机 |

**管理后台交互**：设置类（显示/排版/信息源/端口）为**保存按钮模式**——改动只进草稿，点"💾 保存"才应用；"🔄 重置所有设置"一键恢复默认。模型切换/上传/WiFi/测试/关机为即时动作。

---

## 8. astrbot 接入示例

**模式 B（推荐）**：直接使用开源插件 [astrbot-live2d-kiosk](https://github.com/mydosu/astrbot-live2d-kiosk)（v2.0 队列模式）——部署后**无需写任何对接代码**，QQ/微信正常聊天即可控屏：

- LLM 工具调用：模型按自己回复的情感自动调 `live2d_emotion` / `live2d_action` / `live2d_speak`（表情/动作/气泡）
- 后台配置：板子管理后台「智能助手」填 astrbot 地址 + API Key（只勾 plugin scope）+ 会话选择

**方式 B（自写插件/局域网直连）**：

```python
import json
import requests

BOARD = "http://192.168.137.2:8080"   # 板子地址（USB 直连 / WiFi）

EMOTION_MAP = {   # 情感 → 表情代号（Haru 模型）
    "happy":  "F01",
    "angry":  "F03",
    "sad":    "F05",
    "surprised": "F06",
    "calm":   "F08",
}

def send(type_, **kw):
    requests.post(f"{BOARD}/api/send",
                  json={"type": type_, **kw}, timeout=5)

def express(mood: str, text: str = None):
    """收到 AI 感情 → 控制模型表情/动作 + 气泡"""
    code = EMOTION_MAP.get(mood)
    if code:
        send("emotion", value=code)       # 切换表情
    send("action", value="tapbody_0")     # 伴随动作（可选）
    if text:
        send("speak", text=text)          # 对话气泡

# astrbot 插件事件里调用：
# express("happy", "今天有什么开心的事吗？")
```

---

## 9. 调试 / 测试

| 方法 | 说明 |
|---|---|
| 管理后台「智能助手」卡片 | 测试连接按钮 + 壳状态区（10s 自动刷新） |
| curl | 见第 2 节方式 B 示例（直连 /api/send） |
| 页面日志 | 板子 `grep '\[agent\]' /tmp/kiosk-chromium.log` 查看表情/动作是否命中 |
| 查询模型可用表情 | 板子读模型 `model3.json` 的 `FileReferences.Expressions` |

---

## 10. 注意事项

1. **表情名用模型实际的代号**（`F01` 等），不是"happy/sad"语义词——需要在 agent 里做情感→代号映射（见第 8 节）
2. **Hiyori/Mark 等模型无表情**——emotion 消息会被忽略（页面日志提示），动作正常
3. 消息在模型加载中发送会被忽略（页面自动恢复后需重发）——agent 可做简单重试
4. 所有配置持久化在板子 `/opt/dashboard/live2D panel/config.json`，**重启不丢失**；部署新版本时 config.json 会被自动备份恢复
5. **WiFi 开机自启**：管理后台连过一次 WiFi 后，开机自动重连（wifi-autoconnect 服务）
