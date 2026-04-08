# SQLite 

## 数据类型

- INTEGER
- REAL
- TEXT
- NULL

附加类型

- PRIMARY KEY
- NOT NULL
- --JSON
## 外键

外键强制执行表之间的关系，防止不存在的用户创建记录，或者防止删除引用其他表中行的行。

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

可以为每个表定义多个外键关系，并且外键定义可以引用整个数据库架构中的多个表。

外键还可以定义更新或删除时的行为。

- `CASCADE` - 更新或删除父键会删除与其关联的所有子键（行）。
- `RESTRICT` ——当任何子键引用父键时，父键无法更新或删除。与默认的外键强制执行不同，应用了 `RESTRICT` 关系会立即返回错误，而不是在事务结束时才返回。
- `SET DEFAULT` - 将外键定义引用的子列设置为架构中定义的 `DEFAULT` 值。如果子列未设置 `DEFAULT` ，则无法使用此操作。
- `SET NULL` - 将外键定义引用的子列设置为 SQL `NULL` 。
- `NO ACTION` 采取任何行动。

```sql
CREATE TABLE users (
	user_id INTEGER PRIMARY KEY,
	email_address TEXT,
)

CREATE TABLE scores (
	score_id INTEGER PRIMARY KEY,
	game TEXT,
	score INTEGER,
	player_id INTEGER,
	FOREIGN KEY(player_id) REFERENCES users(user_id) ON DELETE CASCADE
)
```

上面示例中，从 `users` 表中删除用户将删除 `scores` 表中所有相关的行。

## 索引

创建索引使用 `CREATE INDEX` SQL 命令，并指定要创建索引的表和列。

```sql
CREATE TABLE IF NOT EXISTS orders (
    order_id INTEGER PRIMARY KEY,
    customer_id STRING NOT NULL, -- for example, a unique ID aba0e360-1e04-41b3-91a0-1f2263e1e0fb
    order_date STRING NOT NULL,
    status INTEGER NOT NULL,
    last_updated_date STRING NOT NULL
)

CREATE INDEX IF NOT EXISTS idx_orders_customer_id ON orders(customer_id)
```

索引的常用命名格式为 `idx_TABLE_NAME_COLUMN_NAMES` ，这样在管理数据库时，就可以识别索引所针对的表和列。

创建多列索引的例子。

```sql
CREATE INDEX idx_customer_id_transaction_date ON transactions(customer_id, transaction_date)
```

下表是多列索引的使用（不使用）例子

| 询问                                                                                          | 使用的索引是什么？                                |
| ------------------------------------------------------------------------------------------- | ---------------------------------------- |
| `SELECT * FROM transactions WHERE customer_id = '1234' AND transaction_date = '2023-03-25'` | 是的：指定索引中的两列。                             |
| `SELECT * FROM transactions WHERE transaction_date = '2023-03-28'`                          | 否：仅指定 `transaction_date` ，不包含索引中的其他最左侧列。 |
| `SELECT * FROM transactions WHERE customer_id = '56789'`                                    | 是的：指定 `customer_id` ，它是索引中最左边的列。         |

还可以创建只包含一列的部分数据的索引。

```sql
CREATE INDEX idx_order_status_not_complete ON orders(order_status) WHERE order_status != 6
```

使用 `DROP INDEX` 删除索引。删除的索引无法恢复。

要修改索引，[先删除它](https://developers.cloudflare.com/d1/best-practices/use-indexes/#remove-indexes) ，然后使用更新后的定义[创建新索引](https://developers.cloudflare.com/d1/best-practices/use-indexes/#create-an-index) 。向索引添加或删除列， [先删除](https://developers.cloudflare.com/d1/best-practices/use-indexes/#remove-indexes)该索引，然后使用新列[创建新索引](https://developers.cloudflare.com/d1/best-practices/use-indexes/#create-an-index) 。

创建索引后，运行 `PRAGMA optimize` 命令以提高数据库性能。`PRAGMA optimize` 会对数据库中的每个表运行 `ANALYZE` 命令，收集表和索引的统计信息。这些统计信息使查询规​​划器能够在执行用户查询时生成最高效的查询计划。

要验证查询是否使用了索引，请在查询语句前加上 [`EXPLAIN QUERY PLAN`↗](https://www.sqlite.org/eqp.html) 。

```sql
EXPLAIN QUERY PLAN SELECT * FROM users WHERE email_address = 'foo@example.com';
QUERY PLAN
`--SEARCH users USING INDEX idx_email_address (email_address=?)
```

通过查询 `sqlite_schema` 系统表，列出数据库中的索引及其 SQL 定义：

```sql
SELECT name, type, sql FROM sqlite_schema WHERE type IN ('index');
```

## JSON 查询

D1 可以查询和解析存储在数据库中的 JSON 数据，比如提取 JSON 对象中的值。

下面是一个 JSON 数据

```json
{
	"measurement": {
		"temp_f": "77.4",
		"aqi": [21, 42, 58],
		"o3": [18, 500],
		"wind_mph": "13",
		"location": "US-NY"
	}
}
```

下面的语句可以提取某个字段的值。

```sql
-- Extract the temperature value
SELECT json_extract(sensor_reading, '$.measurement.temp_f')-- returns "77.4" as TEXT

-- Extract the maximum PM2.5 air quality reading
sensor_reading -> '$.measurement.aqi[3]' -- returns 58 as a JSON number

-- Extract the o3 (ozone) array in full
sensor_reading -\-> '$.measurement.o3' -- returns '[18, 500]' as TEXT
```

获取数组长度。

```json
{
	"user_id": "abc12345",
	"previous_logins": ["2023-03-31T21:07:14-05:00", "2023-03-28T08:21:02-05:00", "2023-03-28T05:52:11-05:00"]
}
```

```sql
json_array_length(login_history, '$.previous_logins') --> returns 3 as an INTEGER
```

数组长度可以用在复杂的查询中 `json_array_length` 作为谓词 - 例如， `WHERE json_array_length(some_column, '$.path.to.value') >= 5` 。

JSON 插入值的例子。

```json
{"history": ["2023-05-13T15:13:02+00:00", "2023-05-14T07:11:22+00:00", "2023-05-15T15:03:51+00:00"]}
```

```sql
UPDATE users
SET login_history = json_insert(login_history, '$.history[#]', '2023-05-15T20:33:06+00:00')
WHERE user_id = 'aba0e360-1e04-41b3-91a0-1f2263e1e0fb'
```

`json_insert()` 要求三个参数：

1. 要修改的 JSON 数据的列的名称。
2. 要修改的对象中键的路径。使用 `[#]` 表示 `json_insert` 将值追加到数组末尾。
3. 要插入的 JSON 值。

要替换现有值，请使用 `json_replace()` ，它会覆盖已存在的键值对。要设置值（无论该值是否已存在），请使用 `json_set()` 。

使用 `json_each` 可以将数组展开为多行。

```sql
UPDATE users
SET last_audited = '2023-05-16T11:24:08+00:00'
WHERE id IN (SELECT value FROM json_each('[183183, 13913, 94944]'))
```

D1 提供的 JSON 相关函数列表

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
-- 返回错误
-- ERROR 9015: SQL engine error: query error: Error code 1: SQL error or missing database (malformed JSON)`
```
