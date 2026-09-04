# Live2D Kiosk 排障记录

> 2026-08-09 部署过程中的关键问题与解决（详见 [[Kiosk部署完成]]）

## 2026.9.2 排障（天气源与网络）

| 现象 | 根因 | 解决 |
|---|---|---|
| 屏幕天气一直"天气不可用" | wttr.in 的 IP（5.9.243.187）从板子网络路径 **TCP 443 超时**（国内网络屏蔽国外站点）；而 baidu 秒通 | 天气源可配置：默认**高德**（国内可达秒通），wttr 降为海外备选；双源自动降级（详见下方链路诊断） |
| 天气请求长时间显示 `--` 不刷新 | 浏览器 fetch 无默认超时，wttr 连接超时可能 >10s | `fetchTimeout()`：AbortController 6~8s 中断，快速失败显示"天气不可用" |
| 城市留空自动定位总是"上海" | 代码 `city = ... || '上海'` 兜底——直查上海成功即返回，**永远走不到 IP 定位分支** | 删除上海默认；城市留空走定位逻辑 |
| 删除上海默认后自动定位仍失败 | 板子经 USB/ICS **只有 IPv6 公网出口**（IPv4 NAT 不通，`curl -4 baidu` 000）；高德 `/v3/ip` 对 IPv6 出口返回空 adcode（Windows 本机同 key 也空——IPv6 定位不支持） | 新增 **`/api/geoip` 后端代理**（admin app.py）：服务端请求 `myip.ipip.net`（实测支持 IPv6 定位，返回"中国 贵州 贵阳"）→ 解析省市 → 页面用城市名查高德天气——实测"贵阳市 · 22°C · 晴" |
| 高德天气接口带城市名返回空 | （无此问题）高德 weatherInfo/geocode **填城市名/adcode 完全正常**（上海小雨 27°C、贵阳晴 22°C 实测）——只 IP 定位接口对 IPv6 空 | 定位走 geoip 代理后正常 |
| `curl --interface` 绑 IP 测连通反而全超时 | `--interface` 绑 IP 会绕过路由表/走错路径（baidu 也 000） | 诊断连通性用默认路由 + 对照（baidu 通/wttr 不通），别用 --interface |

**网络链路诊断结论**（2026.9.2 实测）：Windows 本机直连 wttr.in 200（走以太网2 家庭路由 IPv4）；板子默认路由首选 usb0(137.1 ICS)，但 ICS 仅 IPv6 出网可达国外域名解析、IPv4 NAT 不通；板子经 eth0(家庭局域网) 到 wttr 也超时 → **当前环境任何路径都无法免 key 访问国外天气源**，国内免 key 源（中国天气网/墨迹/百度）也全超时 → 唯一可行=高德 Web 服务（需 key）。

**关键修复代码**：

```python
# admin/app.py —— /api/geoip 代理（兼容 IPv6 出口的 IP 定位）
@app.route("/api/geoip")
def api_geoip():
    import re as _re, urllib.request
    with urllib.request.urlopen("https://myip.ipip.net", timeout=8) as r:
        text = r.read().decode("utf-8", errors="ignore")
    m = _re.search(r"来自于：(.+)", text)
    parts = [p for p in m.group(1).split() if p]   # ["中国","贵州","贵阳","电信"]
    province, city = parts[1], parts[2]
    if province in ("北京", "上海", "天津", "重庆"): city = province
    elif city.endswith("市"): city = city[:-1]
    return jsonify({"ok": True, "province": province, "city": city})
```

```javascript
// demo/src/main.js —— 城市留空自动定位（页面 → 本地代理 → 高德天气）
const geo = await (await fetch(`${API_BASE}/api/geoip`)).json()   // 板子无 CORS 限制
city = geo.city || ''                                             // 如"贵阳"
// 再用城市名查高德 weatherInfo（高德支持城市名直查）
```

## 2026.8.30 排障（产品化收尾）

