## 含义

`vary`字段告诉客户端和缓存服务器，哪些字段可能影响返回的内容。

比如，服务器对同一个 URL 可以返回两种内容。一种是客户端请求 HTML 网页。

```http
GET /catalog HTTP/1.1
Host: example.com 
Accept: text/html
```

另一种是客户端请求 API 内容。

```http
GET /catalog HTTP/1.1 
Host: example.com 
Accept: application/json
```

服务器根据客户端请求的`Accept`字段，判断应该返回什么内容。

```http
HTTP/1.1 200 OK
Content-Type: text/html
Cache-Control: public, max-age=3600
Vary: Accept
```

服务器返回的`Vary`字段，告诉客户端和缓存服务器，`Accept`字段的值会影响返回的内容。


