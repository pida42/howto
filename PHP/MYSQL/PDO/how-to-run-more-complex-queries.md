## Problem

If you execute SQL query that is more complexed that "one line" queries, like this:

```php

$sql = "
SET @[STRING:var1] = UNIX_TIMESTAMP();
SET @[STRING:var2] = NULL;
SET @[STRING:var3] = NULL;
SET @[STRING:var4] = NULL;
SET @[STRING:var5] = NULL;

SELECT
   (@[STRING:var2] := (UNIX_TIMESTAMP( [STRING:date] ) + TIME_TO_SEC( [STRING:date] ))) AS var2,
   (@[STRING:var3] := (ROUND(((@[STRING:var2] - UNIX_TIMESTAMP( [DATETIME] )) / 3600), 2))) AS var3,

   [...]

   (@[STRING:var5] := (FROM_UNIXTIME(IF(
       (@[STRING:var4] = 0),
       @[STRING:var2],
       (@[STRING:var2] - (@[STRING:var4] * 3600))
   )))) AS var5,

   (@[STRING:var6] := (CASE
      WHEN (UNIX_TIMESTAMP(@[STRING:var5) - @[STRING:var1] <= 0) THEN 'yes'
      ELSE 'no'
   END)) AS  var6,

   [STRING.tableAlias].[STRING.tableColumn1],
   [STRING.tableAlias].[STRING.tableColumn2],
   [STRING.tableAlias].[STRING.tableColumn3],
   [STRING.tableAlias].[STRING.tableColumn4],
   [STRING.tableAlias].[STRING.tableColumn5]

FROM [STRING.tableName] [STRING.tableAlias]
WHERE [STRING.tableAlias].[STRING.tableColumn6] = 0
ORDER BY @[STRING:var5 ASC
";
```

_(I used annotation `[DataType:Name]` for keywords in a standard MySQL query)_

... the query will returns bad results using any "static" driver for communication with MySQL (from PHP). I'm using _PDO_.

### Standard result by running from any MySQL client (for example connected throw socket)


| [column1]      | [column2]           | [column3]     | [column4]           | [column5]  | [column6]           | [column7]  | [column8]  | [column9] | [column10] |
|----------------|---------------------|---------------|---------------------|------------|---------------------|------------|------------|-----------|------------|
| 1472569200     | 8.81                | 0             | 2016-08-30 17:00:00 | yes        | 2016-08-30 08:11:42 | 2016-08-30 | 17:00:00   | 60        | 0          |
| 1474660800     | 362.31              | 24            | 2016-09-22 22:00:00 | no         | 2016-09-08 19:41:09 | 2016-09-23 | 22:00:00   | 60        | 0          |
| 1472306400     | 720.01              | 0             | 2016-08-27 16:00:00 | yes        | 2016-07-28 15:59:21 | 2016-08-27 | 16:00:00   | 60        | 0          |
| 1473523200     | 43.15               | 0             | 2016-09-10 18:00:00 | yes        | 2016-09-08 22:51:05 | 2016-09-10 | 18:00:00   | 60        | 0          |
| 1474628400     | 510.51              | 24            | 2016-09-22 13:00:00 | no         | 2016-09-02 06:29:23 | 2016-09-23 | 13:00:00   | 60        | 0          |
| 1475661600     | 765.64              | 24            | 2016-10-04 12:00:00 | no         | 2016-09-03 14:21:42 | 2016-10-05 | 12:00:00   | 60        | 0          |

_Point: I will talk fe. about `column5`._


### Basic running - fetching result using `PDO::Query` and `PDOStatement::fetchAll()`

In case of PHP looks call like this:

```
[...]
$result = $pdo->query($[php:var1])->fetchAll();
```

In this moment you got results:

| [column1]      | [column2]           | [column3]     | [column4]           | [column5]  | [column6]           | [column7]  | [column8]  | [column9] | [column10] |
|----------------|---------------------|---------------|---------------------|------------|---------------------|------------|------------|-----------|------------|
| 1472569200     | 8.81                | 0             | 2016-08-30 17:00:00 | no         | 2016-08-30 08:11:42 | 2016-08-30 | 17:00:00   | 60        | 0          |
| 1474660800     | 362.31              | 24            | 2016-09-22 22:00:00 | no         | 2016-09-08 19:41:09 | 2016-09-23 | 22:00:00   | 60        | 0          |
| 1472306400     | 720.01              | 0             | 2016-08-27 16:00:00 | no         | 2016-07-28 15:59:21 | 2016-08-27 | 16:00:00   | 60        | 0          |
| 1473523200     | 43.15               | 0             | 2016-09-10 18:00:00 | no         | 2016-09-08 22:51:05 | 2016-09-10 | 18:00:00   | 60        | 0          |
| 1474628400     | 510.51              | 24            | 2016-09-22 13:00:00 | no         | 2016-09-02 06:29:23 | 2016-09-23 | 13:00:00   | 60        | 0          |
| 1475661600     | 765.64              | 24            | 2016-10-04 12:00:00 | no         | 2016-09-03 14:21:42 | 2016-10-05 | 12:00:00   | 60        | 0          |


There is a fucking time hack for many hours... Why I got all values as BOOL (fe) same in all rows?

The point is in PDO driver and simple communication protokol (in this case).

### You need to run complete query as a Transaction

Look to example of running query:

```
$sql = "
 [...]
";

$pdo->beginTransaction();

$pdo->exec("SET @[STRING:var1] = UNIX_TIMESTAMP(NOW())");
$pdo->exec("SET @[STRING:var2] = NULL");
$pdo->exec("SET @[STRING:var3] = NULL");
$pdo->exec("SET @[STRING:var4] = NULL");
$pdo->exec("SET @[STRING:var5] = NULL");

$result = $pdo->query($[php:var1]);

$pdo->commit();

foreach($result->fetchAll() as $item) {
  [...]
}
```

### How it's possible?

Easy bussiness.. Transactions holds all requests in memory. Only after commit are _commands_ released and you got _dynamic_ result. 
