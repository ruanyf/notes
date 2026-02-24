# 网页缓存
## Cache Control

```http
Cache-Control: public, max-age=0, must-revalidate
```

这个标头告诉浏览器，资源可以缓存，但浏览器每次使用前都应重新验证内容的新鲜度。这确保了静态页面的良好网站性能，同时保证陈旧内容永远不会被提供。

## Etag

该头与默认`cache-control`头相辅相成。其值是静态资产文件的哈希值，浏览器可以在后续请求中使用带有 `If-None-Match` 头部的哈希值来检查新度，无需在匹配时重新下载整个文件。