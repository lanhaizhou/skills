---
name: h5-code-skill
description: "移动端 H5 开发与兼容性避坑指南。当用户进行 H5 页面开发、移动端适配、排查 iOS/Android 兼容性问题、解决滚动穿透/软键盘遮挡/1px 边框/时间格式解析 NaN/点击穿透/调用系统能力（电话、短信、相机）、安全区适配（刘海屏）、微信内置浏览器适配或进行移动端代码审查（Code Review）时使用。"
argument-hint: "[audit | html | css | js | fix <issue> | --quick]"
---

# H5 移动端开发与兼容性避坑指南 (H5 Code Skill)

移动端开发面临机型碎片化、iOS 与 Android 内核差异、全面屏与软键盘交互等复杂场景。本 Skill 是针对移动端 H5 开发的标准操作手册（SOP），旨在确保生成的移动端代码兼具健壮性、高性能与跨端一致性。

---

## ⚡ 核心铁律 (Iron Laws)

> [!IMPORTANT]
> **NO DIRTY HACKS WITHOUT COMPATIBILITY VERIFICATION**
> （未经过跨端兼容性验证的代码，禁止直接合并或交付）

1. **铁律 1 (时间解析)**：禁止直接使用 `new Date('YYYY-MM-DD')` 解析时间字符串，必须统一替换为斜杠 `/` 分隔（`dateStr.replace(/-/g, '/')`），防止 iOS WebKit 出现 `Invalid Date / NaN`。
2. **铁律 2 (滚动穿透)**：任何弹窗、抽屉（Drawer）或遮罩层，必须配套完整的滚动穿透阻断（Body Fixed + 滚动位置记录与还原），严禁无脑在 `body` 上粗暴 `preventDefault`。
3. **铁律 3 (视口与安全区)**：所有移动端页面必须配置标准 Viewport (`viewport-fit=cover`)，且所有固定吸底组件必须适配 `env(safe-area-inset-bottom)`。
4. **铁律 4 (表单与软键盘)**：表单输入必须同时处理 **iOS 失焦页面重绘回弹** 与 **Android 软键盘弹起挤压吸底元素**。

---

## 🚩 刹车信号 (Red Flags)

在生成或审查代码时，只要出现以下任何一种迹象，**立即强制中断并修正**：

- ❌ 直接在 HTML 或 JS 中引入已淘汰的 `fastclick.js`（现代浏览器只需 `viewport` + `touch-action: manipulation`）。
- ❌ 弹窗遮罩仅设置了 `overflow: hidden` 但没有处理 iOS Safari 背景滑动。
- ❌ 页面高度写死 `height: 100vh` 导致被移动端浏览器动态导航栏截断（未做 `100dvh` 或 `--vh` 兼容）。
- ❌ 给整个 `body` 统一设置 `user-select: none` 却忘记放开 `input, textarea`，导致用户无法正常选词输入。
- ❌ 二维码仅使用 `<canvas>` 渲染，未提供 `<img>` 标签供微信长按识别。

---

## 🎯 引导注意力的自查问题 (Probing Questions)

在编写或审查每一处移动端代码前，主动带着以下问题进行深度思考：

1. **针对布局**：“在 iPhone 刘海屏或带底部横条的机型上，这个固定在底部的按钮会被挡住吗？在高度极小的屏幕上会变形吗？”
2. **针对弹窗**：“当用户在弹窗遮罩区域上下滑动时，后面的页面内容会跟着动吗？弹窗关掉后页面能回到原来的滚动位置吗？”
3. **针对输入**：“用户在 iOS Safari 输入完收起键盘后，页面底部会不会留出一大片无法点击的空白死区？”
4. **针对点击**：“这个按钮在 iOS Safari 上点击时，会不会闪过一块刺眼的灰色背景？非 a/button 标签点击能灵敏触发吗？”

---

## 🔄 标准工作流 (Standard Workflow)

执行任务时，按照以下清单逐步推进，不要跳过关键步骤：

- [ ] **Step 1: 环境与场景识别** ⛔ `BLOCKING`
  - 明确目标运行环境：普通移动浏览器、微信内置 WebView、原生 App 嵌套 Hybrid WebView 还是 PWA。
  - 确认设计稿基准（如 375px / 750px）与适配方案（`vw` 响应式、`rem` 或 Flex 弹性布局）。

