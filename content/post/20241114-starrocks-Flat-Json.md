---
title: StarRocks Flat Json 物化列验证
date: 2024-11-14T00:00:00+08:00
lastmod: 2024-11-14T00:00:00+08:00
categories: 
  - StarRocks
tags:
  - StarRocks
  - Json
---

## 0x00 概要

- StarRocks 相关文档
    - https://docs.starrocks.io/zh/docs/using_starrocks/Flat_json/#%E4%BD%BF%E7%94%A8-flat-json
    - 根据数据推算存储类型，使用 varchar、int、json 等不同类型存储
- clickhouse 相关文档
    - https://clickhouse.com/blog/a-new-powerful-json-data-type-for-clickhouse  
    - 使用的 Dynamic 和 variant 两种形式存储
- 对该功能有如下几个疑问
    - 存储空间相比之前存储 json 的变化【本文未验证】
    - 导入效率慢多少【本文未验证】
    - 查询效率提升多少【本文未验证】
    - 单个列 A 有 varchar 和 int 时，按什么推算？
        - insert into 写入的时候是生效？
        - NULL 值推算
        - 如何查看行 json 数据推算效果
    - 二级以上深度的情况下，写放大增加多少？【本文未验证】
        - 写放大产生的小文件数量因何而定
        - 最大支持多少深度？

## StarRocks 测试

总结：

- 插入数据量非常少，无法测试性能，从功能行为逻辑上可有效提升部分列查询时的效率、谓词过滤能力、避免无效数据参与计算与过滤
- 边界测试 `k2->'null' = "null"` 时，在 json 行为上获取一个不存在的 key 时，会自动返回一个 null；此时在数据库判定时： null is null 会成为 1=1 的效果

### 测试过程

- 3.3.4 存算一体版本

```sql
查看该功能是否为 true
select * from information_schema.be_configs where name = "enable_json_flat";

动态开启该功能
update information_schema.be_configs  set VALUE = "true" where name = "enable_json_flat";
```

- 基于官网文档建表

```sql
CREATE TABLE `t1` (
    `k1` int,
    `k2` JSON,
    `k3` VARCHAR(20),
    `k4` JSON
)             
DUPLICATE KEY(`k1`)
COMMENT "OLAP"
DISTRIBUTED BY HASH(`k1`) BUCKETS 1
PROPERTIES ("replication_num" = "1");
```

- 先写入一行数据，查看对应 `FROM t1[_META_];`
    - 数据默认只存储物化后的内容，原始数据不会存储；查询的时候重新拼接成 json 数据
    - meta 信息列会跟随数据量、数据内容的变化而不断的推导

```sql
INSERT INTO t1 (k1,k2) VALUES
(15,parse_json('{"str":"test_str0","Integer":11,"Double":3.14,"Object":{"a":"b"},"arr":[1,2,3],"Bool":true,"null":null}'));
```

```SQL
查看对应 FROM t1[_META_];

["nulls(TINYINT)","Bool(JSON)","Double(DOUBLE)","Integer(BIGINT)","Object.a(VARCHAR)","arr(JSON)","null(JSON)","str(VARCHAR)"] 
```


> tips：  
> 当数据 compact 后，默认将占有达到 90% 的相同数据做物化并做 meta 信息展示  
> 在没有 compaction 且数据量较少（ 测试环境是 10 条以内）时，`FROM t1[_META_];` 可能会返回多条信息

- 更换内容再写入一行数据，查看对应 `FROM t1[_META_];`
    - 只是更换了 key name，没有修改数据类型

```sql
INSERT INTO t1 (k1,k2) VALUES
(15,parse_json('{"abc":"test_str0","abd":11,"abf":3.14,"abh":{"a":"b"},"abr":[1,2,3],"abb":true,"abn":null}'));
```

```SQL
查看对应 FROM t1[_META_];（截取了 1 行，此时返回 2 行数据）

["nulls(TINYINT)","abb(JSON)","abc(VARCHAR)","abd(BIGINT)","abf(DOUBLE)","abh.a(VARCHAR)","abn(JSON)","abr(JSON)"] 
```

- 更换内容后，再写入一行数据，查看对应 `FROM t1[_META_];`
    - 更换 key name、修改 value type
    - meta 返回为空（此时 3 条数据，不满足 flat json 生成 meta 的阈值，没有建立物化 json 的 meta 信息）

