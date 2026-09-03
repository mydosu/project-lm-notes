> 回到 [[2026.8.9~8.10 Main]] · 排障 [[Live2D Kiosk 排障记录]] · Agent 对接 [[Agent接口文档]]

# 2026.8.9 Kiosk 部署完成

## 成果

Orange Pi Zero 2W + 2.8" HDMI 屏的 Live2D kiosk **已跑通**：开机自启 → Xorg → Chromium 全屏 → easy-live2d 模型（Hiyori 等 8 个）→ GPU 硬件加速渲染。

## 系统架构

| 组件 | 方案 |
|---|---|
| 显示 | Xorg（root 启动，modesetting + glamor on Mali-G31） |
| 窗口管理 | openbox（极轻量 WM，让 Chromium 真正全屏） |
| 浏览器 | **snap Chromium 150**（自带完整 ANGLE 库，替代 xtradeb 151）+ `--use-gl=angle --use-angle=gles` |
| GPU | ANGLE GLES 后端 → 系统 mesa → **Panfrost Mali-G31 硬件**（非软渲染） |
| Web 服务 | systemd `live2d-web`（python http.server :80，CAP_NET_BIND_SERVICE） |
| Kiosk | systemd `live2d-kiosk`（xinit + kiosk.sh，开机自启） |
| 屏幕旋转 | `xrandr --output HDMI-1 --rotate left`（2.8" 竖屏面板） |

## 核心配置文件（完整代码）

### 1. `/opt/dashboard/kiosk.sh` —— Kiosk 启动脚本（Xorg 启动后自动执行）

**作用**：屏幕旋转+棱镜翻转 → 禁休眠 → CPU 高频 → openbox/unclutter → 清理残留 → 以 myduso 身份启动 Chromium 全屏（含 keepalive 帧率保持）。**注意 `--mode 480x640`（面板原生竖屏，rotate 后 4:3 与 viewport 等比，否则变形白屏）与 `--reflect y`（棱镜反射补偿）**。

```bash
#!/bin/bash
export DISPLAY=:0
# 等待 X server 就绪
for i in $(seq 1 30); do
  xdpyinfo -display :0 >/dev/null 2>&1 && break
  sleep 0.5
done
# 屏幕旋转+翻转：--mode 480x640（原生竖屏，rotate left 后 640x480 与 viewport 800x600 等比）
# --reflect y：分光棱镜反射像上下颠倒 → 输出垂直翻转后棱镜画面正立
xrandr --output HDMI-1 --mode 480x640 --rotate left --reflect y 2>/dev/null
sleep 1
# 禁用屏幕休眠（产品需常亮）
xset -dpms 2>/dev/null; xset s off 2>/dev/null; xset s noblank 2>/dev/null
# CPU 固定最高频（消除调频抖动）
echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor 2>/dev/null
sleep 0.5
# openbox：无 WM 时 Chromium 无法真正全屏
openbox >/dev/null 2>&1 & disown
# 隐藏鼠标指针
pkill -f "unclutter" 2>/dev/null
unclutter -idle 1 -root >/dev/null 2>&1 & disown
# 清理残留进程
pkill -f "chromium.*kiosk" 2>/dev/null; pkill -f "chrome.*kiosk" 2>/dev/null
pkill -f "keepalive.mjs" 2>/dev/null
sleep 1
exec su - myduso -c '
export DISPLAY=:0
export XDG_RUNTIME_DIR=/run/user/1000
# keepalive：挂起 CDP evaluate 强制合成器持续出帧（无连接时仅 3.8fps → 12fps）
(sleep 10; export PATH=/opt/node/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin; cd /opt/dashboard/keepalive && node keepalive.mjs) >/tmp/cdp-keepalive.log 2>&1 &
exec dbus-run-session -- /usr/bin/chromium \
  --kiosk --no-first-run --disable-session-crashed-bubble --disable-infobars \
  --disable-component-update --disable-background-networking --disable-extensions --disable-sync \
  --disable-features=Translate,TranslateUI,LensOverlay,LensRegionSearchOverlay --disable-translate \
  --force-language=en-US --lang=en-US --autoplay-policy=no-user-gesture-required --enable-logging=stderr \
  --use-gl=angle --use-angle=gles --enable-gpu --enable-gpu-rasterization --in-process-gpu \
  --disable-vsync --disable-gpu-vsync --disable-begin-frame-backoff --disable-frame-rate-limit \
  --disable-backgrounding-occluded-windows --disable-renderer-backgrounding --disable-background-timer-throttling \
  --ignore-gpu-blocklist --no-sandbox --remote-debugging-port=9222 --start-fullscreen \
  --window-size=480,640 --window-position=0,0 --check-for-update-interval=31536000 \
  "http://localhost:80/?kiosk=1&v=20260810" > /tmp/kiosk-chromium.log 2>&1
'
```

