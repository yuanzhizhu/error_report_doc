# 错误监控方案

## 一、前言

最近在做的一个项目，有感而发。因涉及到一切商业信息，具体代码就不贴出来了。

本文更多的是涉及到思路性的东西，也许无法特别直观，忘谅解。

## 二、我们需要哪些错误？

### 0、接口错误

比如 `XMLHTTPRequest` 错误，或者 `fetch` 错误（fetch 错误暂未实践）。

需要通过劫持 `XMLHTTPRequest` 上的部分方法去实现监听。

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

## 三、接口错误怎么处理？

接口错误只需要单纯地捕获 Promise 错误吗？

答案当然是“否”！

接口错误 和 Promise 错误，其实是两种情况～

常见的处理接口错误的方法步骤是：

1、重写 XMLHTTPRequest 原型上的 `xhr.prototype.open` 记录接口的 url、重写 `xhr.prototype.send` 记录请求的 data，重写 `xhr.prototype.onStateChange` 记录服务端返回。

2、重写 `xhr.prototype.error` 方法。该 error 方法会在响应码为 `40x`、`50x`的时候触发，并使用该方法上报 `request data` 、 `response data` 、 `接口url`。

3、与后端程序员约定，当服务端错误时，做一个怎么样的返回呈现。

因为有些后端程序员，当服务端出错时，喜欢给你返回个 500 的状态码；

而有些程序员，无论出不出错，都是返回 200 的状态码，但是会是如下格式的 JSON：

```js
// 成功时：
{
  code: 0,
  message: 'success',
  data: { /* some data here */ }
}

// 失败时：
{
  code: -1,  // 非0状态码，比如-1
  message: 'error message', // 错误信息
}
```

若为以上约定， 则 `xhr.prototype.onerror` 被触发到的，往往就只有 `40x` 的情况了。因为当服务端错误时，是在 `xhr.protype.onStateChange` 中，去判断响应 JSON 中的 `code`。当 `code` 不为 `0` 的时候，才判断为错误，进而上报 `request data` 、 `response data` 、 `接口url`。

## 四、接口错误监控，有多大意义呢？

上面的“三”所提到的内容很简单。那么到底是否有意义呢？

事实上，对于一个具备完整监控的后端服务，前端接口错误监控的作用，是被弱化了的。

举个最简单的例子，前端接口错误监控抓取的 `request data` 、 `response data` 、 `接口url`，后端日志会没记录吗？甚至后端日志能够查的更清晰，比如调用者是谁，等等，都能查到。

## 五、重谈跨域

为什么要重新谈跨域呢？

因为，我们现在为了提高加载速度，通常把打包好的 js 传到 CDN。

那么，当引入 CDN 上面 js 的时候，通常跨域的。一旦跨域js报错，通过 `window.onerror` 是无法捕获到错误的详细信息的（只有 Script Error）

解决这个问题之前，先说 Why ，再说 How……


