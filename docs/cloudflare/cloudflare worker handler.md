# Cloudflare Worker 的 Handler 函数

Handler 是 worker 的事件响应函数。

## fetch 函数

fetch 函数响应外部的 HTTP 请求。

```javascript
export default {
  async fetch(request, env, ctx) {
    return new Response('Hello World!');
  },
};
```

- `request` Request：The incoming HTTP request.
- `env` object：The [bindings](https://developers.cloudflare.com/workers/configuration/environment-variables/) available to the Worker. 
- `ctx.waitUntil(promisePromise)` : void 回复响应后，依然会执行的 Promise
- `ctx.passThroughOnException()` : void
## scheduled 函数

```javascript
export default {
  async scheduled(controller, env, ctx) {
    return await fetch("https://example.com", {
      headers: {
        "X-Source": "Cloudflare-Workers",
      },
    });
  },
};
```

- `controller.cron` string：The value of the [Cron Trigger](https://developers.cloudflare.com/workers/configuration/cron-triggers/) that started the `ScheduledEvent`.
- `controller.type` string：The type of controller. This will always return `"scheduled"`.
- `controller.scheduledTime` number：The time the `ScheduledEvent` was scheduled to be executed in milliseconds since January 1, 1970, UTC. It can be parsed as `new Date(controller.scheduledTime)`.