# 移动端 H5 交付前检查清单与质量分级

本清单用于移动端 H5 页面上线前、代码审查（Code Review）或回归测试时的自查核对。问题按严重程度分为 P0 ~ P3 四个等级。

---

## 🚨 P0 阻断级 (必须修复，否则禁止上线)

| 检查项 | 问题描述与排查方式 | 修复标准 |
| :--- | :--- | :--- |
| **时间格式 NaN** | 在 iOS 上调用 `new Date('2026-08-27')` 出现 `Invalid Date` 或显示 `NaN` | 所有时间字符串必须使用 `.replace(/-/g, '/')` 或统一由时间库解析 |
| **页面无法滚动** | 弹窗关闭后由于未正确解除 `overflow: hidden` 或 `fixed` 导致页面锁死 | 弹窗卸载/关闭回调中必须有成对的 `ScrollLock.unlock()` |
| **输入框被键盘遮挡** | Android/iOS 唤起软键盘后关键输入项或提交按钮被遮挡且无法滑动出来 | 输入区域使用 flex 弹性滚动容器，或失焦/聚焦时触发自动滚动对其 |
| **白屏/内核语法报错** | 在低版本 WebView 中直接使用了未编译的最新语法 (如可选链、空值合并) | Babel / SWC 正确配置移动端 `browserslist: ["iOS >= 11", "Android >= 7"]` |

---

## ⚠️ P1 严重级 (核心交互与功能体验缺陷)

| 检查项 | 问题描述与排查方式 | 修复标准 |
| :--- | :--- | :--- |
| **滚动穿透** | 打开全屏弹窗或抽屉后，在弹窗空白处滑动会带动底层页面滚动 | 使用 Body Fixed 锁定方案并恢复 scrollTop |
| **安全区遮挡** | iPhone 底部横条（Home Indicator）遮挡了吸底按钮、提交栏 | 视口配置 `viewport-fit=cover`，吸底栏添加 `env(safe-area-inset-bottom)` |
| **iOS 点击无响应** | 给非 `<a>`/`<button>` 的普通 `<div>` 绑定 `click` 事件在 iOS Safari 偶尔失灵 | 给元素添加 CSS `cursor: pointer;` 或使用原生语义化标签 |
| **键盘收起底部留白** | iOS 输入框失焦后，页面底部出现大片空白死区 | 监听 `focusout` / `blur` 事件，执行 `window.scrollTo(0, 0)` 触发重绘 |
| **二维码无法长按识别** | 微信内生成二维码为 `<canvas>` 或被遮挡导致无法长按识别 | 将 Canvas 输出为 `<img>` 标签覆盖在上层 |

---

## 🔍 P2 体验级 (视觉与基础交互还原)

| 检查项 | 问题描述与排查方式 | 修复标准 |
| :--- | :--- | :--- |
| **1px 边框过粗** | Retina 屏幕上 1px 边框渲染为 2px/3px 物理像素 | 使用伪元素 `transform: scale(0.5)` 描绘细边框 |
| **点击灰色高亮遮罩** | 点击按钮或链接时瞬间闪过系统默认的灰色高亮方块 | 全局设置 `* { -webkit-tap-highlight-color: transparent; }` |
| **数字被自动识别为电话** | 订单号、验证码、金额被移动端浏览器自动加上蓝色链接下划线 | `<head>` 中添加 `<meta name="format-detection" content="telephone=no" />` |
| **iOS 表单自带原生外观** | `<input>` / `<button>` 在 iOS 上呈现立体圆角与阴影 | CSS 添加 `-webkit-appearance: none; border-radius: 0;` |
| **100vh 动态导航栏遮挡** | 页面设置 `height: 100vh` 在移动端被展开的浏览器地址栏截断 | 使用 `100dvh` 或 `--vh` 动态变量计算视口高度 |
| **移动端 :hover 黏滞** | 点击按钮后 `:hover` 颜色常驻不消失 | 使用 `@media (hover: hover)` 仅为 PC 端启用 hover |

---

## 💡 P3 优化级 (性能与极致打磨)

| 检查项 | 问题描述与排查方式 | 修复标准 |
| :--- | :--- | :--- |
| **长列表/图片性能** | 滚动长列表卡顿、首屏图片加载过大 | 引入 `IntersectionObserver` 懒加载，开启 `transform: translate3d` GPU 合成 |
| **BFCache 后退刷新** | 页面返回上一页时状态未更新（如登录态、订单状态） | 监听 `pageshow` 事件并在 `e.persisted === true` 时静默拉取最新数据 |
| **微信字体被强制放大** | 微信用户设置大号字体导致 H5 局部文字折行错乱 | 注入 `WeixinJSBridge` 禁用字体缩放 |
| **真机调试开关** | 测试环境排查问题困难 | 仅在非生产环境按需引入 `vConsole` 或 `eruda` |
