# worker 脚本

## 创建 worker 项目

```bash
$ npm create cloudflare@latest -- my-first-worker
```

进入项目目录。

```bash
$ cd my-first-worker
```

运行开发服务器。

```bash
$ npx wrangler dev
```

部署项目。

```bash
$ npx wrangler deploy
```

## worker 代码

worker 脚本的默认位置是 src/index.js ，它的代码如下。

```javascript
export default {

  async fetch(request, env, ctx) {

    return new Response("Hello World!");

  },

};
```

上面代码中， fetch() 在收到 HTTP 请求时调用。它接受三个参数 [`request`，`env` ， `context`](https://developers.cloudflare.com/workers/runtime-apis/handlers/fetch/)。