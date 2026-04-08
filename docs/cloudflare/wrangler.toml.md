# Wrangler.toml

## 不同的环境设置

Wrangler 允许为同一个 Worker 应用程序创建不同的环境。

```json
{
	"name": "my-worker",
	"env": {
		"<ENV_NAME>": {
		// environment-specific configuration goes here
		}
	}
}
```

创建环境时，Cloudflare 实际上会创建一个名为 `<top-level-name>-<environment-name>` 的新 Worker。例如，一个名为 `my-worker` Worker 项目，其环境为 `dev` ，将部署为一个名为 `my-worker-dev` Worker。

在 Wrangler 命令中，可以使用 `--env` 或 `-e` 标志来指定环境。例如，您可以通过运行 `npx wrangler dev -e=dev` 在 `dev` 环境中开发 Worker，并通过 `npx wrangler deploy -e=dev` 来部署它。

或者，您可以使用 [`CLOUDFLARE_ENV` 环境变量](https://developers.cloudflare.com/workers/wrangler/system-environment-variables/#supported-environment-variables)来选择活动环境。例如， `CLOUDFLARE_ENV=dev npx wrangler deploy` 将部署到 `dev` 环境。命令行参数 `--env` 优先级高于 `CLOUDFLARE_ENV` 环境变量。

大多数键都是可继承的，这意味着可以在环境中使用顶级配置。 [绑定](https://developers.cloudflare.com/workers/runtime-apis/bindings/) （例如 `vars` 或 `kv_namespaces` ）不可继承，需要显式定义。

下面是不同环境设置不同的环境变量。

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

要在特定环境中运行 Wrangler 命令，可以传入 `--env` 或 `-e` 标志。例如，运行  `npx wrangler dev --env staging` 就会在名为 `staging` 环境中开发 Worker，然后使用 `npx wrangler deploy --env staging` 进行部署。

通过设置不同的环境变量 `ENVIRONMENT` ，就可以在脚本中确定是什么环境。

```javascript
if (ENVIRONMENT === "staging") {

// staging-specific code

} else if (ENVIRONMENT === "production") {

// production-specific code

}
```

要将测试环境部署到 `*.workers.dev` 子域，请在所需的环境中添加 `workers_dev = true` 。


## 秘密变量

秘密变量放在与 Wrangler 配置文件位于同一目录下的 `.dev.vars` 文件或 `.env` 文件中。

```
SECRET_KEY="value"
API_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9"
```

这两个文件不能同时使用，如果定义了 `.dev.vars` 文件，则在本地开发期间， `.env` 文件中的值将不会包含在 `env` 对象中。

`.dev.vars` 和 `.env` 文件不应提交到 git。请将 `.dev.vars*` 和 `.env*` 添加到项目的 `.gitignore` 文件中。

要为每个 Cloudflare 环境设置不同的密钥，请创建名为 `.dev.vars.<environment-name>` 或 `.env.<environment-name>` 文件。

- 使用 `.dev.vars.<environment-name>` 文件时，所有密钥都必须针对每个环境单独定义。如果 `.dev.vars.<environment-name>` 存在，则只会加载该文件；不会加载 `.dev.vars` 文件。
- 相反，所有匹配的 `.env` 文件都会被加载，并且它们的值会被合并。对于每个变量，会使用最具体文件中的值，优先级如下：
    - `.env.<environment-name>.local` （最具体）
    - `.env.local`
    - `.env.<environment-name>`
    - `.env` （最不具体的）
## 私有 Worker

如果要让一个 Worker 成为私有，即无法从互联网调用，只能从另一个 Worker 进行调用，那么可以把 workers.dev 发布关闭。

workers_dev = false

## 绑定另一个 worker

```toml
services = [
	{ binding = "AUTH", service = "authentication-service" }
]
```
