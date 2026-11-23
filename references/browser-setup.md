# 浏览器方案适配（references/browser-setup.md）

> 闲鱼是重前端 SPA，**纯 HTTP 拿不到卡片数据**，必须有真浏览器。
> 但「用哪个浏览器」在不同环境里差别很大。本文件给出**降级链**：按优先级挑第一个可用的，全都不可用才停下问用户。
>
> **一条铁律贯穿全文：必须有头（headless=no）。** 无头浏览器的指纹特征明显，闲鱼风控会拦；有头真 Chrome 实测单次会话 200+ 请求零验证码。

---

## 0. 降级链（按顺序试，第一个可用的就停）

```
① DSH 环境？
   └─ 是 → ego 插件（ego_* 工具）           ← 默认，最优
   └─ 否 ↓
② ego 桌面端已装？（macOS / Windows 均可）
   └─ 是 → ego-browser CLI / nodejs 脚本     ← 次优
   └─ 否 ↓
③ 问用户：你希望用哪个浏览器方案？
   ├─ 「我有 Chrome，用我的」 → 有头 Chrome + CDP 附着
   ├─ 「装 ego 桌面端」      → 引导安装后回到 ②
   └─ 「随便，你选」         → 跳到 ④
   ↓
④ Playwright CLI 有头模式                    ← 兜底
```

**为什么是这个顺序**：

| 层 | 风控表现 | 登录态 | 代价 |
|---|---|---|---|
| ① ego 插件 | **最好**：持久化 profile + 真人浏览器指纹 | 落盘复用，跨会话有效 | 需要 DSH |
| ② ego 桌面端 | **同样好**：同一个引擎，独立进程 | 落盘复用 | 需要装桌面端 |
| ③ 用户自己的 Chrome | **好**：真 Chrome，但**profile 被占用时不能开第二个实例** | 用户本来就有登录态 | 需要用户配合 |
| ④ Playwright 有头 | **中等**：Chromium 指纹 + 自动化标记，需要额外去特征 | 需自己管 userDataDir | 无需装任何东西 |

---

## 1. 第一层 · DSH 的 ego 插件（默认）

**判断依据**：工具列表里有 `ego_*` 系工具（`ego_navigate` / `ego_js` / `ego_click` …）。

**调用姿势**：

```
ego_space_open("default")        # 先开/复用任务空间（persistent profile 模式，别关）
ego_navigate(url)                 # 再导航
ego_js("...")                     # 页内执行
ego_click(selector)               # 点击
ego_cdp("Network.setCookies", {}) # 原生命令（cookie 注入）
```

**三个必须知道的坑**：

1. **先 `ego_space_open` 再 `ego_script`**。直接跑脚本会报 `NAVIGATION_FAILED ... no active tab to attach session`。
2. **长驻任务不要关空间**。persistent profile 模式会把登录态留在磁盘上，关了就丢。
3. **截图可能超时**。页面里有大图（760px 商品图 ×3 就能拖慢）时 `Page.captureScreenshot` 会 timeout → 改用 headless Chrome 直接截（见 §5）。

**能力对照**（ego_* ↔ Playwright）：

| 能力 | ego 插件 | Playwright |
|---|---|---|
| 打开页面 / 切标签 | `ego_space_open` `ego_navigate` `ego_tabs` | `page.goto` / `browser_tabs` |
| 页内执行 JS | `ego_js` `ego_script` | `page.evaluate` |
| 点击（收藏按钮） | `ego_click` | `page.click` |
| 读元素 / 快照 | `ego_read_element` `ego_snapshot` | `locator.innerText()` |
| CDP 原生命令 | `ego_cdp` | `context.addCookies()` / 裸 CDP session |
| 截图 | `ego_screenshot` | `page.screenshot()` |

---

## 2. 第二层 · ego 桌面端（macOS / Windows）

**判断依据**：DSH 里没有 `ego_*` 工具，但本机装了 ego 桌面端。

**怎么确认装了**：

```bash
# macOS
ls /Applications | grep -i ego
# Windows
ls "/c/Program Files" 2>/dev/null | grep -i ego
ls "$LOCALAPPDATA/Programs" 2>/dev/null | grep -i ego
# 通用：CLI 在 PATH 里
which ego-browser
```