```sql
INSERT INTO t1 (k1,k2) VALUES
(15,parse_json('{"abc":"test_str0","abd":"gooo","abf":"happy","abh":[1,2,3],"abr":true,"abb":123,"abn":null}'));

mysql> SELECT flat_json_meta(k2) FROM t1[_META_];
+--------------------+
| flat_json_meta(k2) |
+--------------------+
| []                 |
+--------------------+
1 row in set (0.03 sec)
```

- 插入一个不合法的数据（与上述问题相同，实则与数据本身无关），数据在写入过程中转为了 NULL 然后插入(没有开启严格模式 Strict Mode)
    - column 为 NULL 的时候，同样会被 flat json 所计算，会被列入 `"nulls(TINYINT)"` meta 信息内（当 key 列的数据为  None 或者 null 时也会列入该数据列
    - 错误原因：value 是字符串的情况下，没有双引号 `"abd":gooo,"abf":happy,` 导致 json 格式不合法，开启严格模式事物会提交失败

```sql
INSERT INTO t1 (k1,k2) VALUES
(150,parse_json('{"abc":"test_str0","abd":gooo,"abf":happy,"abh":[1,2,3],"abr":true,"abb":123,"abn":null}'));
Query OK, 1 row affected (0.12 sec)
{'label':'insert_e2b25886-a265-11ef-93ed-02427b1a17fc', 'status':'VISIBLE', 'txnId':'13'}

查看对应 FROM t1[_META_];
[] 
```

- 当数据无法被推算的时候，使用 json 行为作为兜底策略
    - clickhouse 使用  Variant 作为数据列推算
    - 当 select 返回数据的时候，explain verbose 里看到数据列为 json

```sql
mysql> SELECT flat_json_meta(k2) FROM t1[_META_];
+--------------------------------------------------------------------------------------------------------------+
| flat_json_meta(k2)                                                                                           |
+--------------------------------------------------------------------------------------------------------------+
| ["nulls(TINYINT)","abb(JSON)","abc(VARCHAR)","abd(JSON)","abf(JSON)","abn(JSON)","abr(JSON)","remain(JSON)"] |
+--------------------------------------------------------------------------------------------------------------+
1 row in set (0.05 sec)
```

> `select from t1[_META_]` 语法边界

- `FROM t1[_META_];` 本身是把指定列的 json meta 信息打印，正常为 1 条（相同 kv 格式化使用同一套）
    - 不支持 where 条件选择
    - 不支持 order by
    - 支持 limit 1 返回行数
- 刚导入了数据后，`FROM t1[_META_];`  可能会返回多行信息
    - 快速执行手动 compact 后，会留下一条信息
    - `ALTER TABLE t1 COMPACT;`
    - `ALTER TABLE t1 BASE COMPACT;`

```sql
-- 验证 where 是否参与过滤行为
mysql> SELECT flat_json_meta(k2) FROM t1[_META_] where k1 != 15 ;
+--------------------------------------------------------------------------------------------------------------------------------+
| flat_json_meta(k2)                                                                                                             |
+--------------------------------------------------------------------------------------------------------------------------------+
| ["nulls(TINYINT)","abb(JSON)","abc(VARCHAR)","abd(BIGINT)","abf(DOUBLE)","abh.a(VARCHAR)","abn(JSON)","abr(JSON)"]             |
| []                                                                                                                             |
| ["nulls(TINYINT)","Bool(JSON)","Double(DOUBLE)","Integer(BIGINT)","Object.a(VARCHAR)","arr(JSON)","null(JSON)","str(VARCHAR)"] |
+--------------------------------------------------------------------------------------------------------------------------------+
3 rows in set (0.05 sec)
```

- 默认返回 3 条数据，使用 limit 1 语法返回 1 条数据

```sql
mysql> SELECT flat_json_meta(k2) FROM t1[_META_] limit 1;
+--------------------------------------------------------------------------------------------------------------------------------+
| flat_json_meta(k2)                                                                                                             |
+--------------------------------------------------------------------------------------------------------------------------------+
| ["nulls(TINYINT)","Bool(JSON)","Double(DOUBLE)","Integer(BIGINT)","Object.a(VARCHAR)","arr(JSON)","null(JSON)","str(VARCHAR)"] |
+--------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.04 sec)
```

