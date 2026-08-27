# CSS & 视觉布局适配避坑指南

移动端屏幕尺寸碎片化、像素密度（DPR）差异大，且存在刘海屏、动态地址栏、滚动链条传播等渲染机制。本模块总结了 CSS 领域的 19 大核心坑位与最佳实现。

---

## 1. 移动端视口与动态高度 (100vh 截断问题)

在 iOS Safari 和 Android Chrome 中，`100vh` 包含了底部动态伸缩的工具栏/导航条，导致 `height: 100vh` 的内容底部被遮挡。

```css
/* 方案 1: 现代 CSS Dynamic Viewport Units (推荐) */
.full-screen-container {
  min-height: 100vh;   /* 基础降级 */
  min-height: 100dvh;  /* Dynamic Viewport Height: 随浏览器工具栏展开/折叠实时动态计算 */
}

/* 方案 2: CSS 变量 + JS 动态注入 (兼顾超旧内核) */
/* 
JS: 
const setVh = () => {
  document.documentElement.style.setProperty('--vh', `${window.innerHeight * 0.01}px`);
};
setVh();
window.addEventListener('resize', setVh);
*/
.full-screen-container-fallback {
  height: 100vh;
  height: calc(var(--vh, 1vh) * 100);
}
```

---

## 2. 全面屏与刘海屏安全区适配 (Safe Area)

全面屏设备（iPhone X+、部分 Android 打孔屏）具有顶部刘海、灵动岛以及底部 Home Indicator 虚拟横条。

```html
<!-- 第一步：HTML 开启 viewport-fit=cover -->
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
```

```css
/* 第二步：CSS 使用 env() 和 constant() 适配安全区 */
.page-container {
  /* 顶部适配 (避免被状态栏/刘海遮挡) */
  padding-top: constant(safe-area-inset-top); /* iOS 11.0 - 11.2 */
  padding-top: env(safe-area-inset-top);      /* iOS 11.2+ */

  /* 底部适配 (固定底栏、吸底按钮必加) */
  padding-bottom: constant(safe-area-inset-bottom);
  padding-bottom: env(safe-area-inset-bottom);
}

/* 固定定位底栏的正确写法 */
.fixed-bottom-bar {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  /* 基础内边距 + 安全区内边距 */
  padding-bottom: calc(12px + env(safe-area-inset-bottom));
  background-color: #ffffff;
}
```

---

## 3. 高清屏 Retina 1px 细边框 (0.5px 问题)

在 DPR >= 2 的屏幕上，直接写 `border: 1px solid #ddd` 会呈现 2px 或 3px 物理像素的粗线。

### 方案 A：伪元素 + Transform Scale (最通用、支持圆角)

```css
.hairline-border {
  position: relative;
}

/* 单条下边框 */
.hairline-border::after {
  content: "";
  position: absolute;
  left: 0;
  bottom: 0;
  width: 100%;
  height: 1px;
  background-color: #e5e5e5;
  transform: scaleY(0.5);
  transform-origin: 50% 100%;
  pointer-events: none;
}

/* 全包围边框 (带圆角) */
.hairline-box {
  position: relative;
}
.hairline-box::after {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  width: 200%;
  height: 200%;
  border: 1px solid #e5e5e5;
  border-radius: 16px; /* 原型圆角 8px 的 2 倍 */
  transform: scale(0.5);
  transform-origin: 0 0;
  box-sizing: border-box;
  pointer-events: none;
}
```

---

## 4. 消除点击高亮遮罩 (Tap Highlight)

移动端点击可交互元素时，Webkit 默认会出现一层灰色或半透明的阴影高亮框。

```css
* {
  -webkit-tap-highlight-color: transparent;
  /* 部分旧 Android 需要额外加 */
  -webkit-tap-highlight-color: rgba(0, 0, 0, 0);
}
```

---

## 5. 禁止滚动链条传播 (Chaining) 与弹性回弹控制

当子容器滚动到尽头时，继续滑动会导致整个外部页面被带动滚动。

```css
.scroll-modal-content {
  overflow-y: auto;
  /* contain: 阻止当前元素的滚动连带影响祖先容器 */
  overscroll-behavior: contain;
  /* 开启 iOS 惯性滚动 */
  -webkit-overflow-scrolling: touch;
}
```

---

## 6. 禁止长按弹出系统菜单 & 选中文本保护

避免用户在长按图片时弹出系统保存菜单，或在快速点击操作时意外选中文本。

```css
/* 全局禁用长按菜单与文本选中 */
body, div, button, img {
  -webkit-touch-callout: none; /* 禁用长按呼出系统菜单 (iOS) */
  -webkit-user-select: none;   /* 禁用文本选定 (iOS Safari) */
  user-select: none;
}

/* ⚠️ 必须放开输入表单，否则会导致光标不可选、无法粘贴文本 */
input, textarea {
  -webkit-user-select: auto;
  user-select: auto;
}

/* 禁止图片被长按拖拽 */
img {
  -webkit-user-drag: none;
  pointer-events: auto;
}
```