- [ ] **Step 2: 加载专项避坑知识** ⚠️ `REQUIRED`
  - 根据涉及的领域，按需加载并阅读对应的 Reference 参考模块：
    - 涉及 Meta 配置、系统拨号/短信/拍照、数字键盘：`Load references/html-pitfalls.md`
    - 涉及 1px 细边框、安全区适配、弹性滚动、Sticky、点击高亮：`Load references/css-pitfalls.md`
    - 涉及 滚动穿透、键盘弹起收起、时间解析、BFCache、微信特有环境：`Load references/js-pitfalls.md`

- [ ] **Step 3: 方案设计与编码落地** ⚠️ `REQUIRED`
  - 针对识别到的坑点应用标准解决方案，避免随手写出反模式代码。
  - 关键跨端逻辑添加明确注释说明兼容意图。

- [ ] **Step 4: 交付前对照检查清单** ⛔ `BLOCKING`
  - 执行 `Load references/checklist.md` 进行 P0 ~ P3 逐级自查。
  - 确保没有任何 P0 / P1 级别的阻断与严重缺陷。

---

## 📚 知识库路由 (References Router)

当需要深入了解具体代码实现与边界场景时，按需读取对应文件：

1. [HTML & 系统能力调用篇](file:///Users/p/Documents/code/skills/h5-code-skill/references/html-pitfalls.md) (`references/html-pitfalls.md`):
   - 拨打电话、短信、邮件、拍照/摄像系统能力调用
   - 禁用浏览器自动识别电话与邮箱
   - 精准弹出数字键盘 (`inputmode="numeric"`)
   - 唤醒 Native App (URL Scheme / Universal Link)
   - Viewport 视口与缩放控制
   - 禁止缓存与 WebApp 增强 Meta

2. [CSS & 视觉布局适配篇](file:///Users/p/Documents/code/skills/h5-code-skill/references/css-pitfalls.md) (`references/css-pitfalls.md`):
   - 移动端视口动态高度 (`100dvh`)
   - 全面屏与刘海屏安全区适配 (`env(safe-area-inset-bottom)`)
   - Retina 高清屏 1px 细边框实现
   - 消除点击高亮灰色遮罩 (`-webkit-tap-highlight-color`)
   - 弹性滚动与禁止滚动链条传播 (`overscroll-behavior`)
   - 禁止长按菜单与文本选中保护
   - iOS 表单原生外观清除与 Sticky 失效排查

3. [JS & 交互逻辑避坑篇](file:///Users/p/Documents/code/skills/h5-code-skill/references/js-pitfalls.md) (`references/js-pitfalls.md`):
   - 终极防滚动穿透控制器 (`ScrollLock`)
   - iOS 键盘收起底部留白重绘与 Android 吸底避让
   - iOS 日期解析 `NaN` 跨端兼容转换
   - 300ms 点击延迟与点击穿透防护
   - BFCache 页面后退刷新状态同步
   - 微信音频自动播放与微信字体缩放防御
   - 二维码长按识别与高精度懒加载

4. [交付前质量检查清单](file:///Users/p/Documents/code/skills/h5-code-skill/references/checklist.md) (`references/checklist.md`):
   - P0 阻断级清单 (时间 NaN、页面锁死、白屏、输入遮挡)
   - P1 严重级清单 (滚动穿透、安全区遮挡、点击无响应、二维码识别)
   - P2 体验级清单 (1px 边框、点击灰底、默认圆角阴影)
   - P3 优化级清单 (GPU 硬件加速、BFCache 恢复、懒加载)

---

## 🎛️ 参数指令速查 (Arguments Guide)

用户调用本 Skill 时可传入参数定制行为：

- `/h5-code-skill audit`：仅执行移动端代码审查并输出 P0~P3 问题报告。
- `/h5-code-skill html`：专注 HTML 结构与 Meta 配置。
- `/h5-code-skill css`：专注 CSS 样式与跨机型屏幕适配。
- `/h5-code-skill js`：专注 JS 交互、滚动穿透与事件兼容。
- `/h5-code-skill fix <issue>`：针对特定移动端 Bug（如 `fix scroll-penetration` 或 `fix ios-keyboard`）直接给出即插即用的修复代码。
- `/h5-code-skill --quick`：快速生成精简版跨端基础脚手架代码。
