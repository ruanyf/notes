# worker 静态资源

如果您同时配置了静态资产和 Worker 脚本，Cloudflare 会首先尝试提供与传入请求匹配的静态资产。您可以在 [HTML 处理文档](https://developers.cloudflare.com/workers/static-assets/routing/advanced/html-handling/)中了解更多关于我们如何匹配资产的信息。

如果未找到合适的静态资产，Cloudflare 将调用您的 Worker 脚本。

如果存在 Worker 脚本（`main`属性），并且配置了 `assets.not_found_handling`，以及使用了 compatibility flag 里面的 `assets_navigation_prefers_asset_serving`（或者 compatibility date 设成of `2025-04-01` 或更大），那么导航请求不会触发 Worker 脚本。导航请求是带有 `Sec-Fetch-Mode: navigate` HTTP 头的请求。 

在配置文件中设置优先返回静态资源，而不调用 worker 脚本。

```toml
compatibility_date = "2021-09-14"
compatibility_flags = [ "assets_navigation_prefers_asset_serving" ]
```

如果没有此标志，运行时将继续对任何与静态资产不完全匹配的请求应用调用 Worker 脚本（如果存在）的旧行为。

设置 `assets.run_worker_first = true` 时，此兼容性标志无效。 `assets.run_worker_first = true` 设置可确保 Worker 脚本在任何资产服务逻辑之前执行。
## 路由行为

默认情况下，如果请求的 URL 与静态资源目录中的文件匹配，则会提供该文件，而无需调用 Worker 代码。

如果未找到匹配的资源且存在 Worker 脚本，则该请求将由 Worker 处理。Worker 可以返回响应，也可以选择使用[资源绑定](https://developers.cloudflare.com/workers/static-assets/binding/) （例如 `env.ASSETS.fetch(request)` ）再次延迟到静态资源。

如果不存在 Worker 脚本，则返回 `404 Not Found` 响应。

## 配置文件

项目根目录下需要一个 [Wrangler 配置文件](https://developers.cloudflare.com/workers/wrangler/configuration/) （`wrangler.jsonc` 、 `wrangler.json` 或 `wrangler.toml` ）。

该文件必须有两个字段。

- [`name`](https://developers.cloudflare.com/workers/wrangler/configuration/#inheritable-keys) ：将此设置为您希望部署到的 Worker 的名称。
-  [`compatibility_date`](https://developers.cloudflare.com/workers/configuration/compatibility-dates/) ：如果您已使用 [Pages Functions](https://developers.cloudflare.com/pages/functions/wrangler-configuration/#inheritable-keys) ，请将其设置为与那里配置的日期相同的日期。否则，请将其设置为当前日期。

```toml
name = "my-worker"

compatibility_date = "2025-04-01"
```

`main`字段用来设置 worker 脚本主入口。

```toml
name = "my-spa"

main = "src/index.js"

compatibility_date = "2025-01-01"
```

### assets.directory

如果 worker 项目有静态资源，必须要有`[assets]`部分，其中设置静态资源目录。

```toml
[assets]

directory = "./dist/client/"
```

如果您的 Worker 仅包含资源文件而没有 Worker 脚本，则应从配置文件中移除 `"binding": "ASSETS"` 字段，因为该选项仅在 Worker 脚本文件由 `"main"` 属性指定时才有效。

### assets.binding

如果 worker 脚本需要读取静态资源，则需要使用`binding`进行绑定。

```toml
[assets]

directory = "./dist"

binding = "ASSETS"
```

binding 属性的值就是 env 环境变量的绑定值，比如`binding = "ASSETS"`就意味着可以从 env.ASSETS 上面读取静态资源。

下面是一个用法实例。

```javascript
// index.js

export default {

  async fetch(request, env) {

	const url = new URL(request.url);

	if (url.pathname.startsWith("/api/")) {

		return new Response(JSON.stringify({ name: "Cloudflare" }), {

	headers: { "Content-Type": "application/json" },

	});

	}

	return env.ASSETS.fetch(request);

},

};
```

### assets.not_found_handling

需要设置如何处理 404 报错。分为两种情况，一种是 SPA，会将所有请求匹配到根目录 ( `/` )，这样您就可以捕获 `/about` 或 `/help` 等 URL，并在 SPA 中对其进行响应。

```toml
[assets]

directory = "./dist/client/"

not_found_handling = "single-page-application"
```

另一种是指定 404 页面，系统会自动寻找最近的 404.html。这意味着不同的目录可以有不同的 404.html，比如 `/blog/404.html` 和 `/404.html`。

```toml
[assets]

directory = "./dist/client/"

not_found_handling = "404-page"
```

### assets.run_worker_first

run_worker_first 设置收到请求后，是否首先运行 worker 脚本，而不是直接读取静态资源。

```toml
[assets]

directory = "./public/"

binding = "ASSETS"

run_worker_first = true
```

该属性也可以进行细粒度控制。


```toml
[assets]

directory = "./dist/"

not_found_handling = "single-page-application"

binding = "ASSETS"

run_worker_first = [ "/api/*", "!/api/docs/*" ]
```

该属性可以设置具体路径，只要满足该路径，就优先运行 worker 脚本。

下面是脚本实例。

```javascript
export default {

async fetch(request, env) {

  const url = new URL(request.url);

  if (url.pathname === "/api/name") {

    return new Response(JSON.stringify({ name: "Cloudflare" }), { headers: { "Content-Type": "application/json" },});

  }

  return new Response(null, { status: 404 });

},

};
```

### assets.html_handling

是否自动在请求尾部添加斜杠，默认值为 auto-trailing-slash，也可以设成 `force-trailing-slash` 或 `drop-trailing-slash` 或 `none`。

```toml
[assets]

directory = "./dist/"

not_found_handling = "404-page"

html_handling = "auto-trailing-slash"
```