| 现象 | 根因 | 解决 |
|---|---|---|
| 气泡调大后长消息超出屏幕边框（左边超） | 长串无空格不断行（break-word 不够）+ transform 缩放以中心为原点（左右同时扩） | `word-break: break-all` + `transform-origin: left top`（左对齐、向右下扩展）+ `max-width: calc(100% / var(--bubble-scale))`（放大后视觉宽度不超左半屏） |
| 气泡放大后覆盖右侧模型 | scale 1.55 时视觉宽度超过左半屏（fit-content 不受限） | max-width/max-height 随缩放反比（CSS 变量 `--bubble-scale`，JS 设置）——实测右缘 355 < 模型区 400 |
| 测试后时间/日期/天气位置全变 | 测试脚本 POST 完整 layout 覆盖用户自定设定（无备份） | 从部署备份 `/tmp/cfg.bak` 恢复用户 layout（时间 1.75/日期 1.6/天气 1.35/气泡 1.55/字号 16）；**测试铁律：只发消息不动配置** |
| 每次切换字体闪现默认字体再切换 | `@font-face font-display: swap`——字体加载期间先显示 fallback | `font-display: block` + `document.fonts.load()` 预加载（等就绪再应用）+ seq 令牌防竞态（快速连切只应用最新）——实测 8 次采样无中间 fallback |
| 气泡背景灰蒙蒙 | 固定 `rgba(24,24,42,.88)` 深色底 | 后台可选基色 → 135° 半透明磨砂渐变 `linear-gradient(135deg, 色88→色55→色35)` + `backdrop-filter: blur(8px)`（气泡为静态元素低频变化，实测不掉帧——区别于早年整体 HUD blur 每帧拖累） |
| 会话选择重启后失效（下拉无选中） | 插件重启 sessions 内存清空，下拉无对应选项 | 下拉加"（已保存）"兜底 option + `cur` 用 config 值兜底 |
| 保存会话后刷新后台丢失 | `load()` 不自动拉会话列表（只有手动刷新） | `load()` render 后自动 `refreshSessions(true)`（静默，未配置不 toast） |
| 调时间大小/位置时日期跟着动 | `date-display` 是 `time-block` 子元素（transform 作用于容器） | 移出为平级元素（`margin-top:-14px` 视觉贴近） |

## 帧率问题（最重要）

| 现象 | 根因 | 解决 |
|---|---|---|
| 检测时 12fps，平时 3.8fps | CDP/DevTools 挂起 evaluate 会禁用合成器帧退避 | 板子本地 node + puppeteer-core **keepalive.mjs** 挂起 evaluate → 稳定 12fps |
| 12fps → 9fps | 气泡 `backdrop-filter: blur(6px)` 每帧 GPU 模糊（Mali-G31 开销大） | 移除 blur（背景加深补偿）→ 恢复 12.1 |
| GPU 硬解 | `--use-angle=gles`（ANGLE GLES 后端直通系统 mesa → Panfrost）；snap Chromium 150 自带完整 ANGLE 库 | 已验证 glRenderer: `ANGLE (Mesa, Mali-G31 MC1 (Panfrost))` |

## 模型

| 问题 | 解决 |
|---|---|
| 切换模型卡死（强制重启） | `gl.readPixels` 同步读 GPU 永久阻塞主线程 → 彻底移除像素测量，改用画布尺寸 + zoom 滑条 |
| Haru 加载超时 | ready 超时 15s 过紧（Haru 需 20-30s）→ 放宽 30s + 切换锁 + 失败清理 |
| 像素分析不可行 | extract.pixels 返回 undefined、RenderTexture 渲染全透明（Mali GLES）→ 放弃，画布尺寸自适应 |

## 后台 / WebSocket

| 问题 | 解决 |
|---|---|
| ws 9000 无监听（静默失败） | websockets v15 `serve()` 端口绑定在 `async with` 进入时——`await gather(serve(...))` 不绑定 |
| ws 握手永不完成（TCP ESTAB 但 handler 不执行） | relay 用同步 `queue.Queue.get(timeout=1)` 卡死事件循环 → 改 **asyncio.Queue + run_coroutine_threadsafe** |
| 管理后台按钮无反应 | admin.html `$('sw-' + id)` 拼出 `sw-sw-time`（null）→ 脚本中断 → 修复为 `$(id)` |

