# HTML & 系统能力调用避坑指南

移动端浏览器（特别是 WebKit 内核）在 HTML 解析、系统交互与 Meta 配置上有诸多特有机制。本模块总结了 HTML 领域的 8 大核心坑位与标准解决方案。

---

## 1. 系统通讯与功能调用

通过特定协议的链接和 `<input>` 元素可直接调起系统级功能，必须严格保证 URI 格式与属性拼写。

```html
<!-- 1. 拨打电话 -->
<a href="tel:10086">拨打电话</a>

<!-- 2. 发送短信 (可附带初始内容，部分 Android 系统分隔符为 ?，iOS 为 & 或 ?) -->
<a href="sms:10086?body=退订业务">发送短信</a>

<!-- 3. 发送邮件 (支持主题与正文预填) -->
<a href="mailto:support@example.com?subject=意见反馈&body=问题描述...">发送邮件</a>

<!-- 4. 调用系统相册 / 相机 -->
<!-- 仅相册选择单张/多张图片 -->
<input type="file" accept="image/*" multiple />

<!-- 强制直接调起相机拍照 (iOS/Android 支持 capture 属性) -->
<input type="file" accept="image/*" capture="camera" />
<!-- capture="user" (前置摄像头) / capture="environment" (后置摄像头) -->
<input type="file" accept="image/*" capture="environment" />

<!-- 调起系统录像机 -->
<input type="file" accept="video/*" capture="camcorder" />

<!-- 调起系统录音机 -->
<input type="file" accept="audio/*" capture="microphone" />
```

> [!WARNING]
> 部分机型在 `<input type="file" accept="image/*" capture="camera">` 拍照返回后，可能因内存占用过高导致页面被系统回收并强制刷新。应对策略：在打开拍照前通过 `sessionStorage` 备份表单状态。

---

## 2. 忽略系统自动识别

移动端浏览器（特别是 iOS Safari 和微信内置浏览器）会自动将连续数字识别为电话号码、将邮箱地址识别为链接，并自动添加下划线与蓝色字色，破坏设计还原度。

```html
<head>
  <!-- 禁用自动识别电话号码 -->
  <meta name="format-detection" content="telephone=no" />
  <!-- 禁用自动识别邮箱地址 -->
  <meta name="format-detection" content="email=no" />
  <!-- 禁用自动识别地理位置地址 (iOS) -->
  <meta name="format-detection" content="adress=no" />
  
  <!-- 组合写法 (推荐) -->
  <meta name="format-detection" content="telephone=no, email=no, adress=no" />
</head>
```

---

## 3. 精准弹出移动端数字键盘

不同类型的数字输入（电话号码、短信验证码、身份证号、金额）需要匹配不同的软键盘类型以保证输入体验。

| 场景 | 推荐标签配置 | 效果说明 |
| :--- | :--- | :--- |
| **纯数字 (验证码/卡号)** | `<input type="text" inputmode="numeric" pattern="[0-9]*" />` | **推荐**。iOS 和 Android 均弹出纯 9 宫格数字键盘，无符号干扰 |
| **电话号码** | `<input type="tel" />` | 弹出带 `+`、`*`、`#` 符号的拨号键盘 |
| **金额/小数** | `<input type="text" inputmode="decimal" />` | 弹出带小数点 `.` 的数字键盘 |
| **带 X 的身份证号** | `<input type="text" pattern="[0-9Xx]*" />` | 基础数字键盘，允许输入 X |

> [!IMPORTANT]
> 慎用 `<input type="number">`：
> 1. iOS 上可能依然弹出带符号的通用键盘；
> 2. `input.value` 遇到不符合规范的数字时会直接返回空字符串 `""`（如非法字符或超大数字）；
> 3. 会默认出现上下增减微调按钮（需 CSS 清除）。

---

## 4. 唤醒原生 App (URL Scheme 与 Universal Link)

从 H5 唤起 Native 客户端的常见兼容策略：

