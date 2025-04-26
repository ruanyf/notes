# Wrangler

wrangler 是 Cloudflare 开发者平台的命令行开发工具，允许你在命令行调用 Cloudflare 的各种功能。

官方仓库：[GitHub](https://github.com/cloudflare/workers-sdk/tree/main/packages/wrangler)

官方文档：[Cloudflare](https://developers.cloudflare.com/workers/wrangler/)

## 安装 & 更新

安装前，必须保证已经安装了 node.js 和 npm。

使用下面的命令，安装在你的项目里面。

```bash
$ npm install wrangler --save-dev
```

安装后，使用下面的命令检查一下版本。

```bash
$ npx wrangler --version

// or
$ npx wrangler version

// or
$ npx wrangler -v
```

使用下面的命令，更新 wrangler。

```bash
$ npm install wrangler@latest
```