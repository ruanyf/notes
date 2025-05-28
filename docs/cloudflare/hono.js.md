# hono.js 教程

## 安装

```bash
$ npm i hono
$ npm i @hono/node-server
```

## 应用开发

下面是一个最基本的服务器应用。

```javascript
import { serve } from '@hono/node-server';
import { Hono } from 'hono';  
  
const app = new Hono();

app.get('/', (c) => {  
  return c.text('Hello!');  
});

serve(app, (info) => {  
  console.log(`Listening on http://localhost:${info.port}`);  
});
```

Hono 应用的默认端口是 3000。

## 获取 HTTP 请求

路由是`/contacts/:id`。

```javascript
const { id } = c.req.param();  
  
// 或  
  
const id = c.req.param( 'id' );
```

如果请求体是`application/json`类型，使用`c.req.json()`获取。

```javascript
app.post('/entry', async (c) => {
  const body = await c.req.json()
  // ...
})

const { firstName, lastName } = c.req.json();  
  
// 或  
  
const firstName = c.req.json( 'firstName' );  
const lastName = c.req.json( 'lastName' );
```

如果请求体类型为`text/plain`，则需要使用`c.req.text()`；如果请求体类型为`blob`，则需要使用`c.req.blob()`。

从 URL 查询字符串中读取参数，使用`c.req.query()`方法。

```javascript
const { limit, offset } = c.req.query();  
  
// 或  
  
const limit = c.req.query( 'limit' );  
const offset = c.req.query( 'offset' );
```