## 显示 / 系统

| 问题 | 解决 |
|---|---|
| 翻译气泡 | `--disable-translate` 无效 → Chromium policy 文件（`TranslateEnabled: false`） |
| GPU 硬解不可用（llvmpipe/拒绝） | `--use-gl=egl` 被拒、`--use-angle=gl` 失败、`--use-angle=vulkan` 只有 llvmpipe 软渲染 → **`--use-angle=gles`**（ANGLE GLES 后端直通系统 mesa → Panfrost） |
| panvk 崩溃 | Mali-G31（Bifrost v7）panvk 实验性（需 `PAN_I_WANT_A_BROKEN_VULKAN_DRIVER=1`），实测崩溃 → 弃用，回退 gles 硬解 |
| Chromium 无法真正全屏 | 无 WM 时 kiosk 窗口停在默认 800x600 → 装 openbox（极轻量 WM） |
| 鼠标指针常驻 | `unclutter -idle 1 -root`（静止隐藏、移动重现） |
| 中文显示方框 | 板子缺中文字体 → `apt install fonts-noto-cjk` |
| 屏幕 10 分钟休眠 | Xorg DPMS 默认 600s → `xset -dpms` + ServerFlags 全 0 |
| 加载遮罩不在屏幕中央 | Chromium viewport(800x600) 与屏幕(640x480) **1:1 映射**（CDP/scrot 对比证实），`fixed inset:0` 居中于 viewport(400,300) 而非屏幕(320,240) | loader 改按 `window.screen` 尺寸定位 → 居中（质心 380,278 → 303,232） |
| UI 百分比错位（panel 400px > 左半 320） | viewport 800 vs 窗口 640 溢出 → side-panel 宽度用 `window.screen.width / 2` |
| 白屏 + 画面宽胖变形 | `--reflect y` 触发 mode 重协商漂移 768x1024；误用横屏 mode（rotate 后 3:4 与 viewport 4:3 比例相反） | `xrandr --mode 480x640`（面板原生竖屏，rotate 后 4:3 等比）+ 锁定 mode；**kiosk viewport 固定 800x600，X screen 必须同比例** |
| Chromium 一直加载旧版页面 | index.html 被缓存（引用旧 bundle） | kiosk URL 加版本参数 `?v=20260810` 强制刷新 |
| SSH 密钥认证被拒 | home 目录权限 0707（other 可写）→ sshd StrictModes 拒绝 → `chmod 700` |
| iw scan 500 错误 | signal 值 `"-85.00"` 带小数 → `int(float(...))` 解析 |
| dnsmasq 启动失败（illegal repeated keyword） | 备份 `.bak` 文件留在 /etc/dnsmasq.d/ 被重复读取 | 备份文件移出目录 |
| 新配置字段 POST 后不生效 | 改了 app.py 但没上传板子/没重启 live2d-admin | 上传 app.py + `systemctl restart live2d-admin` |
| 屏幕卡死（SSH 连不上、ping 通） | 激进 Chromium flags `--renderer-process-limit=1` 单渲染进程内存暴涨 → 2GB+zram 耗尽 OOM 瘫痪 | 撤回全部激进 flags（kiosk.sh 注释警示） |
| 板子开久了自动重启/卡死 | **schedutil 频繁调频触发 H616 CCU 时钟驱动 bug**（内核 Oops: `ccu_div_set_rate` / `sugov:0`） | 恢复 performance + `cpufreq-performance.service` 开机锁定（Before=live2d-kiosk） |
| 模型大小/设置重启后变回默认 | 手动部署 `cp -r dist/.` 覆盖了 config.json（dist 是默认值） | 统一走 deploy.py（自动备份恢复 config.json），禁止手动 cp |

## 关键修复代码

### 帧率（keepalive 完整代码见 [[Kiosk部署完成]] 第 7 节）