### 2. `/etc/systemd/system/live2d-kiosk.service` —— Kiosk 开机自启服务

**作用**：开机自启 Xorg + kiosk.sh；**`TimeoutStopSec=10` 解决 sudo reboot 卡 90s**（Chromium 不响应 SIGTERM，10s 后强制 SIGKILL）。

```ini
[Unit]
Description=Live2D Kiosk (Xorg + Chromium)
After=network.target
Wants=live2d-web.service

[Service]
Type=simple
Environment=HOME=/root
WorkingDirectory=/opt/dashboard
ExecStart=/usr/bin/xinit /opt/dashboard/kiosk.sh -- /usr/bin/Xorg :0 -nolisten tcp -noreset vt1
# Chromium 不响应 SIGTERM（keepalive 挂起 evaluate 占主线程）→ 10s 后强制杀，避免重启卡死
TimeoutStopSec=10
KillMode=control-group
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3. `/etc/systemd/system/live2d-web.service` —— 页面静态服务（:80）

**作用**：python http.server 托管页面；`CAP_NET_BIND_SERVICE` 让非 root 绑定 80 端口（不需要 root 权限）。

```ini
[Unit]
Description=Live2D Web Server (static)
After=network.target

[Service]
Type=simple
User=myduso
Group=myduso
AmbientCapabilities=CAP_NET_BIND_SERVICE
WorkingDirectory=/opt/dashboard/live2D panel
ExecStart=/usr/bin/python3 -m http.server 80 --bind 0.0.0.0 --directory "/opt/dashboard/live2D panel"
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### 4. `/etc/systemd/system/live2d-admin.service` —— 管理后台（Flask :8080 + WebSocket :9000）

```ini
[Unit]
Description=Live2D Kiosk Admin (Flask :8080 + WebSocket :9000)
After=network.target
Wants=live2d-web.service

[Service]
Type=simple
User=myduso
Group=myduso
WorkingDirectory=/opt/dashboard/admin
ExecStart=/usr/bin/python3 /opt/dashboard/admin/app.py
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### 5. `/etc/X11/xorg.conf.d/10-glamor.conf` —— GPU 加速 + 禁休眠

**作用**：modesetting + glamor（Mali-G31 Panfrost 加速的前提——否则 X11 EGL 平台缺失）；ServerFlags 全部 0 = 永不自动关屏。

```conf
Section "ServerFlags"
    Option "BlankTime" "0"
    Option "StandbyTime" "0"
    Option "SuspendTime" "0"
    Option "OffTime" "0"
EndSection

Section "Device"
    Identifier  "HDMI"
    Driver      "modesetting"
    Option      "AccelMethod" "glamor"
    Option      "DRI" "3"
EndSection
```

### 6. `/etc/chromium/policies/managed/kiosk-policy.json` —— 浏览器策略（消翻译气泡）

**作用**：`--disable-translate` flag 无效（M151 UI 浮层），策略文件才有效——消除右下角翻译/搜索气泡。

```json
{
  "TranslateEnabled": false,
  "LensRegionSearchEnabled": false,
  "PasswordManagerEnabled": false,
  "AutofillCreditCardEnabled": false,
  "DefaultBrowserSettingEnabled": false,
  "PromptForDownloadLocation": false,
  "DownloadRestrictions": 3,
  "BlockThirdPartyCookies": true,
  "HomepageLocation": "http://localhost/"
}
```

### 7. `/opt/dashboard/keepalive/keepalive.mjs` —— 帧率保持（12fps 关键）

**作用**：挂起 `page.evaluate`（async Promise 永不 resolve + rAF 循环保引用）。DevTools 挂起调用期间 Chromium **禁用合成器帧退避** → 无任何外部连接也稳定 12fps（否则合成器退避只有 3.8fps）。板子本地跑（node v22.23.2 在 /opt/node）。

```javascript
// 保持挂起的 page.evaluate（async Promise 永不 resolve + rAF 循环保引用）。
// DevTools 挂起调用期间 Chromium 禁用合成器帧退避 → 帧率 3.8 → ~12fps。
import puppeteer from 'puppeteer-core'

