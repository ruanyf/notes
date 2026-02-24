# worker 脚本

## 创建 worker 项目

```bash
$ npm create cloudflare@latest -- my-first-worker
$ cd my-first-worker
$ npx wrangler dev
```

开发完成后，部署项目。

```bash
$ npx wrangler deploy
```

预览域名是 `<YOUR_WORKER>.<YOUR_SUBDOMAIN>.workers.dev`。
## worker 简介

Cloudflare Worker 是一种云函数服务，采用 JavaScript 语言，通过 V8 引擎运行。

用户访问网站时，不管请求什么路径，都会调用入口脚本（`package.json`的`main`字段），默认是 `src/index.js`。如果要对不同的路径有不同的返回结果，就需要设置路由。

Worker 脚本采用 ECMAScript 模块格式，默认输出一个对象。

```javascript
export default {
	// ...
}
```

  该对象用来指定各种事件的响应函数，也就是说，它的属性就是各种事件的名字，比如`fetch`属性就是对应`fetch`事件的处理函数。

```javascript
export default {
  async fetch(request, env, ctx) {
    return new Response("Hello World!");
  },
};
```

上面代码中， fetch() 在收到 HTTP 请求时调用。它接受三个参数 [`request`，`env` ， `context`](https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/)。fetch() 是`fetch`事件的响应函数，即发生`fetch`事件时，自动调用该函数。`fetch`事件指的是用户发起请求。

`fetch()`函数接受三个参数。

- request：用户发来的请求，是 Request 的一个实例对象。
- env：环境对象。
- ctx：上下文对象。

`fetch()`返回的是一个 Response 对象实例的 Promise 对象，用户收到的 HTTP 回应就是这个 Response 实例对象。

下面是一个最简单的 Hello World 例子。

```javascript
export default {
/**
* @param {Request} request
* @param {Env} env
* @param {ExecutionContext} ctx
* @returns {Promise<Response>}
*/

async fetch(request) {
  return new Response('Hello worker!', { status: 200 });
}
```

下面的例子是对不同的请求路径，返回不同的结果。

```javascript
import welcome from "welcome.html";

export default {
	async fetch(request, env, ctx) {
		const url = new URL(request.url);
		if (url.pathname === "/api") {
			const data = await import("./data.js");
			return Response.json(data);
		}
		return new Response(welcome, {
			headers: {
				"content-type": "text/html",
			},
		});
	},
};
```

上面示例中，`import`命令可以加载 HTML 文件，放入一个变量，比如`import welcome from "welcome.html"`，标准的 ECMAScript 语法不允许加载这样写。另外，`import()`函数返回的是一个 Promise 对象，所以前面要加 await 命令。

