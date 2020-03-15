# 错误监控方案

## 前言

因涉及到公司机密，具体代码就不贴出来了。

本文更多的是涉及到思路性的东西，也许无法特别直观，忘谅解。

## 我们需要哪些错误？

### 1、一般错误

是指 `SyntaxError`，`ReferenceError`……`TypeError`等等的错误。这些错误能直接被 `window.onerror = fn` 方式监听到。

当然，这里也包括一些 `异步函数` 中引发的错误，如 `setTimeout()`、`setInterval()` 等等，虽然是异步触发的，但也能被 `window.error = fn` 监听到。

```js
window.onerror = function(errMsg, source, line, col, err) {
  // TODO
};
```

### 2、Promise 错误

这类错误无法被 `window.onerror = fn` 监听到，需要采用 `window.addEventListener('unhandledrejection', fn)` 的方式去监听。

```js
window.addEventListener("unhandledrejection", function(PromiseRejectionEvent) {
  // 目前已知，需要对PromiseRejectionEvent.reason进行切割
  // 切割后能得到 报错文件、行列号
  // 也许有更好的方法，等待探究
});
```

### 3、资源加载错误

比如图片加载错误，也无法用 `window.onerror = fn` 监听到，需要采用 `window.addEventListener('error', fn, true)` 的方式，即使用 `捕获` 的方式去做。

```js
window.addEventListener(
  "error",
  function(Event) {
    // Event.target代表image element
    // 可以从image element取到src
  },
  true
);
```