**为什么它和插件一样好**：**同一个引擎**，只是进程独立。持久化 profile、真人浏览器指纹、任务空间都保留。

**调用姿势**（CLI + nodejs heredoc）：

```bash
ego-browser nodejs <<'JS'
// page / browser / taskSpaces / site / fetch / cdp 已预加载
await page.goto('https://www.goofish.com/search?q=...');
const data = await page.evaluate(() => { /* 抓卡片 */ });
console.log('@@RESULT@@' + JSON.stringify(data));
JS
```

**平台差异**：

| | macOS | Windows |
|---|---|---|
| 安装位置 | `/Applications/ego*.app` | `%LOCALAPPDATA%\Programs\ego*` 或 `C:\Program Files\ego*` |
| profile 目录 | `~/Library/Application Support/ego-*` | `%APPDATA%\ego-*` |
| 路径写法 | POSIX | **MSYS 风格 `/c/...`，不是 `C:\`** |
| 后台驻留 | launchd / 手动 | 任务栏常驻，**退出会释放 profile 锁** |

**Windows 特有坑**：

- **profile 被占用**时第二个实例起不来 → 先确认桌面端已退出，或改用它的 CDP 端口附着（见 §3）
- 从 DSH 里 `Start-Process` 启动 GUI 会以 Node 模式秒退（`ELECTRON_RUN_AS_NODE=1` 污染）→ 必须先 `Remove-Item Env:\ELECTRON_RUN_AS_NODE`

---

## 3. 第三层 · 问用户（没有 ego，也没有 DSH）

**这一步必须问，不能猜。** 因为「用哪个浏览器」涉及用户的登录态和隐私偏好，猜错代价是丢账号或丢数据。

**问什么（一次问清，别来回）**：

```
需要用一个「有头浏览器」来操作闲鱼（无头会被风控拦）。
你希望用哪种？

A. 我自己有 Chrome，且已经登录闲鱼  → 我附着到你的 Chrome
B. 装 ego 桌面端（macOS/Windows 都有）→ 我给你安装步骤，装完继续
C. 都行，你决定                      → 我用 Playwright 有头模式
```

### 3A · 用户自己的 Chrome（CDP 附着）

**前提**：Chrome 必须带 `--remote-debugging-port` 启动。已经开着的 Chrome **不带**这个参数，附不上去。

```bash
# Windows
"/c/Program Files/Google/Chrome/Application/chrome.exe" \
  --remote-debugging-port=9222 \
  --user-data-dir="$LOCALAPPDATA/Google/Chrome/User Data"