const RETRY_MS = 5000

async function hold() {
  while (true) {
    try {
      const browser = await puppeteer.connect({ browserURL: 'http://127.0.0.1:9222' })
      const page = (await browser.pages())[0]
      // 挂起 evaluate：永不 resolve（rAF 循环保持 Promise 引用避免被 GC）
      await page.evaluate(
        () =>
          new Promise((resolve) => {
            const loop = () => { requestAnimationFrame(loop) }
            requestAnimationFrame(loop)
            window.__kaResolve = resolve
          }),
      )
      await new Promise(() => {})  // 保持连接
    } catch (err) {
      console.log('[keepalive]', err.message, 'retry in', RETRY_MS, 'ms')
      await new Promise((r) => setTimeout(r, RETRY_MS))
    }
  }
}

hold()
```

### 8. `/usr/local/bin/wifi-autoconnect.sh` + `/etc/systemd/system/wifi-autoconnect.service` —— WiFi 开机自启

**作用**：产品无网线，开机自动连接后台保存过的 WiFi（wpa_supplicant.conf 存在才执行；oneshot 服务，无配置则静默跳过）。

```bash
#!/bin/bash
# WiFi 开机自动连接（由 wifi-autoconnect.service 调用；未配置过 WiFi 时静默跳过）
CONF="/etc/wpa_supplicant/wpa_supplicant.conf"
[ -f "$CONF" ] || exit 0
# 清理可能残留的 wpa_supplicant（与 dbus 版共存冲突保护）
pkill -f "wpa_supplicant.*wlan0" 2>/dev/null
sleep 1
ip link set wlan0 up 2>/dev/null
sleep 1
wpa_supplicant -B -i wlan0 -c "$CONF" 2>/dev/null
sleep 4
dhclient wlan0 2>/dev/null || dhcpcd wlan0 2>/dev/null
exit 0
```

```ini
[Unit]
Description=WiFi auto connect (Live2D kiosk)
After=network.target
Wants=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/wifi-autoconnect.sh
RemainAfterExit=no