```bash
# 1. 移除气泡 backdrop-filter blur（Mali-G31 每帧 GPU 模糊 12→9fps）——页面 CSS 改：
#    background: rgba(24,24,42,.88)（去掉 backdrop-filter: blur(6px)）

# 2. 挂起 CDP evaluate 保 12fps（keepalive.mjs 核心，板子 /opt/dashboard/keepalive/）
node keepalive.mjs   # 完整代码见 Kiosk部署完成.md 第 7 节
```

### 模型切换卡死（readPixels 阻塞主线程）

```javascript
// main.js：彻底移除像素测量（gl.readPixels 同步读 GPU 在 Mali GLES 永久阻塞）
// 改用：画布尺寸自适应 + 后台 zoom/layout 滑条；加载加超时保护
const [model] = await Promise.race([
  loadModel(name),
  new Promise((_, rej) => setTimeout(() => rej(new Error('模型加载超时 (30s)')), 30000)),
])
```

### WebSocket 两连坑（app.py）

```python
# ① websockets v15 必须 async with 进入才绑定端口：
async with websockets.serve(handler, "0.0.0.0", port):   # 不能 await gather(serve(...))
# ② 跨线程投递必须 asyncio.Queue + run_coroutine_threadsafe（同步 queue.get 会卡死事件循环）：
asyncio.run_coroutine_threadsafe(_send_queue.put(msg), _ws_loop)
```

### 翻译气泡（Chromium 策略，flag 无效）

```bash
sudo mkdir -p /etc/chromium/policies/managed
# kiosk-policy.json 内容见 Kiosk部署完成.md 第 6 节（TranslateEnabled: false 等）
```

### 白屏 + 画面宽胖变形

```bash
# --reflect y 会触发 mode 重协商漂移 768x1024；误用横屏 mode 比例相反
# 正确：面板原生竖屏 mode，rotate 后与 viewport 800x600 同为 4:3
xrandr --output HDMI-1 --mode 480x640 --rotate left --reflect y
# 若页面加载旧 bundle：URL 加版本参数 ?kiosk=1&v=20260810（绕过 index.html 缓存）
```

### SSH 密钥认证被拒（StrictModes）

```bash
# /home/myduso 权限 0707（other 可写）被 sshd 拒绝
sudo chmod 700 /home/myduso
```

### dnsmasq 启动失败（illegal repeated keyword）

```bash
# 备份文件 .bak 留在 /etc/dnsmasq.d/ 被重复读取 → 移出目录
sudo mv /etc/dnsmasq.d/usb0.conf.bak /root/
```

### 管理后台按钮无反应（脚本中断）

```javascript
// admin.html：['sw-time',...].forEach(id => $('sw-' + id)) 拼出 sw-sw-time(null) 抛错
// 修复：直接 $(id)（不要重复拼前缀）；颜色选择器 input 事件用 'input' 而非 'change'
```

### OOM 卡死（激进 Chromium flags）

```bash
# --renderer-process-limit=1 单渲染进程内存暴涨 → 系统瘫痪（SSH 连不上但 ping 通）
# 修复：撤回全部激进 flags（勿加 renderer-process-limit / disable-software-rasterizer）
```

### schedutil 调频触发内核崩溃（自动重启根因）

```bash
# kern.log: Internal error: Oops — Comm: sugov:0 / ccu_div_set_rate（H616 CCU 时钟驱动 bug）
# 修复：恢复 performance + systemd 开机锁定（kiosk.sh 写入曾静默失败，服务双保险）
cat > /etc/systemd/system/cpufreq-performance.service << 'EOF'
[Unit]
Description=Set CPU governor to performance (stable)
After=systemd-modules-load.service
Before=live2d-kiosk.service
[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor'
RemainAfterExit=yes
[Install]
WantedBy=multi-user.target
EOF
systemctl enable cpufreq-performance.service
```

## 2026.9.4 排障（清屏/会话过滤/新模块交互）

