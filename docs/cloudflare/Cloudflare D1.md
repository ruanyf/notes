# Cloudflare D1

## 简介

- 具有 SQLite 的 SQL 语义
- 内置灾难恢复功能
- Worker 和 HTTP API 访问
- 专为跨多个较小（10 GB）数据库进行横向扩展而设计
- 允许构建包含数千个数据库的应用程序。

D1 支持部分 SQLite 扩展以增加功能，包括：

- [FTS5 模块 ↗](https://www.sqlite.org/fts5.html) 用于全文搜索（包括 `fts5vocab` ）。
- [JSON 扩展 ↗](https://www.sqlite.org/json1.html) 用于 JSON 函数和运算符。
- [数学函数 ↗](https://sqlite.org/lang_mathfunc.html) 。

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
			"database_name": "prod-d1-tutorial", // 仅用于描述数据库，代码不直接引用
			"database_id": "<unique-ID-for-your-database>"
		}
	]
}
```

注意，绑定名不同于数据库名，必须是有效的 JavaScript 变量名。

绑定可以在 worker 中以 `env.<BINDING_NAME>` 访问。

默认情况下，数据库会在最接近创建命令发出的区域创建。不过，如果你是在英国开发应用，但面向美国市场，这种情况下，你可能希望数据库位于美国。

创建数据库时，你可以将 –location=wnam 传递给命令，以更改领导者所在的位置。目前可选的地点有 wnam（西北美洲）、enam（北美东部）、weur（西欧）、eeur（东欧）和 apac（亚太地区）。

## 创建数据表

保存 SQL 语句为文件 schema.sql

```sql
DROP TABLE IF EXISTS Customers;
CREATE TABLE IF NOT EXISTS Customers (
	CustomerId INTEGER PRIMARY KEY,
	CompanyName TEXT,
	ContactName TEXT
);
INSERT INTO Customers (CustomerID, CompanyName, ContactName) VALUES 
	(1, 'Alfreds Futterkiste', 'Maria Anders'),
	(4, 'Around the Horn', 'Thomas Hardy'),
	(11, 'Bs Beverages', 'Victoria Ashworth'),
	(13, 'Bs Beverages', 'Random Name')
;
```


然后，命令行在本地初始化数据库。

```bash
$ npx wrangler d1 execute prod-d1-tutorial --local --file=./schema.sql
```

远程初始化数据库

```sql
$ npx wrangler d1 execute prod-d1-tutorial --remote --file=./schema.sql
```

## 运行查询

本地运行查询

```bash
$ npx wrangler d1 execute prod-d1-tutorial --local --command="SELECT * FROM Customers"
```

远程运行查询

```bash
$ npx wrangler d1 execute prod-d1-tutorial --remote --command="SELECT * FROM Customers"
```

## 删除数据库

```bash
$ npx wrangler d1 delete prod-d1-tutorial
```

## 导出数据库

导出完整的 D1 数据库架构和数据。

```bash
$ npx wrangler d1 export <database_name> --remote --output=./database.sql
```

导出单个表结构和数据。

```bash
$ npx wrangler d1 export <database_name> --remote --table=<table_name> --output=./table.sql
```

仅导出 D1 数据库模式。

```bash
$ npx wrangler d1 export <database_name> --remote --output=./schema.sql --no-data
```

仅导出 D1 表结构。

```bash
$ npx wrangler d1 export <database_name> --remote --table=<table_name> --output=./schema.sql --no-data
```

仅导出 D1 数据库数据。

```bash
$ npx wrangler d1 export <database_name> --remote --output=./data.sql --no-schema
```

仅导出 D1 表数据。

```bash
$ npx wrangler d1 export <database_name> --remote --table=<table_name> --output=./data.sql --no-schema
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

Worker 连接数据库

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

- prepare() ：SQL 语句准备，语句中带有占位符
- bind()：数值绑定占位符
- run()：运行查询，返回所有行，如果查询不到任何行结果，返回 none

下面是另一个例子。

