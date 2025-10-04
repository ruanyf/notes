# worker 绑定 R2

## 设置文件

wrangler.jsonc

```json
{

  "r2_buckets": [

    {

       "binding": "MY_BUCKET",

       "bucket_name": "<YOUR_BUCKET_NAME>"

     }

  ]

}
```

wrangler.toml

```toml
[[r2_buckets]]

binding = 'MY_BUCKET' # <~ valid JavaScript variable name

bucket_name = '<YOUR_BUCKET_NAME>'
```

## binding 方法

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const key = url.pathname.slice(1);

    switch (request.method) {
      case "PUT":
        await env.MY_BUCKET.put(key, request.body);
        return new Response(`Put ${key} successfully!`);

      default:
        return new Response(`${request.method} is not allowed.`, {
          status: 405,
          headers: {
            Allow: "PUT",
          },
        });
    }
  },
};
```