Cloudflare 提供一个[在线练习场](https://workers.cloudflare.com/playground)，可以在那里测试脚本。
## 定时任务

定时任务需要在入口脚本里面设置`scheduled`事件的监听函数。

```javascript
export default {
  async scheduled(controller, env, ctx) {
      console.log("cron processed");
  },
};
```

然后，在 wrangler 配置文件里面设置触发器。

```json
{
"triggers": {
// Schedule cron triggers:
// - At every 3rd minute
// - At 15:00 (UTC) on first day of the month
// - At 23:59 (UTC) on the last weekday of the month

"crons": [
"*/3 * * * *",
"0 15 1 * *",
"59 23 LW * *"
]
}
}
```

也可以为不同环境设置触发器。

```json
{
"env": {
	"dev": {
		"triggers": {
			"crons": [
				"0 * * * *"
			]
		}
	}
}
}
```

删除定时任务，可以清空 crons 数组。
## env 对象

fetch() 方法的第二个参数是 env 对象，表示运行环境，可以读取环境变量。

环境变量的设置方法是写在 wrangler 配置文件的`[vars]`里面。

```json
{
	"$schema": "./node_modules/wrangler/config-schema.json",
	"name": "my-worker-dev",
	"vars": {
		"API_HOST": "example.com",
		"API_ACCOUNT_ID": "example_user",
		"SERVICE_X_DATA": {
			"URL": "service-x-api.dev.example",
			"MY_ID": 123
		}
	}
}
```

`[vars]`可以从 env 读取。

```javascript
export default {
  async fetch(request, env, ctx) {
      return new Response(`API host: ${env.API_HOST}`);
  },
};
```
另一种方法是从  [`cloudflare：workers`](https://developers.cloudflare.com/workers/runtime-apis/bindings/#importing-env-as-a-global) 导入 env，读取环境变量。

```javascript
import { env } from "cloudflare:workers";

// Access environment variables at the top level
const apiHost = env.API_HOST;

export default {
	async fetch(request) {
		return new Response(`API host: ${apiHost}`);
	},
};
```

wrangler 配置文件的 env 字段可以设置不同的环境，为同一个 worker 指定不同的配置。

```json
{
	"$schema": "./node_modules/wrangler/config-schema.json",
	"name": "my-worker-dev",
	// top level environment
	"vars": {
		"API_HOST": "api.example.com"
	},
	"env": {
		"staging": {
			"vars": {
				"API_HOST": "staging.example.com"
			}
		},
		"production": {
			"vars": {
				"API_HOST": "production.example.com"
			}
		}
	}
}
```

环境变量还可以通过全局 `process.env` 访问。

## ctx 对象

ctx 对象表示上下文（context）

- `ctx.waitUntil()` 延长了 worker 的运行时间，在不阻止返回回复的情况下完成工作，且在回复返回后仍可能继续。它接受一个 Promise，Worker 运行时会继续执行该`Promise` ，即使 Worker 已经返回了响应。

它适合将事件发放给外部分析提供商，以及使用[缓存 API 将项目放入缓存](https://developers.cloudflare.com/workers/runtime-apis/cache/)。

`waitUntil`时间限制为30秒

```javascript
export default {
  async fetch(request, env, ctx) {
      // Forward / proxy original request
          let res = await fetch(request);
    // Add custom header(s)    
    res = new Response(res.body, res);    
    res.headers.set("x-foo", "bar");
    // Cache the response
    // NOTE: Does NOT block / wait
    ctx.waitUntil(caches.default.put(request, res.clone()));
    // Done
        return res;
    },
};
```

-  `CTX.passThroughOnException()`：允许 Worker 在抛出未处理异常时，失败[打开 ↗](https://community.microfocus.com/cyberres/b/sws-22/posts/security-fundamentals-part-1-fail-open-vs-fail-closed) ，并将请求传递给起源服务器。

```javascript
export default {
  async fetch(request, env, ctx) {
  // Proxy to origin on unhandled/uncaught exceptions
    ctx.passThroughOnException();
    throw new Error("Oops");
  },
};
```

- `CTX.props` 当你的 worker 被另一个 worker 调用时，`ctx.props` 可以提供该 worker 的信息。
- CTX.exports 提供当前脚本顶层 exports 的输出接口。
## 秘密变量

秘密变量可以放入 `.dev.vars` 文件或 `.env` 文件，与 Wrangler 配置文件同一个目录中。

这两个文件不要两者兼用。如果你定义了一个 `.dev.vars` 文件，那么 `.env` 文件中的值在本地开发时不会包含在 `env` 对象中。

```
SECRET_KEY="value"
API_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9"
```

要为每个环境设置不同的秘密文件，可以创建名为 `.dev.vars.<environment-name>` 或 `.env.<environment-name>` 的文件。加载时，所有 .dev.vars 文件不会合并，所有 .env 文件则会合并。

下面是一个示例，Cloudflare Workers 项目中使用以下 [`wrangler secret`](https://developers.cloudflare.com/workers/wrangler/commands/#secret) 命令创建一个秘密：

```
wrangler secret put SECRET_NAME
```

然后，使用以下代码片段获取代码中的秘密值：

```javascript
const secretValue = env.SECRET_NAME;
```

## 全局对象

worker 脚本可以直接使用、不必 import 的全局对象

- [caches](https://developers.cloudflare.com/workers/runtime-apis/cache/)
- [console](https://developers.cloudflare.com/workers/runtime-apis/console/)
- TextEncoder()
- Decode()
- EventSource()
- [fetch()](https://developers.cloudflare.com/workers/runtime-apis/fetch/)
- [Headers()](https://developers.cloudflare.com/workers/runtime-apis/headers/)
- [HTMLRewriter()](https://developers.cloudflare.com/workers/runtime-apis/html-rewriter/)
- MessageChannel()：在同一个 worker 的不同部分之间传递消息
- 