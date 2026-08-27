# JS & 交互逻辑避坑指南

移动端在手势事件、软键盘弹出/收起、日期解析、往返缓存（BFCache）以及微信等特定 WebView 环境中存在大量隐蔽 Bug。本模块总结了 JS 交互领域的 13 大核心坑位与最佳代码实现。

---

## 1. 滚动穿透 (Scroll Penetration / Mask Scroll)

**现象**：打开遮罩弹窗后，在弹窗上滑动，底部的页面内容仍然在跟随滚动。

### 生产级终极方案：Body 动态 Fixed + 滚动记忆还原

```javascript
/**
 * 移动端弹窗防滚动穿透控制器
 */
class ScrollLock {
  static scrollTop = 0;

  /**
   * 锁定页面滚动
   */
  static lock() {
    this.scrollTop = window.pageYOffset || document.documentElement.scrollTop || document.body.scrollTop;
    document.body.style.position = 'fixed';
    document.body.style.top = `-${this.scrollTop}px`;
    document.body.style.width = '100%';
    document.body.style.overflow = 'hidden';
  }

  /**
   * 解锁并恢复先前的滚动位置
   */
  static unlock() {
    document.body.style.position = '';
    document.body.style.top = '';
    document.body.style.width = '';
    document.body.style.overflow = '';
    window.scrollTo(0, this.scrollTop);
  }
}

// 弹窗组件使用示例:
// showModal() -> ScrollLock.lock();
// closeModal() -> ScrollLock.unlock();
```

---

## 2. 软键盘弹起与收起异常 (高度坍塌与页面不归位)

### 痛点 A：iOS 键盘收起后页面底部留白、无法点击
**原因**：iOS 键盘收起后，WebView 视口位置没有自动滚回正确坐标，DOM 渲染错位。

```javascript
/**
 * 修复 iOS input 失焦后页面底部留白死区
 */
function fixIOSKeyboardBlur() {
  const isIOS = /iPhone|iPad|iPod/i.test(navigator.userAgent);
  if (!isIOS) return;

  // 监听所有输入框的 blur 事件
  document.body.addEventListener('focusout', (e) => {
    const tagName = e.target && e.target.tagName;
    if (tagName === 'INPUT' || tagName === 'TEXTAREA') {
      setTimeout(() => {
        const currentScroll = window.pageYOffset || document.documentElement.scrollTop || 0;
        // 微调触发 WebKit 重新计算布局渲染
        window.scrollTo(0, Math.max(currentScroll - 1, 0));
        window.scrollTo(0, currentScroll);
      }, 100);
    }
  });
}
```

### 痛点 B：Android 软键盘弹起挤压 `position: fixed` 底部按钮
**原因**：Android 软键盘弹起会缩小 `window.innerHeight`，导致固定吸底元素被顶到键盘正上方遮挡内容。

```javascript
/**
 * Android 软键盘弹出时隐藏底部固定栏
 * @param {HTMLElement} fixedElement 吸底固定元素
 */
function fixAndroidKeyboardResize(fixedElement) {
  const isAndroid = /Android/i.test(navigator.userAgent);
  if (!isAndroid || !fixedElement) return;

  const originalHeight = window.innerHeight;

  window.addEventListener('resize', () => {
    const currentHeight = window.innerHeight;
    // 当视口高度缩减超过 150px 时，判定为软键盘弹起
    if (originalHeight - currentHeight > 150) {
      fixedElement.style.display = 'none';
    } else {
      fixedElement.style.display = '';
    }
  });
}
```

---

## 3. iOS 日期格式解析返回 `Invalid Date` / `NaN`

**原因**：iOS WebKit 的 `Date` 构造函数不支持短横线连接的日期字符串（如 `2026-08-27` 或 `2026-08-27 12:00:00`）。

```javascript
/**
 * 跨端安全的时间戳/日期解析函数
 * @param {string|number|Date} dateValue
 * @returns {Date}
 */
function safeParseDate(dateValue) {
  if (dateValue instanceof Date) return dateValue;
  if (typeof dateValue === 'number') return new Date(dateValue);
  
  if (typeof dateValue === 'string') {
    // 将 'YYYY-MM-DD' 替换为 iOS 兼容的 'YYYY/MM/DD'
    const safeStr = dateValue.replace(/-/g, '/').replace(/T/g, ' ');
    return new Date(safeStr);
  }
  
  return new Date();
}

// 示例：
const d = safeParseDate('2026-08-27 19:00:00'); // iOS / Android 均正常解析
```

---

## 4. 300ms 点击延迟与点击穿透防护

### 现代 300ms 消除方案
现代移动浏览器（iOS 9.3+、Chrome 32+）只要在 HTML 中添加：
```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```
或在 CSS 中声明：
```css
html, body, button, a {
  touch-action: manipulation; /* 禁用双击缩放手势，直接消除 300ms 点击等待 */
}
```
即可彻底告别庞大冗余的 `fastclick.js`。

### 点击穿透 (Click Penetration) 防御
**现象**：点击遮罩层上的关闭按钮（用 `touchstart` 触发关闭），遮罩瞬间消失后，300ms 后原位置底层的链接被原生 `click` 误触发。

```javascript
/**
 * 解决方案：
 * 1. 统一交互事件：全站交互统一使用 click，不要混用 touchstart 触发 UI 关闭。
 * 2. 或在关闭弹窗时增加 300ms 销毁延迟，等待底层 click 队列消费完毕。
 */
function safeCloseModal(modalElement, callback) {
  modalElement.classList.add('fade-out');
  // 延迟销毁，防止穿透到底层表单或链接
  setTimeout(() => {
    modalElement.style.display = 'none';
    if (callback) callback();
  }, 320);
}
```