[Install]
WantedBy=multi-user.target
```

### 9. RNDIS USB 网络（板子 192.168.30.1）

完整配置见 [[开发板和电脑端通过RNDIS通信]]；网段已改为 **192.168.30.1**（避免与用户局域网冲突）：`/etc/systemd/network/10-usb0.network` 的 `Address=192.168.30.1/24`，`/etc/dnsmasq.d/usb0.conf` 的 `dhcp-range=192.168.30.100,192.168.30.200`。

## 一键部署（电脑端）

```powershell
# 构建前端并部署到板子（自动备份/恢复 config.json，用户设置不丢失）
cd D:\CODE\Live2D
python scripts\deploy.py              # 构建 + 部署全部 + 重启服务
python scripts\deploy.py --skip-build # 跳过 npm build
python scripts\deploy.py --admin-only # 只部署后台
```

## 部署配置说明

- **sudo 免密**：`/etc/sudoers.d/myduso` 配置 NOPASSWD（kiosk.sh 内以 root 跑 Xorg、admin 后台关机/重启需要）
- 其他部署中的问题与修复（GL 后端 / panvk / 无 WM 全屏 / canvas 尺寸 / 翻译气泡 / vsync / 鼠标指针 / 白屏 / 加载遮罩 等）见 [[Live2D Kiosk 排障记录]]

## 板子文件布局（最终版，2026.8.10）

```bash
/opt/dashboard/
├── live2D panel/        # 网页静态文件 + config.json（运行时配置，用户设置持久化）
├── admin/               # 管理后台（Flask app.py + admin.html）
├── keepalive/           # 帧率保持（node v22 + puppeteer-core + keepalive.mjs）
└── kiosk.sh             # kiosk 启动脚本（旋转/GPU/keepalive/常亮/性能）
/opt/node/               # node v22.23.2 arm64
/usr/local/bin/wifi-autoconnect.sh   # WiFi 开机自启脚本
/etc/systemd/system/
├── live2d-web.service   # :80 静态服务（enabled）
├── live2d-kiosk.service # Chromium kiosk（enabled）
├── live2d-admin.service # Flask+ws 后台（enabled）
└── wifi-autoconnect.service  # WiFi 开机自连（enabled，oneshot）
/etc/X11/xorg.conf.d/10-glamor.conf  # glamor 加速 + 禁用 DPMS
/etc/chromium/policies/managed/kiosk-policy.json  # 禁翻译气泡等
```
> 板子只保留**实现功能的文件**；开发文件（源码/脚本/文档）全部保存在电脑 `D:\CODE\Live2D`。
> `/home/myduso` 保持干净（仅系统目录 + g1(RNDIS configfs) + install.sh(ddns-go)）。

## 当前帧率（最终稳定 12fps）

- **稳定 12fps**（keepalive 挂起 evaluate + `--disable-vsync`；第十轮移除气泡 `backdrop-filter: blur` 后从 9fps 恢复 12.1）
- 帧率测量方法：页面内置 rAF 计数 `[FPS]` 写入 /tmp/kiosk-chromium.log；**注意 CDP 连接本身会把 3.8→12fps**，测真实值须断开外部连接后读日志
- 已试无效：`--in-process-gpu`、`--disable-gpu-compositing`、渲染分辨率 0.75、纹理 2048→1024、`--force-frame-rate`（均无提升且已复原画质）

## 功能迭代记录（2026.8.9 ~ 8.10）

> 部署完成后的功能迭代与验证。**过程中的问题/根因/解决见 [[Live2D Kiosk 排障记录]]**，这里只记进展。

**第一~二轮：全功能版**——左半屏时间/天气/气泡 + 右半模型；Flask 管理后台 + WebSocket 控制；config.json 配置；模型画布自适应 + 后台 zoom 滑条
**第三轮：测试反馈修复**——后台按钮 JS 修复、中文字体安装
**第四轮：模型切换加固**——移除像素测量（Mali 阻塞主线程）、加载超时/切换锁/失败清理
**第五轮：界面自由排版**——layout 字段（4 模块 X/Y 偏移滑条）
**第六轮：气泡宽度优化 + 模块缩放**——fit-content 气泡 + 每模块 S 缩放（viewport 溢出改用 window.screen 宽度）
**第七轮：气泡循环滚动 + 自定义占位文本**
**第八轮：信息获取双模式**——WiFi（wpa_supplicant 手配）/ RNDIS（timeinfo 电脑推送），后台切换
**第九轮：禁用屏幕休眠**——xset + ServerFlags（产品常亮）
**第十轮：帧率修复 + ws 端口 + 保存/重置**——blur 移除恢复 12fps、wsPort 可配、draft 保存按钮模式
**第十一轮：自动化配置收尾**——开机自启确认、WiFi 自启服务、板子清理（只留功能文件）、一键部署脚本、Agent 文档
**第十二轮：重启全功能验证**——开机自启全链路 ✅
**第十三轮：伪全息适配**——棱镜 → `xrandr --reflect y` 翻转；管理后台白色+粉色清新风
**第十四轮：显示修复**——白屏/宽胖（`--mode 480x640` 锁定，viewport 4:3 等比）
**第十五轮：后台侧边栏分页**——5 页导航 + 全部滑条数值输入 + 模型缩放迁移到界面排版
**第十六轮：后台美化**——毛玻璃 + 渐变 + 光斑 + 卡片自适应 + 网格对齐
**第十七轮：气泡字体自适应/大小 + 模块字体颜色**
**第十八轮：日期/星期独立控制**——showDate / layout.date / fontColors.date
**第十九轮：后台文案大众化**——面向消费者去技术术语
**第二十轮：RNDIS 网段 192.168.30.1**——避免与用户局域网冲突
**第二十一轮：后台重启按钮**——POST /api/reboot
**第二十二轮：加载遮罩居中**——按 window.screen 尺寸定位
**第二十三轮：reboot 根治 + 模型持久化**——TimeoutStopSec=10；部署保护 config.json

## 功能迭代记录（2026.8.30 模式 B + 产品化）

> 板子在外面场景的网络架构 + 收尾功能。**过程中的问题/根因/解决见 [[Live2D Kiosk 排障记录]]**，这里只记进展。

**第二十四轮：模式 B 网络架构**——插件 v2.0 队列模式（pending/sessions/ping Web API）+ 板子壳 board_shell.py 每 3s 轮询控屏（systemd board-shell.service）；去 ws（页面 `/api/poll` 2s HTTP 轮询，RECENT_MESSAGES 上限 50）；Windows 仅 ICS 网桥；板子主动出站（无公网/隧道）
**第二十五轮：会话切换 + 可观测性**——消息带 origin（unified_msg_origin）壳过滤、后台会话下拉（sessions API）、测试连接按钮（代理插件 ping）+ 壳状态区（/tmp/board_shell.status，10s 自动刷新）
**第二十六轮：LLM 回复上屏**——`register_on_llm_response` + raw_completion 兜底（result_chain 空时）；屏幕显示助手回复（speak_user_msg=false）
**第二十七轮：气泡体验**——滞留可配（bubbleHold 0=一直）、溢出修复（break-all + transform-origin left top + 宽度随缩放反比）、日期独立布局
**第二十八轮：清空消息 + 单项重置**——插件 clear API + `/api/clear`（清插件队列+本地+页面占位）；每数值项 ↺ 单项重置（显示 7 项 + 排版 5 模块 + 智能助手 3 项，一键重置保留）
**第二十九轮：后台美化**——装饰 emoji 全清（保留状态符号）、22px 圆角/过渡/focus 光圈、select 自定义粉色箭头、文件选择卡片化（虚线按钮+文件名）、number 箭头隐藏（多位数不遮挡）
**第三十轮：背景系统**——10 个渐变主题（极光/粉嫩/深色/薄荷/日落/海洋/星云/蜜桃/薰衣草/糖果）+ 自定义背景图（canvas 4:3 裁剪拖拽上传，图片优先于主题，切换主题自动移除）
**第三十一轮：字体系统**——气泡半透明磨砂渐变（后台自选基色 blur 8px）、8 款艺术字（圆润系 5 + 科技/优雅/中文圆体）、每模块独立字体（fontStyles）、中文适配（站酷快乐体 449KB + 小薇体 945KB，各 3838 字子集）、切换闪现修复（block + 预加载 + 防竞态）

## 功能迭代记录（2026.9.2 重构与天气修复）

> 8.30 深夜～9.2 的仓库/文档层工作（历史合并、笔记开源、Agent 文档 v2）见 [[2026.8.30 Main]]。这里的坑与根因见 [[Live2D Kiosk 排障记录]]。

**第三十二轮：后台代码重构 + 界面美化**——抽 api()/SAVE_FIELDS/SWITCH_FIELDS/draftField/pair() helper（消灭重复与嵌套三元）；按钮流光、卡片上浮、toast 弹性、侧栏顺滑过渡、粉色细滚动条
**第三十三轮：天气源系统**——wttr.in 国内不可达 → 天气源可配置（后台下拉 + Key）；默认高德（amap），wttr 海外备选，双源自动降级；fetch 加 AbortController 超时快速失败
**第三十四轮：联网模式重构（去 RNDIS 推送）**——RNDIS=底层驱动、ICS=网络共享；`infoSource` 语义从"信息源"改为**联网方式**（wifi/usb），删除 externalClock/externalWeather/timeinfo 电脑推送代码；后台文案改"USB 共享网络（电脑 ICS）"
**第三十五轮：天气自动定位**——新增 `/api/geoip` 后端代理（服务端请求 myip.ipip.net 解析省市，兼容 IPv6 出口）；城市留空自动定位实测"贵阳市 · 22°C · 晴"
**第三十六轮：后台 UI 精修**——侧栏导航线性 SVG 图标（收起态不空）+ 收起态渐变粉胶囊底 + chevron 旋转过渡按钮；全局 :focus-visible 焦点光圈、开关/滑条/色块手感、dirty 柔和呼吸、tabular-nums、窄屏触达放大、prefers-reduced-motion；修 dev-addr 旧地址残留（30.1→137.2）

## 最终常用命令

```powershell
# 一键部署（本地，构建+部署+重启，config.json 自动保护）
cd D:\CODE\Live2D; python scripts\deploy.py

# 密码 SSH 通道（板子密钥失效时）
python D:\CODE\rk-ssh.py "命令"

# 板子查帧率 / agent 消息命中 / ws 端口
grep '\[FPS\]' /tmp/kiosk-chromium.log | tail
grep '\[agent\]' /tmp/kiosk-chromium.log | tail
ss -tlnp | grep 9000

# 板子重启后验证自启
systemctl is-active live2d-web live2d-kiosk live2d-admin wifi-autoconnect
```

## 调试工具

- Chromium `--remote-debugging-port=9222` + 本地 puppeteer-core（ssh 隧道 `ssh -L 9222:127.0.0.1:9222`）
- 页面暴露 `window.__kioskDebug`（sprite 坐标/screen 尺寸）
- 板子截图 `DISPLAY=:0 scrot -o /tmp/x.png`