```javascript
/**
 * 唤醒原生 App 兼容处理
 * @param {string} schemeUrl 例如 'weixin://' 或 'tbopen://'
 * @param {string} downloadUrl 唤醒失败后的下载落地页
 */
function openNativeApp(schemeUrl, downloadUrl) {
  const isIOS = /iPhone|iPad|iPod/i.test(navigator.userAgent);
  const startTime = Date.now();

  // 1. 尝试唤醒
  if (isIOS) {
    // iOS 推荐 location.href
    window.location.href = schemeUrl;
  } else {
    // Android 部分 Webview 拦截 iframe 唤起更平滑
    const iframe = document.createElement('iframe');
    iframe.src = schemeUrl;
    iframe.style.display = 'none';
    document.body.appendChild(iframe);
    setTimeout(() => document.body.removeChild(iframe), 3000);
  }

  // 2. 检查是否唤醒成功（若成功唤醒，页面进入后台，定时器会被挂起推迟执行）
  setTimeout(() => {
    const endTime = Date.now();
    // 如果定时器正常在 2500ms 左右触发，说明没有唤醒 App，跳转下载页
    if (endTime - startTime < 2600) {
      window.location.href = downloadUrl;
    }
  }, 2000);
}
```

> [!NOTE]
> 在 iOS 9+ 上，**Universal Links (通用链接)** 相比 URL Scheme 体验更好（无唤醒确认弹窗、直接跳转且支持原生域名回退）。

---

## 5. Viewport 视口与缩放控制

移动端标准 Viewport 配置模板，兼顾全面屏适配与防误触缩放。

```html
<meta 
  name="viewport" 
  content="width=device-width, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0, user-scalable=no, viewport-fit=cover" 
/>
```

- `width=device-width`：视口宽度等于设备物理独立像素宽度。
- `initial-scale=1.0`：初始缩放比例 1:1。
- `user-scalable=no`：禁止用户双击或双指手势缩放。
- `viewport-fit=cover`：**关键属性**。让页面内容填满包含刘海/底栏在内的整个屏幕安全区（Safe Area）。

---

## 6. 禁止浏览器非预期强缓存

在移动端 WebView 或微信内置浏览器中，页面缓存极其顽固。更新静态资源时应配合 Meta 标签及构建版本号 Hash。

```html
<head>
  <meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate" />
  <meta http-equiv="Pragma" content="no-cache" />
  <meta http-equiv="Expires" content="0" />
</head>
```

---

## 7. WebApp / iOS Safari 专属 Meta 增强

当 H5 被添加到主屏幕或以独立 PWA/WebApp 模式运行时：

```html
<head>
  <!-- 启用 WebApp 独立全屏模式 (隐藏 Safari 顶部地址栏与底部导航) -->
  <meta name="apple-mobile-web-app-capable" content="yes" />
  
  <!-- 状态栏外观: default (白底黑字) | black (黑底白字) | black-translucent (半透明沉浸式) -->
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
  
  <!-- 添加到主屏幕后的应用名称 -->
  <meta name="apple-mobile-web-app-title" content="应用名称" />
  
  <!-- 添加到主屏幕后的应用高清图标 -->
  <link rel="apple-touch-icon" href="/path/to/icon-180.png" />
  
  <!-- 针对 Android Chrome 的主题色 -->
  <meta name="theme-color" content="#1890ff" />
</head>
```

---

## 8. 修复 iOS 移动端 `:active` 与 `:hover` 伪类

1. **`:active` 失效问题**：iOS WebKit 默认不触发非链接元素的 `:active` 点击态。
   - **解决方案**：在 `<body>` 上添加一个空的 `ontouchstart` 监听。
   ```html
   <body ontouchstart="">
   ```
2. **`:hover` 黏着问题**：移动端点击元素后，`:hover` 样式会持续保留直到用户点击其他区域。
   - **解决方案**：使用现代媒体查询，仅在支持 hover 的指针设备（如鼠标）上应用 `:hover` 效果。
   ```css
   @media (hover: hover) and (pointer: fine) {
     .button:hover {
       background-color: #40a9ff;
     }
   }
   ```
