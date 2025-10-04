# worker ctx 对象

## 方法

`ctx.waitUntil()` 可延长 Worker 的生命周期，让您能够执行工作而不会阻塞返回响应，并且响应返回后仍可继续执行。它接受一个 `Promise` ，即使 Worker 的 [handler](https://developers.cloudflare.com/workers/runtime-apis/handlers/) 返回了响应，Workers 运行时仍会继续执行该 Promise 。

你可以多次调用 `waitUntil()` 。与 `Promise.allSettled` 类似，即使传递给某个 `waitUntil` 调用的 Promise 被拒绝，传递给其他 `waitUntil()` 调用的 Promise 仍会继续执行。

```javascript
export default {
  async fetch(request, env, ctx) {
    // Forward / proxy original request
    let res = await fetch(request);

    // Add custom header(s)
    res = new Response(res.body, res);
    res.headers.set('x-foo', 'bar');

    // Cache the response
    // NOTE: Does NOT block / wait
    ctx.waitUntil(caches.default.put(request, res.clone()));

    // Done
    return res;
  },
};
```