# macOS
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/Library/Application Support/Google/Chrome"
```

然后：

```
ego_cdp 或 Playwright chromium.connectOverCDP("http://127.0.0.1:9222")
```

**风险**：用户可能不想让你碰他的主 profile（里面有所有登录态）。**先说明，再动手。**

### 3B · 引导装 ego 桌面端

给下载页，装完回到 §2。**不要自己下载安装包替用户装**——安装动作应由用户执行。

### 3C · 用户说「随便」

跳到 §4。

---

## 4. 第四层 · Playwright CLI 有头模式（兜底）

**判断依据**：前三层都不可用。

**最小可用脚本**：

```bash
npx playwright install chromium   # 首次
node -e '
const { chromium } = require("playwright");
(async () => {
  const ctx = await chromium.launchPersistentContext("./.xianyu-profile", {
    headless: false,                    // ← 铁律：必须有头
    viewport: { width: 1440, height: 900 },
    args: [
      "--disable-blink-features=AutomationControlled",
      "--no-default-browser-check",
    ],
  });
  const page = ctx.pages()[0] || await ctx.newPage();
  await page.goto("https://www.goofish.com/search?q=...");
  const data = await page.evaluate(() => { /* 抓卡片 */ });
  console.log(JSON.stringify(data));
  // 不要 close：persistent context 关了就丢登录态
})();
'
```

**四个关键设置**：

| 设置 | 作用 |
|---|---|
| `headless: false` | **必须**。无头必被拦 |
| `launchPersistentContext("./.xianyu-profile")` | 登录态落盘，下次不用重登 |
| `--disable-blink-features=AutomationControlled` | 去掉最明显的自动化标记 |
| **不调 `ctx.close()`** | 关了 profile 锁释放，但内存态丢失 |

**Playwright 的额外去特征**（比 ego 差的地方，需要自己补）：

```js
await ctx.addInitScript(() => {
  Object.defineProperty(navigator, 'webdriver', { get: () => undefined });
  Object.defineProperty(navigator, 'plugins', { get: () => [1, 2, 3, 4, 5] });
  Object.defineProperty(navigator, 'languages', { get: () => ['zh-CN', 'zh', 'en'] });
});
```

> **参考实现**：`ai-goofish-monitor` 项目把这套做得比较完整（注入 JS 抹 `navigator.webdriver`、伪造 plugins/languages/maxTouchPoints、覆写 `permissions.query`、全链路随机延迟、Chrome 扩展导出真机指纹）。要深挖可以读它的源码。

---

## 5. 跨方案通用契约（不管用哪层，都必须满足）

这四条是**方法论层**的要求，和具体工具无关：

### 5.1 cookie 注入用 Netscape 格式 → CDP

```js
const list = text.split('\n').filter(l => l && !l.startsWith('#')).map(l => {
  const p = l.split('\t');
  return { name: p[5], value: p[6], domain: p[0], path: p[2],
           secure: p[3] === 'TRUE',
           httpOnly: /<会话类cookie名>/.test(p[5]),
           expires: Number(p[4]) || -1 };
});
await cdp("Network.setCookies", { cookies: list });
```

- 会话类 cookie 设 `httpOnly: true` 是安全的（只挡 JS 读，不影响上行）
- **不要用「重设 sameSite=None」去救失效会话**——试过，无效。失效就请用户重新导出

### 5.2 登录态判定只信 DOM

`cookie 存在` ≠ `会话有效`。判定方法：导航到 `/personal`，读 DOM 是否含 `我的闲鱼` / `我的交易` / `我的收藏` / `账户设置`。

**不要用** `notLoginContainer` 单独判登录（它同时出现在已登录页面的某些状态下）。

### 5.3 翻页以「首条 item id 变了」为硬判据

`&page=N` 完全无效。必须用页面内组件，且**不能只信 UI 反馈**——要对比首条 item id 是否真的变了。

### 5.4 截图超时的绕过

页面里有大图时 `Page.captureScreenshot` 会 timeout。绕过方式（任何环境通用）：

```bash
CHROME="/c/Program Files/Google/Chrome/Application/chrome.exe"
"$CHROME" --headless=new --disable-gpu --hide-scrollbars \
  --window-size=1920,1080 \
  --screenshot="/abs/path/out.png" \
  "file:///abs/path/page.html"
```

> 注意：**这里的 headless 只是用来截图，不用于访问闲鱼**。截图是本地渲染，不涉及风控。

---

## 6. 全都不行时怎么办

如果连 Playwright 都装不上（无 node、无网络、无权限）：

**明确告知用户缺能力，不要伪造结果。**

```
本 skill 的检索与核验部分需要真实浏览器：
闲鱼是重前端 SPA，纯 HTTP 请求拿不到搜索结果卡片（返回的是空壳 + JS）。

当前环境缺少：<具体缺什么>
可选：装 ego 桌面端 / 装 Playwright / 提供一个带 CDP 端口的 Chrome
```

**绝对不要**：
- ❌ 用 `curl` 抓搜索页然后解析 HTML（拿到的是空壳）
- ❌ 根据记忆编造"当前挂单价格"
- ❌ 用上一轮的报价充当本轮结果（挂单会售出/降价）

---

## 7. 环境自检清单（开工前跑一遍）

```
□ 有 ego_* 工具？                        → 走 §1
□ 有 ego-browser CLI？                   → 走 §2
□ 有 Playwright？(npx playwright --version) → 走 §4
□ 有带 CDP 端口的 Chrome？               → 走 §3A
□ 都没有                                 → 走 §3（问用户）/ §6（告知缺能力）
```

**同时确认**：
```
□ headless = false（有头）
□ profile 是持久化的（userDataDir / launchPersistentContext）
□ 不会在任务结束时 close 掉 context
```

---

## 8. 一句话总结

> **引擎可以换，铁律不能换：有头、持久化 profile、不关 context。**
>
> 这三条满足哪一层都行；这三条不满足，用 ego 也会被拦。