| 现象 | 根因 | 解决 |
|---|---|---|
| 点"清空屏幕消息"没反应 | `/api/poll` 拉取即清空，kiosk 与预览 iframe **两个消费者竞争** clear 广播——常被预览抢走 | `/api/clear` 改更新 `CLEAR_TS` 时间戳，poll 返回 `clear_ts`，两端各自比对执行清屏（幂等无竞争） |
| 配置了"显示单一会话"却显示所有会话 | 壳过滤条件 `if target and origin and origin != target`——**origin 为空的消息不过滤**（LLM 自动回复等消息提取不到 origin 就为空） | 改 `if target and origin != target`：配置会话后非该会话消息一律过滤（含 origin 空） |
| 手机断开蓝牙后后台仍显示已连接 | 状态轮询间隔 15s 太长，断开后要等最多 15s 才刷新 | 轮询 15s→3s + visibilitychange 切回页面立即刷新 |
| 新 WiFi/蓝牙模块无法移动（拖拽变成缩放） | 判定"距任一边 ≤10px → 缩放"，新模块高仅 ~21px——按中间 dy 也 <10 命中缩放 | 改**四角判定**（dx 且 dy 都 ≤ EDGE 才缩放），小模块中心可移动、角上仍可缩放 |
| WiFi/蓝牙布局滑条数值框为空 | admin.html 布局模块数组共 5 处，render() 与预览回传消息处理两处**漏加 wifi/bt** | 补齐数组（`['time','date','weather','bubble','model','wifi','bt']`），滑条显示原始坐标 230/-195/1.38 |

## 2026.9.5 排障（预览交互与 pin/flow-comp 教训）

| 现象 | 根因 | 解决 |
|---|---|---|
| 取消 wifi/bt 显示后气泡位置错位（预览里贴天气、屏幕间隔远） | 模块都在 side-panel flex 列，隐藏 wifi/bt 后流变短 + flex 居中 → 其他模块整体位移，transform 偏移不变 → 错位 | 尝试两方案后**均回滚**（见下两行）：最终干净版——用户布局本就在隐藏状态下保存，无需补偿 |
| **pin 方案**（模块转绝对定位）导致预览全没、屏幕模块错位 | 把模块按"全部显示"流位置转 `position:absolute`——与用户当前隐藏 wifi/bt 的布局状态冲突 | **回滚**（git revert 1d7f718） |
| **flow-comp 方案**（translateY 流补偿）导致 iframe 预览模块全被推出屏幕（只有背景+虚线框） | `measureFlow` 在 iframe 环境时序下量到错误流位置，算出天量补偿（时间 transform 从 -140 被加成 -341px，top -161 全在可视区外）——直连/headless 时序碰巧正常，iframe 内必现 | **回滚两个 flow-comp commit**（git revert 5e1555f 7ba5b4a），回到干净版；iframe 内部实测模块位置恢复（时间 40/日期 146/天气 193/气泡 241） |
| 预览里鼠标在虚线框外（文字右侧空白）也能拖到模块 | 时间/日期/天气是块级被 flex 拉伸到整行宽（630px），虚线框只贴文字（205px）；事件绑在**元素**上 | 交互改绑**虚线框**（pv-box，pointer-events: auto）——所见即所得，只有框内可操作 |
| 预览里鼠标移到虚线框外显示"移动标识" | CSS `#side-panel > * { cursor: move }` 给元素设了 move——块级拉伸区全显示 move | 元素上移除 `cursor: move`（保留 touch-action/user-select），move 只留虚线框——框内 move / 框外默认（板端实测） |
| 预览 iframe 一直加载旧版页面（修复后仍异常） | demo 静态服务（python http.server）**无 Cache-Control 头**，浏览器缓存旧 index.html/JS | 自定义 http.server 加 `Cache-Control: no-cache, no-store` + Pragma/Expires（systemd 换 ExecStart） |
| 同步 index.html 后预览全没 | 误把**开发版 index.html**（引用 `/src/main.js`，板子无 src）当成部署文件 | 正确同步 **dist/index.html**（引用构建产物 index-*.js）；以后部署统一走 `scripts/deploy.py` |

## 相关

- [[开发板和电脑端通过RNDIS通信]]
- [[Kiosk部署完成]]