### flat json 相关参数

- 控制 flat json 按什么范围的数据生产 meta 信息

```text
enable_json_flat
默认值：false
类型：Boolean
是否动态：是
描述：是否开启 Flat JSON 特性。开启后新导入的 JSON 数据会自动打平，提升 JSON 数据查询性能。
引入版本：v3.3.0

json_flat_null_factor
默认值：0.3
类型：Double
是否动态：是
描述：控制 Flat JSON 时，提取列的 NULL 值占比阈值，高于该比例不对该列进行提取，默认为 0.3。该参数仅在 enable_json_flat 为 true 时生效。
引入版本：v3.3.0

json_flat_sparsity_factor
默认值：0.9
类型：Double
是否动态：是
描述：控制 Flat JSON 时，同名列的占比阈值，当同名列占比低于该值时不进行提取，默认为 0.9。该参数仅在 enable_json_flat 为 true 时生效。
引入版本：v3.3.0

json_flat_column_max
默认值：100
类型：Int
是否动态：是
描述：控制 Flat JSON 时，最多提取的子列数量。该参数仅在 enable_json_flat 为 true 时生效。
引入版本：v3.3.0

enable_compaction_flat_json
默认值：True
类型：Bool
是否动态：是
描述：控制是否为 Flat Json 数据进行 Compaction。
引入版本：v3.3.3

enable_lazy_dynamic_flat_json
默认值：True
类型：Bool
是否动态：是
描述：当查询在读过程中未命中 Flat JSON Schema 时，是否启用 Lazy Dynamic Flat JSON。当此项设置为 true 时，StarRocks 将把 Flat JSON 操作推迟到计算流程，而不是读取流程。
引入版本：v3.3.3

```

## StarRocks 查询验证

> flat json 在物化 key / values 时，数据是通过 cast 函数将 json kv 转为 int、varchar、decimal 等行为存储到 BE 节点，当单个 column 有多种数据格式会使用 json 行为存储（保底行为）  
> 需要注意数据写入、查询 后行为的一致性，针对 bool、tinyint、38 位以上的 浮点数、None / NULL 等数据边界一定要注意写入和查询时的一致性（建议自行验证是否符合业务逻辑）  

```sql
mysql> SELECT get_json_string(k2,'\$.abf') FROM t1 WHERE k2->'abb' = 123;
+------------------------------+
| get_json_string(k2, '$.abf') |
+------------------------------+
| happy                        |
+------------------------------+
1 row in set (0.04 sec)
```

### Profile 分析是否命中 flat json

> 从 profile 上可以看到 json kv 在字段谓词过滤时 hit / unhit 的行为

- AccessPathHits
    - Hit：命中 meta 后加速的数据
    - HitMerge：数据存在多个版本（比如导入数据后，没有 compaction）
    - hit 和 HitMerge 在查询 1 个 column 时二选一；目前从测试上未见同时出现常见
- AccessPathUnhits
    - Unhit：未命中 meta 列的查询数据

```yaml
UniqueMetrics:
- MorselQueueType: fixed_morsel_queue
- Predicates: json_query(2: k2, 'Double') = CAST(3.14 AS JSON)
- Rollup: t1
- SharedScan: False
- Table: t1
- AccessPathHits: 3
  - [Hit]Double: 1
  - [Hit]Object.a: 1
  - [Hit]remain: 1
- AccessPathUnhits: 2
  - [Unhit]Double: 1
  - [Unhit]Object: 1
- PushdownAccessPaths: 2
- PushdownPredicates: 1
```

> 当大量数据写入后未 compaction 时，会有 HitMerge 的信息

```yaml
UniqueMetrics:
- MorselQueueType: fixed_morsel_queue
- Predicates: json_query(2: k2, 'Double') = CAST(3.14 AS JSON)
- Rollup: t1
- SharedScan: False
- Table: t1
- AccessPathHits: 7
  - [HitMerge]abb: 1
  - [HitMerge]abc: 1
  - [HitMerge]abd: 1
  - [HitMerge]abf: 1
  - [HitMerge]abh.a: 1
  - [HitMerge]abn: 1
  - [HitMerge]abr: 1
- AccessPathUnhits: 0
```