```javascript
export default {
    async fetch(request, env) {
        const {pathname} = new URL(request.url);
        const companyName1 = `Bs Beverages`;
        const companyName2 = `Around the Horn`;
        const stmt = env.DB.prepare(`SELECT * FROM Customers WHERE CompanyName = ?`);

        if (pathname === `/RUN`) {
            const returnValue = await stmt.bind(companyName1).run();
            return Response.json(returnValue);
        }

        return new Response(
            `Welcome to the D1 API Playground!
            \nChange the URL to test the various methods inside your index.js file.`,
        );
    },
};
```


```javascript
​**try**​ {
​  results = ​**await**​ env.DB.prepare(​_`_
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

## 使用 LIKE 搜索

可以使用 SQL 的 `LIKE` 运算符执行搜索。

```javascript
const { results } = await env.DB.prepare(
  "SELECT * FROM Customers WHERE CompanyName LIKE ?",
)
  .bind("%eve%")
  .run();
console.log("results: ", results);
```

## API

###  `prepare()`

D1PreparedStatement
- bind()：支持有序参数（ `?NNNN` ）和匿名参数（ `?` ）
- [`run()`](https://developers.cloudflare.com/d1/worker-api/prepared-statements/#run) ：对于 `UPDATE` 、 `DELETE` 或 `INSERT` 等写入操作， `results` 为空。
-  [`raw()`](https://developers.cloudflare.com/d1/worker-api/prepared-statements/#raw) ：运行预先准备好的查询（或多个查询），并将结果以数组的数组形式返回。返回的结果不包含元数据。默认情况下，列名不包含在结果集中。要将列名作为结果数组的第一行包含在内，请设置 `.raw({columnNames: true})` 。
-  [`first()`](https://developers.cloudflare.com/d1/worker-api/prepared-statements/#first)：运行预编译查询，并将查询结果的第一行作为对象返回。此操作不返回任何元数据，而是直接返回对象本身。列名可以作为参数名。如果没有查询到结果，返回 null。为了提高性能，请考虑在语句末尾添加 `LIMIT 1` 。

###  `batch()`

在一次数据库调用中发送多条 SQL 语句。这可以显著提升性能，因为它减少了与 D1 之间的网络往返延迟。列表中每条语句都会按顺序执行并提交，而非并发执行。

批量语句是 [SQL 事务 ↗](https://www.sqlite.org/lang_transaction.html) 。如果序列中的某个语句失败，则会针对该特定语句返回错误，并中止或回滚整个序列。

```javascript
const companyName1 = `Bs Beverages`;
const companyName2 = `Around the Horn`;

const stmt = env.DB.prepare(`SELECT * FROM Customers WHERE CompanyName = ?`);

const batchResult = await env.DB.batch([
	stmt.bind(companyName1),
	stmt.bind(companyName2)
]);
```

batch() 接受一个数组作为参数，数组成员的类型是 [`D1PreparedStatement`](https://developers.cloudflare.com/d1/worker-api/d1-database/#prepare) 。

返回值是一个 D1Result 对象数组。每个对象成员与参数数组的参数 D1Database::prepare 语句对应的位置。

```javascript
const companyName1 = `Bs Beverages`;
const companyName2 = `Around the Horn`;
const stmt = await env.DB.batch([
  env.DB.prepare(`SELECT * FROM Customers WHERE CompanyName = ?`).bind(companyName1),
  env.DB.prepare(`SELECT * FROM Customers WHERE CompanyName = ?`).bind(companyName2)
]);

console.log(stmt[1].results); 
/*
[{
	"CustomerId": 4,
	"CompanyName": "Around the Horn",
	"ContactName": "Thomas Hardy"
}]
*/

return Response.json(stmt)

```

### `exec()`

直接执行一个或多个查询，无需预处理语句或参数绑定。

```javascript
const returnValue = await env.DB.exec(`SELECT * FROM Customers WHERE CompanyName = "Bs Beverages"`);
```

它的参数是一个不带参数绑定的 SQL 查询语句。 输入可以是一个或多个查询，以 `\n` 分隔。

返回结果是 D1ExecResult 对象，有以下属性。

- `count` 属性包含已执行查询的数量。
- `duration` 属性包含操作的持续时间，单位为毫秒。

```javascript
const returnValue = await env.DB.exec(`SELECT * FROM Customers WHERE CompanyName = "Bs Beverages"`);
return Response.json(returnValue);
// {
//   "count": 1,
//   "duration": 1
// }
```

如果发生错误，则会抛出包含查询和错误消息的异常，执行停止，后续语句不会执行。

 这种方法的性能可能较差（在某些情况下，预处理语句可以重复使用），更重要的是，安全性较低。仅将此方法用于维护和一次性任务（例如，迁移作业）。

## 返回对象

|D1 Worker Binding API|返回对象|
|---|---|
|[`D1PreparedStatement::run`](https://developers.cloudflare.com/d1/worker-api/prepared-statements/#run) ， [`D1Database::batch`](https://developers.cloudflare.com/d1/worker-api/d1-database/#batch)|`D1Result`|
|[`D1Database::exec`](https://developers.cloudflare.com/d1/worker-api/d1-database/#exec)|`D1ExecResult`|

### D1Result

该对象包含：

- 成功状态
- 一个包含操作内部持续时间（以毫秒为单位）的元对象
- 结果（如有）以数组形式返回。

```javascript
{
  success: boolean, // true if the operation was successful, false otherwise
  meta: {
    served_by: string // the version of Cloudflare's backend Worker that returned the result
    served_by_region: string // the region of the database instance that executed the query
    served_by_primary: boolean // true if (and only if) the database instance that executed the query was the primary
    timings: {
      sql_duration_ms: number // the duration of the SQL query execution by the database instance (not including any network time)
    }
    duration: number, // the duration of the SQL query execution only, in milliseconds
    changes: number, // the number of changes made to the database
    last_row_id: number, // the last inserted row ID, only applies when the table is defined without the `WITHOUT ROWID` option
    changed_db: boolean, // true if something on the database was changed
    size_after: number, // the size of the database after the query is successfully applied
    rows_read: number, // the number of rows read (scanned) by this query
    rows_written: number // the number of rows written by this query
    total_attempts: number //the number of total attempts to successfully execute the query, including retries
  }
  results: array | null, // [] if empty, or null if it does not apply
}

// 实例
{
  "success": true,
  "meta": {
    "served_by": "miniflare.db",
    "served_by_region": "WEUR",
    "served_by_primary": true,
    "timings": {
      "sql_duration_ms": 0.2552
    },
    "duration": 0.2552,
    "changes": 0,
    "last_row_id": 0,
    "changed_db": false,
    "size_after": 16384,
    "rows_read": 4,
    "rows_written": 0
  },
  "results": [
    {
      "CustomerId": 11,
      "CompanyName": "Bs Beverages",
      "ContactName": "Victoria Ashworth"
    },
    {
      "CustomerId": 13,
      "CompanyName": "Bs Beverages",
      "ContactName": "Random Name"
    }
  ]
}
```

###  `D1ExecResult`

该对象包含：

- 已执行查询的数量
- 操作持续时间（毫秒）

```javascript
{
  "count": number, // the number of executed queries
  "duration": number // the duration of the operation, in milliseconds
}

// 实例
{
  "count": 1,
  "duration": 1
}
```


##  兼容的 PRAGMA 语句

`PRAGMA table_list` 列出数据库中的表和视图。这包括由 D1 维护的系统表。

```bash
$ npx wrangler d1 execute [DATABASE_NAME] --command='PRAGMA table_list'
```

`PRAGMA table_info("TABLE_NAME")` 显示给定 `TABLE_NAME` 架构（列、类型、空值、默认值）。

```bash
$ npx wrangler d1 execute [DATABASE_NAME] --command='PRAGMA table_info("Order")'
```

`PRAGMA table_xinfo("TABLE_NAME")` 与 `PRAGMA table_info(TABLE_NAME)` 类似，但还包括[生成的列](https://developers.cloudflare.com/d1/reference/generated-columns/) 。

```bash
$ npx wrangler d1 execute [DATABASE_NAME] --command='PRAGMA table_xinfo("Order")'
```

`PRAGMA index_list("TABLE_NAME")`显示给定 `TABLE_NAME` 索引。

```bash
$ npx wrangler d1 execute [DATABASE_NAME] --command='PRAGMA index_list("Territory")'
```

## 查询 `sqlite_master`
  
您还可以查询 `sqlite_master` 表，以显示所有表、索引以及用于生成它们的原始 SQL。

```bash
$ SELECT name, sql FROM sqlite_master
```

## 外键约束

外键约束允许强制执行跨表的关系。例如，您可以使用外键`user_id`绑定`users`表和`orders`表，从而防止为不存在的用户创建订单。外键约束还可以防止删除引用其他表中行的行。

`PRAGMA foreign_keys = on`默认情况下，D1 会强制所有查询和迁移中的外键约束都有效。这与在 SQLite 中为每个事务设置外键约束的行为完全相同。

在对 D1 数据库运行[查询](https://developers.cloudflare.com/d1/worker-api/)、[迁移](https://developers.cloudflare.com/d1/reference/migrations/)或[导入数据](https://developers.cloudflare.com/d1/best-practices/import-export-data/)时，有时可能需要在创建表或更改架构期间禁用外键验证。D1 的外键强制执行机制使用 SQLite 的指令`PRAGMA foreign_keys = on`。

D1 允许您调用`PRAGMA defer_foreign_keys = on`或`off`，这允许您暂时违反外键约束（直到当前事务结束）。如果在事务结束时您尚未解决未解决的外键违规问题，则会失败并返回`FOREIGN KEY constraint failed`错误。

要延迟执行外键，请`PRAGMA defer_foreign_keys = on`。

```sql
-- Defer foreign key enforcement in this transaction.
PRAGMA defer_foreign_keys = on

-- Run your CREATE TABLE or ALTER TABLE / COLUMN statements
ALTER TABLE users ...

-- This is implicit if not set by the end of the transaction.
PRAGMA defer_foreign_keys = off
```

外键可以通过`CREATE TABLE`语句创建表或`ALTER TABLE`向现有表添加列来添加。

```sql
CREATE TABLE users (
    user_id INTEGER PRIMARY KEY,
    email_address TEXT,
    name TEXT,
    metadata TEXT
)

CREATE TABLE orders (
    order_id INTEGER PRIMARY KEY,
    status INTEGER,
    item_desc TEXT,
    shipped_date INTEGER,
    user_who_ordered INTEGER,
    FOREIGN KEY(user_who_ordered) REFERENCES users(user_id)
)
```

外键定义`REFERENCES table(column)`。

外键定义还可以附加 `ON UPDATE` 和 `ON DELETE`子句，指定5种操作。

- `CASCADE`- 更新或删除父键会删除与其关联的所有子键（行）。
- `RESTRICT`_当任何子_键引用父键时，父键无法更新或删除。与默认的外键强制执行机制不同，`RESTRICT`应用此机制的关系会立即返回错误，而不是在事务结束时才返回。
- `SET DEFAULT`- 将外键定义引用的子列设置为`DEFAULT`架构中定义的值。如果`DEFAULT`子列未设置任何值，则无法使用此操作。
- `SET NULL`- 将外键定义引用的子列设置为 SQL `NULL`。
- `NO ACTION`- 不采取任何行动。

```sql
FOREIGN KEY(player_id) REFERENCES users(user_id) ON DELETE CASCADE
```

## JSON 查询

直接在 D1 中解析 JSON 的最大好处之一是它可以直接减少数据库的往返次数（查询次数）。

JSON 数据在 D1 中以 `TEXT` 列的形式存储。

- JSON 中的 null 值被视为 D1 `NULL` 。
- JSON 数字被视为 `INTEGER` 或 `REAL` 。
- 布尔值被视为 `INTEGER` 数值： `true` 为 `1` ， `false` 为 `0` 。
- 对象和数组值作为 `TEXT` 。

JSON 函数如下。

| 功能                                                          | 描述                                                                                                      | 例子                                                                                   |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `json(json)`                                                | 验证提供的字符串是否为 JSON，并返回该 JSON 对象的精简版本。                                                                     | `json('{"hello":["world" ,"there"] }')` 返回 `{"hello":["world","there"]}`             |
| `json_array(value1, value2, value3, ...)`                   | 根据这些值返回一个 JSON 数组。                                                                                      | `json_array(1, 2, 3)` 返回 `[1, 2, 3]`                                                 |
| `json_array_length(json)` - `json_array_length(json, path)` | 返回 JSON 数组的长度                                                                                           | `json_array_length('{"data":["x", "y", "z"]}', '$.data')` 返回 `3`                     |
| `json_extract(json, path)`                                  | 使用 `$.path.to.value` 语法提取给定路径中的值。                                                                       | `json_extract('{"temp":"78.3", "sunset":"20:44"}', '$.temp')` 返回 `"78.3"`            |
| `json -> path`                                              | 使用路径语法提取给定路径中的值，并将其作为 JSON 返回。                                                                          |                                                                                      |
| `json ->> path`                                             | 使用路径语法提取给定路径中的值，并将其作为 SQL 类型返回。                                                                         |                                                                                      |
| `json_insert(json, path, value)`                            | 在指定路径插入值。不会覆盖现有值。                                                                                       |                                                                                      |
| `json_object(label1, value1, ...)`                          | 接受键值对，并返回 JSON 对象。                                                                                      | `json_object('temp', 45, 'wind_speed_mph', 13)` 返回 `{"temp":45,"wind_speed_mph":13}` |
| `json_patch(target, patch)`                                 | 使用 JSON [MergePatch ↗](https://tools.ietf.org/html/rfc7396) 方法将提供的补丁合并到目标 JSON 对象中。                     |                                                                                      |
| `json_remove(json, path, ...)`                              | 删除指定路径下的键和值。                                                                                            | `json_remove('[60,70,80,90]', '$[0]')` 返回 `70,80,90]`                                |
| `json_replace(json, path, value)`                           | 在指定路径插入值。会覆盖现有值，但如果该值不存在，则不会创建新键。                                                                       |                                                                                      |
| `json_set(json, path, value)`                               | 在指定路径插入值。此操作会覆盖现有值。                                                                                     |                                                                                      |
| `json_type(json)` - `json_type(json, path)`                 | 返回所提供值或指定路径中的值的类型。返回 `null` 、 `true` 、 `false` 、 `integer` 、 `real` 、 `text` 、 `array` 或 `object` 中的一个。 | `json_type('{"temperatures":[73.6, 77.8, 80.2]}', '$.temperatures')` 返回 `array`      |
| `json_valid(json)`                                          | 无效 JSON 返回 0（false），有效 JSON 返回 1（true）。                                                                 | `json_valid({invalid:json})` 返回 `0\`                                                 |
| `json_quote(value)`                                         | 将提供的 SQL 值转换为 JSON 表示形式。                                                                                | `json_quote('[1, 2, 3]')` 返回 `[1,2,3]`                                               |
| `json_group_array(value)`                                   | 将提供的值以 JSON 数组的形式返回。                                                                                    |                                                                                      |
| `json_each(value)` - `json_each(value, path)`               | 将对象中的每个元素作为单独的行返回。它只会遍历顶层对象。                                                                            |                                                                                      |
| `json_tree(value)` - `json_tree(value, path)`               | 将对象中的每个元素作为单独的行返回。它会遍历整个对象。                                                                             |                                                                                      |

当处理非 JSON 格式或无效的 JSON 数据时，JSON 函数将返回 `malformed JSON` 错误。

```sql
SELECT json_extract('not valid JSON: just a string', '$')
// ERROR 9015: SQL engine error: query error: Error code 1: SQL error or missing database (malformed JSON)`
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