---

## 7. 禁止横竖屏切换时系统自动调整字号

```css
html, body {
  -webkit-text-size-adjust: 100% !important;
  text-size-adjust: 100% !important;
}
```

---

## 8. 重置 iOS 表单元素默认圆角、阴影与原生外观

iOS 上的 `<input>`, `<button>`, `<select>` 会自带高光渐变、内阴影及大圆角。

```css
input, textarea, button, select {
  -webkit-appearance: none; /* 清除 iOS 原生控件皮肤 */
  appearance: none;
  border-radius: 0;         /* 清除 iOS 默认圆角 */
  box-shadow: none;         /* 清除 iOS 默认内阴影 */
  outline: none;
  background-color: transparent;
}
```

---

## 9. 美化滚动条 (隐藏或定制)

移动端通常需要隐藏页面或横向滑动 tab 的原生滚动条以提升视觉整洁度。

```css
/* 隐藏 webkit 浏览器滚动条 */
.hide-scrollbar::-webkit-scrollbar {
  display: none;
  width: 0;
  height: 0;
}
.hide-scrollbar {
  -ms-overflow-style: none; /* IE 10+ */
  scrollbar-width: none;    /* Firefox */
}
```

---

## 10. Placeholder 占位符排版与居中问题

在 iOS 上，若 input 设置了过大的 `line-height`，会导致 placeholder 上下偏移或文字被截断。

```css
/* 推荐做法：input 自身 line-height 设为 normal，通过 padding 撑开高度 */
.custom-input {
  height: 44px;
  line-height: normal;
  padding: 10px 12px;
  font-size: 14px;
  box-sizing: border-box;
}

/* 占位符颜色与样式定制 */
.custom-input::-webkit-input-placeholder {
  color: #bfbfbf;
  font-size: 14px;
}
.custom-input::placeholder {
  color: #bfbfbf;
  font-size: 14px;
}
```

---

## 11. 开启 GPU 硬件加速与动画防闪烁

复杂动画或吸顶滑动时可能出现掉帧、卡顿或偶发白屏。

```css
.accelerate-gpu {
  transform: translate3d(0, 0, 0);
  -webkit-transform: translate3d(0, 0, 0);
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
  perspective: 1000;
  -webkit-perspective: 1000;
}

/* 现代 CSS 属性告知合成器 (避免滥用，只加在运动元素上) */
.animated-box {
  will-change: transform, opacity;
}
```

---

## 12. iOS `position: sticky` 失效避坑

`position: sticky` 在移动端极其常用，但在以下情况会无声失效：
1. **父级/祖先元素存在 `overflow: hidden / auto / scroll`**：会导致粘性定位的作用域被限制在该容器内，无法相对视口 sticky。
2. **未声明触发阈值**：必须同时声明 `top: 0`、`bottom: 0` 等至少一个定位方向。
3. **父容器高度不足**：Sticky 元素的活动空间受限于其直接父元素的高度。

```css
.sticky-header {
  position: -webkit-sticky;
  position: sticky;
  top: 0;
  z-index: 100;
}
```

---

## 13. Android 小字号单行居中偏上问题

在 Android 机型中，当字号小于 12px 时，设置 `height == line-height` 会因为字体基线计算导致文字向上偏移 1~2px。

```css
/* 解决方案 1: 使用 Flexbox 居中代替 line-height */
.tag {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  height: 20px;
  padding: 0 6px;
  font-size: 11px;
}

/* 解决方案 2: 整体放大一倍后 scale(0.5) */
.mini-tag {
  font-size: 20px;
  height: 36px;
  transform: scale(0.5);
  transform-origin: 0 0;
}
```

---

## 14. 文本换行与多行截断标准写法

```css
/* 单行文本截断省略 */
.ellipsis-single {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

/* 多行文本截断省略 (标准 WebKit 方案) */
.ellipsis-multiline {
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 2; /* 截断行数 */
  overflow: hidden;
  word-break: break-all; /* 防止长字母/数字不换行直接撑破容器 */
}
```

---

## 15. 移动端自适应布局最佳方案

```css
/* 1. 现代项目优先推荐配合 PostCSS 插件转 vw */
/* 
postcss.config.js -> postcss-px-to-viewport:
viewportWidth: 375, // 设计稿基准 375px
unitPrecision: 3,
viewportUnit: 'vw'
*/

/* 2. 背景图自适应 */
.responsive-bg {
  background-repeat: no-repeat;
  background-position: center center;
  background-size: cover; /* 或 contain / 100% 100% */
}

/* 3. 高清屏高清背景图切换 */
.icon-logo {
  background-image: url('logo@2x.png');
}
@media (-webkit-min-device-pixel-ratio: 3), (min-resolution: 3dppx) {
  .icon-logo {
    background-image: url('logo@3x.png');
  }
}
```
