# Project LM 开发笔记（Live2D 伪全息终端）

Orange Pi Zero 2W 上构建 **Live2D 伪全息桌面终端**（2.8" HDMI 竖屏 + Chromium kiosk + 分光棱镜）的完整开发记录。

> 本仓库是 [live2d-kiosk](https://github.com/mydosu/live2d-kiosk) 主仓库配套的**开发笔记库**，由 Obsidian 管理（wikilink 双链），用 Obsidian 打开体验最佳；GitHub 网页端也能按 Markdown 直接阅读。

## 内容结构

```
Date/                     每日开发摘要（Main=主线 / Branch=支线）
Problem and Solution/     排障记录库 + 部署手册（现象|根因|解决 表格）
用户手册/                 面向使用者的操作文档（Windows ICS / RNDIS 驱动）
Agent接口文档.md          板子接口协议（消息协议 / API 说明）
```

## 覆盖主题

- **部署全流程**：Armbian 刷写 → 显示链（`xrandr --mode 480x640 --rotate left --reflect y`）→ Chromium kiosk → systemd 服务编排
- **性能与稳定性**：Mali-G31 硬解姿势（`--use-angle=gles`）、12fps 帧率保持（CDP keepalive）、H616 CCU 内核崩溃（schedutil → performance governor）、2GB 内存 OOM 教训
- **网络架构（模式 B）**：板子无公网场景——astrbot 插件内存队列 + 板子壳 3s 轮询，板子主动出站，Windows 仅 USB(RNDIS) 网桥
- **配套插件**：astrbot-live2d-kiosk（LLM 工具控制屏幕表情/动作/气泡）——见 [astrbot-live2d-kiosk](https://github.com/mydosu/astrbot-live2d-kiosk)
- **后台管理系统**：Flask 管理后台（模型/显示/排版/背景/字体配置，移动端可用）

## 排障记录亮点（Problem and Solution/Live2D Kiosk 排障记录.md）

- 气泡长消息超屏 → `word-break: break-all` + `transform-origin: left top` + 宽度随缩放反比
- 字体切换闪现 → `font-display: block` + `document.fonts.load()` 预加载 + 防竞态
- 插件加载 4 连错（AStrBot API 对齐：插件名标识符 / `@event_message_type(EventMessageType.ALL)` / config 可选 / `register_on_llm_response`）
- LLM 回复提取链：`result_chain.message`（方法形态）→ `_completion_text` → `raw_completion` 兜底

## 说明

- 笔记为开发过程的真实记录（含失败尝试），非教程——按时间线阅读 `Date/` 可看到完整演进
- 内网 IP 保留为技术上下文；公网域名/账号已打码（`<板子DDNS域名>` 等占位符）
- 排障记录与部署手册分离：**"怎么搭成的"→ `Kiosk部署完成.md`；"踩了什么坑、根因、怎么修"→ 排障记录**

## 许可

MIT（见 LICENSE）。笔记中的截图资源版权归原作者，仅作演示。