### Json 中的 None 与 字符串 null 、SQL NULL

- 通过 explain verbose 检查列属性

```sql
explain verbose SELECT get_json_string(k2,'\$.abf') FROM t1 WHERE k2->'null' = "null";   

-- 截取重要信息列如下
5 <-> get_json_string[([2: k2, JSON, true], '$.abf'); args: JSON,VARCHAR; result: VARCHAR; args nullable: true; result nullable: true] 
Predicates: json_query[([2: k2, JSON, true], 'null'); args: JSON,VARCHAR; result: JSON; args nullable: true; result nullable: true] = cast('null' as JSON) 
```

- 测试如下三种场景
    - key 信息，value 为 SQL NULL
    - 不存在的 key 信息，json 库自动转为 null
    - 获取存在的 key 信息，value 为 null （等效一个 string 字符）

```sql
// 获取存在的 key 信息，value 为 null（ 等效 MySQL NULL）
mysql> select parse_json('{"a": null}')->"a", parse_json('{"a": null}')->"b" is null;
+--------------------------------+------------------------------------------+
| parse_json('{"a": null}')->'a' | (parse_json('{"a": null}')->'b') IS NULL |
+--------------------------------+------------------------------------------+
| null                           |                                        1 |
+--------------------------------+------------------------------------------+
1 row in set (0.01 sec)


// 获取不存在的 key 信息，json 库自动转为 null 
mysql> select parse_json('{"a": null}')->"b", parse_json('{"a": null}')->"b" is null;
+--------------------------------+------------------------------------------+
| parse_json('{"a": null}')->'b' | (parse_json('{"a": null}')->'b') IS NULL |
+--------------------------------+------------------------------------------+
| NULL                           |                                        1 |
+--------------------------------+------------------------------------------+
1 row in set (0.02 sec)


// 获取存在的 key 信息，value 为 null （等效一个 string 字符） 
mysql> select parse_json('{"a": "null"}')->"a", parse_json('{"a": "null"}')->"a" is null;
+----------------------------------+--------------------------------------------+
| parse_json('{"a": "null"}')->'a' | (parse_json('{"a": "null"}')->'a') IS NULL |
+----------------------------------+--------------------------------------------+
| "null"                           |                                          0 |
+----------------------------------+--------------------------------------------+
1 row in set (0.02 sec)

```

> Tips：使用 `explain verbose SELECT` 检查 SQL 语句在执行过程中使用的 column type；方便判断数据在 cast 转换后是否符合预期中的数学加工逻辑

```sql
mysql> explain verbose select  k2->'null' from t1;                                                                                           
+------------------------------------------------------------------------------------------------------------------------------------+
| Explain String                                                                                                                     |
+------------------------------------------------------------------------------------------------------------------------------------+
| RESOURCE GROUP: default_wg                                                                                                         |
|                                                                                                                                    |
| PLAN COST                                                                                                                          |
|   CPU: 57344.0                                                                                                                     |
|   Memory: 0.0                                                                                                                      |
|                                                                                                                                    |
| PLAN FRAGMENT 0(F00)                                                                                                               |
|   Fragment Cost: 28672.0                                                                                                           |
|   Output Exprs:5: expr                                                                                                             |
|   Input Partition: RANDOM                                                                                                          |
|   RESULT SINK                                                                                                                      |
|                                                                                                                                    |
|   1:Project                                                                                                                        |
|   |  output columns:                                                                                                               |
|   |  5 <-> json_query[([2: k2, JSON, true], 'null'); args: JSON,VARCHAR; result: JSON; args nullable: true; result nullable: true] |
|   |  cardinality: 28                                                                                                               |
|   |                                                                                                                                |
|   0:OlapScanNode                                                                                                                   |
|      table: t1, rollup: t1                                                                                                         |
|      preAggregation: on                                                                                                            |
|      partitionsRatio=1/1, tabletsRatio=1/1                                                                                         |
|      tabletList=10165                                                                                                              |
|      actualRows=28, avgRowSize=1025.0                                                                                              |
|      ColumnAccessPath: [/k2/null(json)]                                                                                            |
|      cardinality: 28                                                                                                               |
+------------------------------------------------------------------------------------------------------------------------------------+
25 rows in set (0.03 sec)
```