---

## 5. BFCache (往返缓存) 导致后退不刷新

**现象**：用户跳转外部页面后点击“返回”，页面停留在旧状态，表单未重置或接口未重新拉取。

```javascript
/**
 * 处理页面从往返缓存恢复时的状态同步
 */
window.addEventListener('pageshow', (event) => {
  // event.persisted 为 true 表示页面来自 BFCache
  if (event.persisted) {
    console.log('Page loaded from BFCache, refreshing critical data...');
    // 重新获取用户鉴权状态或关键数据
    fetchLatestUserInfo();
  }
});
```

---

## 6. 音视频自动播放限制 (Autoplay Policy)

**限制**：移动端浏览器禁止无用户交互（User Gesture）的声音自动播放。

```javascript
/**
 * 安全播放背景音乐 (支持微信环境自动拉起)
 * @param {HTMLAudioElement} audio
 */
function playAudioSafe(audio) {
  // 1. 尝试直接播放 (如果已静音或策略允许)
  const promise = audio.play();
  if (promise !== undefined) {
    promise.catch(() => {
      console.log('Autoplay blocked, waiting for user gesture or WeChat Bridge');
    });
  }

  // 2. 微信内置环境专属 Hack: 监听 WeixinJSBridgeReady
  const wechatPlay = () => {
    audio.play();
    document.removeEventListener('WeixinJSBridgeReady', wechatPlay);
  };
  document.addEventListener('WeixinJSBridgeReady', wechatPlay, false);

  // 3. 用户首次任意点击/触摸时触发播放
  const firstTouchPlay = () => {
    audio.play();
    document.removeEventListener('touchstart', firstTouchPlay);
    document.removeEventListener('click', firstTouchPlay);
  };
  document.addEventListener('touchstart', firstTouchPlay, { once: true });
  document.addEventListener('click', firstTouchPlay, { once: true });
}
```

---

## 7. 微信内置浏览器字体大小缩放防御

当用户在微信设置中放大了全局字体时，可能导致 H5 页面文字折行错乱。

```javascript
/**
 * 禁止 Android 微信修改字体大小
 */
(function () {
  if (typeof WeixinJSBridge === 'object' && typeof WeixinJSBridge.invoke === 'function') {
    handleFontSize();
  } else {
    document.addEventListener('WeixinJSBridgeReady', handleFontSize, false);
  }

  function handleFontSize() {
    // 设置字体大小为默认值
    WeixinJSBridge.invoke('setFontSizeCallback', { fontSize: 0 });
    // 重写字体改变监听
    WeixinJSBridge.on('setting:font_changed', function () {
      WeixinJSBridge.invoke('setFontSizeCallback', { fontSize: 0 });
    });
  }
})();
```

---

## 8. 长按识别二维码失败规避

**关键细节**：
1. **必须使用原生 `<img>` 标签**：微信客户端识别二维码仅能识别 `<img>` 的 `src` 内容。如果使用 `<canvas>` 生成的二维码，必须调用 `canvas.toDataURL('image/png')` 赋值给一个 `<img>` 标签覆盖在上面。
2. **二维码尺寸**：图片尺寸建议不低于 `200px * 200px`，避免像素过低导致识别率下降。
3. **避免遮挡**：确保 `<img>` 标签上方没有 `pointer-events: auto` 的透明遮罩层拦截长按事件。

```javascript
/**
 * 将 Canvas 二维码转为微信可长按识别的图片
 * @param {HTMLCanvasElement} canvas
 * @param {HTMLImageElement} targetImg
 */
function convertCanvasToImage(canvas, targetImg) {
  const dataUrl = canvas.toDataURL('image/png');
  targetImg.src = dataUrl;
  targetImg.style.width = '200px';
  targetImg.style.height = '200px';
  // 确保允许长按事件
  targetImg.style.pointerEvents = 'auto';
}
```

---

## 9. 回到顶部与平滑滚动兼容方案

```javascript
/**
 * 平滑滚动到顶部 (带兼容性降级)
 * @param {HTMLElement|Window} container
 */
function scrollToTop(container = window) {
  try {
    container.scrollTo({
      top: 0,
      behavior: 'smooth'
    });
  } catch {
    // 降级使用 requestAnimationFrame 平滑滚动
    const scrollStep = () => {
      const current = container === window ? window.pageYOffset : container.scrollTop;
      if (current > 0) {
        const next = Math.max(0, current - current / 8);
        if (container === window) {
          window.scrollTo(0, next);
        } else {
          container.scrollTop = next;
        }
        requestAnimationFrame(scrollStep);
      }
    };
    requestAnimationFrame(scrollStep);
  }
}
```

---

## 10. 高性能图片懒加载 (IntersectionObserver)

```javascript
/**
 * 移动端高性能图片懒加载
 */
function initLazyLoad() {
  if ('IntersectionObserver' in window) {
    const observer = new IntersectionObserver((entries, obs) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          const img = entry.target;
          img.src = img.dataset.src;
          img.removeAttribute('data-src');
          obs.unobserve(img);
        }
      });
    }, {
      rootMargin: '100px 0px' // 提前 100px 预加载
    });

    document.querySelectorAll('img[data-src]').forEach((img) => observer.observe(img));
  } else {
    // 降级直接加载
    document.querySelectorAll('img[data-src]').forEach((img) => {
      img.src = img.dataset.src;
    });
  }
}
```
