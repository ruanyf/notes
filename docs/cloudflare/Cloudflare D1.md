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

默认情况下，数据库会在最接近创建命令发出的区域创建。不过，如果你是在英国开发应用，但面向美国市场，这种情况下，你可能希望数据库位于美国。

创建数据库时，你可以将 –location=wnam 传递给命令，以更改领导者所在的位置。目前可选的地点有 wnam（西北美洲）、enam（北美东部）、weur（西欧）、eeur（东欧）和 apac（亚太地区）。

命令行运行本地的 SQL 语句文件

```bash
$ npx wrangler d1 execute prod-d1-tutorial --local --file=./schema.sql
```

运行 SQL 语句

```bash
$ npx wrangler d1 execute prod-d1-tutorial --local --command="SELECT * FROM Customers"
```

## 数据库结构修改

创建一个数据库结构修改（MIGRATION）。

```bash
$ npx wrangler d1 migrations create <DATABASE_NAME> <MIGRATION_NAME>
```

这会在项目根目录创建一个名为 migrations 的新文件夹。你需要用你创建数据库时用的数据库名称替换 <DATABASE_NAME>，并给数据库结构修改一个描述性名称。所有 migration 都会按时间顺序存储在 migrations 文件夹中，因此描述性名称有助于识别结构修改的目标。

上面这条命令会在 migrations 目录里创建一个结构修改脚本，里面可以写入 SQL 语句。

执行数据库结构修改。

```bash
## 本地修改
$​ npx wrangler d1 migrations apply photo-service --local

## 远程生产环境修改
$​ npx wrangler d1 migrations apply photo-service --remote
```

## Worker 执行 SQL 语句

```javascript
​**try**​ {​  results = ​**await**​ env.DB.prepare(​_`_
​​ ​ _SELECT i.*, c.display_name AS category_display_name_​​
 ​ _FROM images i_
 ​​ ​ _INNER JOIN image_categories c ON i.category_id = c.id_​
 ​ ​ _ORDER BY created_at DESC_​​ ​ _LIMIT ?1`_
 ​​  )
 ​  .bind(limit)
 ​  .all()​
   } ​**catch**​ (e) {
   ​  ​**let**​ message;
   ​  ​**if**​ (e ​**instanceof**​ Error) message = e.message;​
    ​  console.log({​  message: message​  });
    
    ​**return**​ ​**new**​ Response(​_'Error'_​, { status: 500 })
}
```

.prepare() 方法用来准备 SQL 语句。
.all() 方法表示执行 .prepare() 返回的语句，返回所有的查询结果。.all() 方法的返回格式如下。

```javascript
{
	results: array | null, // [] if empty, or null if it doesn't apply
	success: boolean, // true if the operation was successful, false otherwise
	meta: {
		duration: number, // duration of the operation in milliseconds
	}
}
```

.first() 返回第一个结果

```javascript
try {
  result = await env.DB.prepare(`
		SELECT i.*, c.display_name AS category_display_name
		FROM images i
		INNER JOIN image_categories c ON i.category_id = c.id
		WHERE i.id = ?1`
	)
	.bind(request.params.id)
	.first()
} catch (e) {
	let message;
	if (e instanceof Error) message = e.message;
	console.log({
		message: message
	});
	return new Response('Error', { status: 500 })
}
```

如果列名作为 .first() 的参数，将直接返回该列的值。

```javascript
const stmt = db.prepare('SELECT COUNT(1) as img_count FROM images');
const total = await stmt.first('img_count');
// Outputs 5
console.log(total);

const stmt = db.prepare('SELECT image_url, title FROM images');
const total = await stmt.first();
// outputs {image_url: 'http://www.example.com/foo.png', title: 'Some Title'}
console.log(total);

```

插入数据

```javascript
try {
result = await env.DB.prepare(`
INSERT INTO images
(category_id, user_id, image_url, title,
format, resolution, file_size_bytes)
VALUES
(?1, ?2, ?3, ?4, ?5, ?6, ?7)`
)
.bind(
json.category_id,
json.user_id,
json.image_url,
json.title,
json.format,
json.resolution,
json.file_size_bytes
)
.run()
} catch (e) {
let message;
if (e instanceof Error) message = e.message;
}
console.log({
message: message
});

```

.run() 方法执行 SQL 语句。

如果插入语句，需要返回插入的 ID。

```javascript
result = await env.DB.prepare(`
INSERT INTO images
(category_id, user_id, image_url, title, format, resolution, file_size_bytes)
VALUES
(?1, ?2, ?3, ?4, ?5, ?6, ?7)
RETURNING *;`
)
.bind(
json.category_id,
json.user_id,
json.image_url,
json.title,
json.format,
json.resolution,
json.file_size_bytes
)
.first()
```

返回执行结果。

```javascript
response that is returned like so:
return new Response(
JSON.stringify(result),
{
status: 201,
headers: { 'Content-type': 'application/json' }
}
);
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