# Cloudflare D1

## 简介

- 具有 SQLite 的 SQL 语义
- 内置灾难恢复功能
- Worker 和 HTTP API 访问
- 专为跨多个较小（10 GB）数据库进行横向扩展而设计
- 允许构建包含数千个数据库的应用程序。

## 创建数据库

wrangler 命令行创建数据库

```bash
$ npx wrangler@latest d1 create [数据库名]

✅ Successfully created DB 'prod-d1-tutorial' in region WEUR
Created your new D1 database.

{
  "d1_databases": [
    {
      "binding": "prod_d1_tutorial",
      "database_name": "prod-d1-tutorial",
      "database_id": "<unique-ID-for-your-database>"
    }
  ]
}
```

注意，数据库名的字符数少于 32 个，并使用破折号（-）代替空格。

使用`wrangler d1 create`创建数据库后，可以自动添加绑定到 wrangler 配置文件，也可以手动添加进去。

```json
{
	"d1_databases": [
		{
			"binding": "prod_d1_tutorial", // available in your Worker on env.DB
			"database_name": "prod-d1-tutorial",
			"database_id": "<unique-ID-for-your-database>"
		}
	]
}
```

注意，绑定名不同于数据库名，必须是有效的 JavaScript 变量名。

绑定可以在 worker 中以 `env.<BINDING_NAME>` 访问。

命令行运行本地的 SQL 语句文件

```bash
$ npx wrangler d1 execute prod-d1-tutorial --local --file=./schema.sql
```

运行 SQL 语句

```bash
$ npx wrangler d1 execute prod-d1-tutorial --local --command="SELECT * FROM Customers"
```

## SQL 数据库操作

创建表格

```sql
DROP TABLE IF EXISTS Customers;

CREATE TABLE IF NOT EXISTS Customers (CustomerId INTEGER PRIMARY KEY, CompanyName TEXT, ContactName TEXT);
```

插入数据

```sql
INSERT INTO Customers (CustomerID, CompanyName, ContactName) VALUES (1, 'Alfreds Futterkiste', 'Maria Anders'), (4, 'Around the Horn', 'Thomas Hardy'), (11, 'Bs Beverages', 'Victoria Ashworth'), (13, 'Bs Beverages', 'Random Name');
```

Worker 脚本查询

```javascript
export default {
  async fetch(request, env) {
    const { pathname } = new URL(request.url);

    if (pathname === "/api/beverages") {
      // If you did not use `DB` as your binding name, change it here
      const { results } = await env.prod_d1_tutorial
        .prepare("SELECT * FROM Customers WHERE CompanyName = ?")
        .bind("Bs Beverages")
        .run();
      return Response.json(results);
    }

    return new Response(
      "Call /api/beverages to see everyone who works at Bs Beverages",
    );
  },
};
```

删除数据库

```bash
$ npx wrangler d1 delete prod-d1-tutorial
```