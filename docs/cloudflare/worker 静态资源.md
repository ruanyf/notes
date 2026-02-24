# worker 静态资源

## 静态资源的配置

静态资源需要跟 worker 脚本分开，它的位置在  [Wrangler](https://developers.cloudflare.com/workers/wrangler/configuration/#assets) 配置文件中指定。

```toml
"$schema" = "./node_modules/wrangler/config-schema.json"
name = "my-spa"
main = "src/index.js"
compatibility_date = "2026-02-16"

[assets]

directory = "./dist"
binding = "ASSETS"
```

绑定静态资源后，Worker 脚本可以直接使用静态资源。

```javascript
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

## 静态资源的路由

如果同时配置了静态资源和 Worker 脚本，Cloudflare 会首先尝试提供与传入请求匹配的静态资源。如果匹配，并不会调用 worker 脚本。您可以在 [HTML 处理文档](https://developers.cloudflare.com/workers/static-assets/routing/advanced/html-handling/)中了解更多关于我们如何匹配资产的信息。

如果不匹配，未找到合适的静态资源，Cloudflare 将调用 Worker 脚本，外部请求会被传给 Worker 脚本处理。

如果这时没有提供 Worker 脚本，会返回一个 `404 Not Found` 回应。

请求不匹配静态资源时的默认行为，可以在 Wrangler 配置文件里面通过`assets.not_found_handling` 改变。

 - [`not_found_handling = "single-page-application"`](https://developers.cloudflare.com/workers/static-assets/routing/single-page-application/): 如果静态资源请求没有匹配的文件，则返回 `200 OK`  和 index.html。这主要适用于单页应用的场合。
 - [`not_found_handling = "404-page"`](https://developers.cloudflare.com/workers/static-assets/routing/static-site-generation/#custom-404-pages): 设置对于不匹配静态资源的请求，返回 `404 Not Found` 回应，以及最近的`404.html` 。

```toml
[assets]
directory = "./dist"
not_found_handling = "single-page-application"
```

如果希望 Worker 脚本在静态资源之前执行，则设置 `run_worker_first`。如果设为  `true`，则 Worker 脚本在所有请求之前执行，如果设为一个路由模式的数组，则 Worker 脚本会选择性地首先执行。

```toml
name = "my-spa-worker"
compatibility_date = "2026-02-16"
main = "./src/index.ts"

[assets]
directory = "./dist/"
not_found_handling = "single-page-application"
binding = "ASSETS"
run_worker_first = [ "/api/*", "!/api/docs/*" ]
```

静态资源一旦被请求，就会进入 Cloudflare 缓存服务器。

如果存在 Worker 脚本（`main`属性），并且配置了 `assets.not_found_handling`，以及使用了 compatibility flag 里面的 `assets_navigation_prefers_asset_serving`（或者 compatibility date 设成of `2025-04-01` 或更大），那么导航请求不会触发 Worker 脚本。导航请求是带有 `Sec-Fetch-Mode: navigate` HTTP 头的请求。 

在配置文件中设置优先返回静态资源，而不调用 worker 脚本。

```toml
compatibility_date = "2021-09-14"
compatibility_flags = [ "assets_navigation_prefers_asset_serving" ]
```

如果没有此标志，运行时将继续对任何与静态资产不完全匹配的请求应用调用 Worker 脚本（如果存在）的旧行为。

设置 `assets.run_worker_first = true` 时，此兼容性标志无效。 `assets.run_worker_first = true` 设置可确保 Worker 脚本在任何资产服务逻辑之前执行。

## 新建项目

```bash
$ npm create cloudflare@latest -- my-site
$ cd my-site
$ npx wrangler dev
```

部署项目

```bash
$ npx wrangler deploy
```

## Wrangler 配置文件

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

`env.ASSETS.fetch()` 的用法如下

```typescript
fetch(request: Request | URL | string): Promise<Response>
```

该方法用来获取当前项目的静态资源，下面是一些例子。

- `env.ASSETS.fetch(request)`
- `env.ASSETS.fetch(new URL('https://assets.local/my-file'))` 
- `env.ASSETS.fetch('https://assets.local/my-file')`

方法参数里面的 URL，域名不重要，会被忽略，系统只使用路径进行匹配。
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

run_worker_first 设置收到请求后，是否首先运行 worker 脚本，而不是直接读取静态资源。默认值是 run_worker_first = false ，如果设为 true，则会首先执行脚本。

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

该属性可以设置具体路径，只要满足该路径，就优先运行 worker 脚本。除了具体路径，还可以使用通配符`*`和否定运算符`!`。通配符`*`包含子目录，否定运算符`!`则具有更高的优先级。

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

数组里面模式顺序并不重要。
### assets.html_handling

是否自动在请求尾部添加斜杠，默认值为 auto-trailing-slash，也可以设成 `force-trailing-slash` 或 `drop-trailing-slash` 或 `none`。

```toml
[assets]
directory = "./dist/"
not_found_handling = "404-page"
html_handling = "auto-trailing-slash"
```

auto-trailing-slash 表示请求正常文件时不添加尾部斜杠，请求索引文件 index.html 会添加尾部斜杠。

默认情况下，文件后缀名 .html 会被省略，如果希望保留，可以将 html_handling 设为 none。
## .assetsignore 文件

静态资源的根目录下可以放置 .assetsignore 文件，格式与 .gitignore 文件相同，用来告诉 Wrangler 哪些静态资源不必上传到 Cloudflare 服务器。

## HTTP 头

静态资源响应的默认响应头，可以通过在静态资源目录中创建一个名为 `_headers` 的纯文本文件（无扩展名）来覆盖、删除或添加。该文件本身不会作为静态资源提供，而是由 Workers 解析，其规则将应用到静态资源响应上。

`_headers` 文件中定义的头覆盖了 Cloudflare 通常发送的内容。

`_headers` 文件中定义的自定义头部不会应用于你的 Worker 代码生成的响应，即使请求 URL 与 `_headers` 中定义的规则相符。

`_headers` 文件规则定义为多行块。块的第一行是 URL 或 URL 模式，规则的头应该应用在这里。在下一行，必须写入缩进的头部名称和头部值列表：

```
[url]  [name]: [value]
```

支持使用绝对 URL，但请注意绝对 URL 必须以 `https` 开头，且不支持指定端口。`_headers` 规则在匹配请求时忽略了接收请求的端口和协议。例如，像 `https://example.com/path` 这样的规则会与 `other://example.com:1234/path` 请求匹配。

你可以在后续行中定义任意数量的 `[name]： [value]` 对。

```
# This is a comment
/secure/page
	X-Frame-Options: DENY
	X-Content-Type-Options: nosniff
	Referrer-Policy: no-referrer

/static/*
	Access-Control-Allow-Origin: *
	X-Robots-Tag: nosnippet

https://myworker.mysubdomain.workers.dev/*
	X-Robots-Tag: noindex
```

匹配多个规则 URL 模式的入站请求将继承所有规则的头部。

你可以定义最多 100 个头部规则。`_headers` 文件中的每一行都有 2000 个字符的限制。整行，包括间距、头部名和数值，都计入该限制。

如果 `_headers` 文件中一个头部被应用两次，则用逗号分隔符将这些值连接起来。

你可能想删除默认的头部，或者被更普遍规则添加的头部。这可以通过在头部名称前加上感叹号和空格（`！`）来实现。

```
/*
  Content-Security-Policy: default-src 'self';

/*.jpg
  ! Content-Security-Policy
```

[`_redirects`](https://developers.cloudflare.com/workers/static-assets/redirects/) 提供与 URL 匹配功能的相同功能也适用于 `_headers` 文件。但请注意，重定向会先于头部，因此当请求同时匹配重定向和首部时，重定向优先。

### 通配符

匹配时，一个用星号（`*`）表示的溅射图案会贪婪地匹配所有字符。你只能在网址中包含一个 splat。

匹配值可以在头部值中作为 `：splat` 占位符引用。

### 占位符

占位符可以用 `:p laceholder_name` 定义。冒号（`：`）后跟字母表示占位符的开头，后续占位符名称必须由字母数字字符和下划线组成（`：[A-Za-z]\w*`）。每个命名占位符只能引用一次。占位符匹配除分隔符外的所有字符，当分隔符属于主机时，是句号（`.`）或斜杠（`/`），只有在路径的一部分时才可能是前斜杠（`/`）。

同样，匹配值也可以用于头部值，并使用 `:p laceholder_name`。

```
/movies/:title  x-movie-name: You are watching ":title"
```

### 示例 跨域原产资源共享（CORS）

为了让其他域从你的工人获取所有静态资产，可以在 `_headers` 文件中添加以下内容：

```
/*
  Access-Control-Allow-Origin: *
```

### 示例 防止 workers.dev 网址出现在搜索结果中

[谷歌 ↗](https://developers.google.com/search/docs/advanced/robots/robots_meta_tag#directives) 和其他搜索引擎通常支持 `X-Robots-Tag` 头，用来指导爬虫如何索引你的网站。

例如，为了防止你的 `\*.\*.workers.dev` URL 被索引，请在`你的_headers` 文件中添加以下内容：

```
https://:version.:subdomain.workers.dev/*
  X-Robots-Tag: noindex
```

### 示例 自定义浏览器缓存行为

如果你有一个包含指纹资产（文件名中带有哈希值的资产）文件夹，你可以在浏览器中配置更激进的缓存行为，以提升回头访客的性能：

```
/static/*
  Cache-Control: public, max-age=31556952, immutable
```

### 示例 内容安全政策

```
/app/*
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff  
  Referrer-Policy: no-referrer  
  Permissions-Policy: document-domain=()  
  Content-Security-Policy: script-src 'self'; frame-ancestors 'none';
```

## 重定向

项目的静态资源目录中，以一个名为 `_redirects` 的纯文本文件（无扩展名）声明重定向。该文件本身不会作为静态资产提供，而是由 Workers 解析，其规则将应用到静态资源响应上。

`_redirects` 文件中定义的重定向不会应用于你的 Worker 代码所服务的请求，即使请求 URL 与 `_redirects` 中定义的规则相符。

该文件每行只能定义一个重定向，且必须遵循此格式，否则将被忽略。

```
[source] [destination] [code?]
```
- source：匹配路径，可包含通配符、占位符
- destination：目的地网址，可包含查询字符串、通配符、占位符
- code：状态码，可省略，只能使用 200、301 和 302，默认为 302

以 `#` 开头的行将被视为评论。

完整示例：

```
/home301 / 301
/home302 / 302
/querystrings /?query=string 301
/twitch https://twitch.tv
/trailing /trailing/ 301
/notrailing/ /nottrailing 301
/page/ /page2/#fragment 301
/blog/* https://blog.my.domain
/:splat/products/:code/:name /products?code=:code&name=:name
```

- 重定向的顺序很重要。如果同一`source`路径有多个重定向，最顶层的重定向首先应用。
- 静态重定向应先于动态重定向。
- 不能对查询参数进行匹配
- 网址只能包含一个通配符
- 占位符匹配除分隔符外的所有字符

`_redirects` 文件限制为 2000 个静态重定向和 100 个动态重定向，总共重定向数为 2100 个。每个重定向声明都有 1000 个字符的限制。