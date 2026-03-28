# Wrangler.toml

## 私有 Worker

如果要让一个 Worker 成为私有，即无法从互联网调用，只能从另一个 Worker 进行调用，那么可以把 workers.dev 发布关闭。

workers_dev = false

## 绑定另一个 worker

```toml
services = [
	{ binding = "AUTH", service = "authentication-service" }
]
```
