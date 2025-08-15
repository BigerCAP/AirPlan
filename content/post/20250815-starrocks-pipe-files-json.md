---
title: 通过 pipes 配合 insert into select files 增量导入
date: 2025-08-15T00:00:00+08:00
lastmod: 2025-08-15T00:00:00+08:00
categories:
  - software
tags:
  - S3
  - starrocks
  - files
---

## 测试目的 

1. json 文件数据导入到 StarRocks 数据指定表并按列存储字段    
2. json 文件所在 s3 目录新增文件后，数据库自动导入新增文件
3. json 文件数据避免重复

## 测试过程

1. 部署 1 套 v3.3.14 版本 StarRocks 集群（单节点 FE、BE）
    1. pipes 功能请看[官网文档](https://docs.starrocks.io/zh/docs/sql-reference/sql-statements/loading_unloading/pipe/CREATE_PIPE/)
        1. 异步调度工具
        2. 支持分辨数据目录内文件的变更状态
    2. files 表函数功能[官网文档](https://docs.starrocks.io/zh/docs/sql-reference/sql-functions/table-functions/files/)
        1. 支持 S3 对象存储、NFS、Hdfs、本地文件 
        2. 支持 导入和导出
        3. 支持 orc、parquet、csv、avro 格式
2. 部署 1 套 Minio 服务（本地 binary 启动）

### 测试数据样本

1. csv 文件数据样本，保存到部署 StarRocks 的节点 1 份

```json
{"category":"C++","author":"avc","title":"C++ primer","price":89.5}
{"category":"Java","author":"avc","title":"Effective Java","price":95}
{"category":"Linux","author":"avc","title":"Linux kernel","price":195}
```

2. 需要处理 S3 目录如下场景
  - 单个目录下可能存在多种文件格式
  - 单个目录或多个目录存在多个文件（每隔数秒新增 1 个文件）
  - 目录内已有文件可能存在更新（文件大小的变化）



### StarRocks 验证

1. 新建数据表，采用 duplicate key 表是为了验证多次创建任务对数据的影响

```sql
create database test;

CREATE TABLE test.`ooojson` (
    `category` varchar(100) NOT NULL COMMENT "",
    `author` varchar(1048) NULL COMMENT "",
    `dt` datetime NULL COMMENT "" -- 写文档是后加的内容，实际验证时未添加
  ) ENGINE=OLAP 
  DUPLICATE KEY(category);
```


2. 使用 select files 基于本地文件测试读取
    - 更多使用姿势参考 https://docs.starrocks.io/zh/docs/sql-reference/sql-functions/table-functions/files/
    - json 数据按照 csv 文件放在 /opt/ 目录下

```sql
SELECT  get_json_string($1,"$.category") as category,get_json_string($1,"$.author") as author  FROM FILES
(
    "path" = "file:///opt/ooojson.csv",
    "format" = "csv",
    "csv.column_separator"="\\x014f"  -- 声明 1 个不存在的分隔符，避免 json 中 \\t 的关键词，导致 json 数据被分割
    --"csv.enclose"='"', 不要声明这个，这个会强制格式掉 json 里的双引号，导致 json 格式不合法
    --"csv.skip_header"="1", -- 如果第一行不是 header，这个参数可忽略
    -- "csv.escape"="\\" -- 配合 pipe 使用时存在已知 bug，SQL 格式化丢失 1 个反斜线
);
```

```sql
mysql> 
mysql> SELECT  get_json_string($1,"$.category") as category,get_json_string($1,"$.author") as author  FROM FILES
    -> (
    ->     "path" = "file:///opt/ooojson.csv",
    ->     "format" = "csv",
    ->     "csv.column_separator"="\\x014f"
    ->     --"csv.enclose"='"',
    ->     --"csv.skip_header"="1"
    -> );
+----------+--------+
| category | author |
+----------+--------+
| C++      | avc    |
| Java     | avc    |
| Linux    | avc    |
+----------+--------+
3 rows in set (0.11 sec)
```

3. 测试将数据配合 insert into 写入到目标表（test.ooojson)
    1. 通过 show load 可以跟踪到该 ETL 任务执行成功

```sql
insert into test.ooojson(category,author)

SELECT  get_json_string($1,"$.category") as category,get_json_string($1,"$.author") as author  FROM FILES
(
    "path" = "file:///opt/ooojson.csv",
    "format" = "csv",
    "csv.column_separator"="\\x014f"  -- 声明 1 个不存在的分隔符，避免 json 中 \\t 的关键词，导致 json 数据被分割
    --"csv.enclose"='"', 不要声明这个，这个会强制格式掉 json 里的双引号，导致 json 格式不合法
    --"csv.skip_header"="1", -- 如果第一行不是 header，这个参数可忽略
    -- "csv.escape"="\\" -- 配合 pipe 使用时存在已知 bug，SQL 格式化丢失 1 个反斜线
);
```

4. 通过 pipe 循环写入（此时需要使用到 minio）
    - 语法参考：https://docs.starrocks.io/zh/docs/loading/s3/
5. 创建 pipe 能力，如下方式确定数据范围【如果变化就导入，如果无任何变化不会触发导入】
    - `"path" = "s3://ooopipe/ooojson.csv",` 这个只会监控 ooojson 单个文件，不会判断其他文件
    - `"path" = "s3://ooopipe/ooo*.csv",` 这种会监控 ooo 开头的文件
    - `"path" = "s3://ooopipe/*.csv",`  这会监控所有 csv 结尾的文件
    - 支持使用通配符匹配路径，比如： `"path" = "s3://abc*/dt=*/ooopipe/*.csv"`

```sql
CREATE or replace PIPE user_pipess
PROPERTIES
(
    "AUTO_INGEST" = "TRUE",
    "POLL_INTERVAL"="3" -- 间隔 3 秒捕获 1 次，默认是 300 秒 1 次；频率越快、compact 成本越大
)
AS
INSERT INTO ooojson
SELECT  get_json_string($1,"$.category") as category,get_json_string($1,"$.author") as author  FROM FILES
(
    "path" = "s3://ooopipe/ooo*.csv",
    "format" = "csv",
    "csv.column_separator"="\\x014f",
    "aws.s3.enable_ssl" = "false",
    "aws.s3.endpoint" = "http://127.0.0.1:9000",
    "aws.s3.access_key" = "minioadmin",
    "aws.s3.secret_key" = "minioadmin"
);
```

6. 查看 pipe 异步任务执行状态
    - 通过 show pipes 查看任务运行状态‘
    - 或者 `select * from information_schema.pipe_files` 查看文件导入历史记录

```sql

mysql> show pipes\G;
*************************** 1. row ***************************
DATABASE_NAME: test
      PIPE_ID: 278645
    PIPE_NAME: user_pipess
        STATE: RUNNING
   TABLE_NAME: test.ooojson
  LOAD_STATUS: {"loadedFiles":3,"loadedBytes":627,"loadingFiles":0,"lastLoadedTime":"2025-08-14 15:52:37"}
   LAST_ERROR: NULL
 CREATED_TIME: 2025-08-14 15:52:35
1 row in set (0.00 sec)

-- 通过 meta 数据库查看
mysql> select * from information_schema.pipe_files \G;
*************************** 10. row ***************************
         JobId: 278650
         Label: pipe-user_pipess-task-278647-0
         State: FINISHED
      Progress: ETL:100%; LOAD:100%
          Type: INSERT
      Priority: NORMAL
      ScanRows: 9
  FilteredRows: 0
UnselectedRows: 0
      SinkRows: 9
       EtlInfo: NULL
      TaskInfo: resource:N/A; timeout(s):3600; max_filter_ratio:0.0
      ErrorMsg: NULL
    CreateTime: 2025-08-14 15:52:37
  EtlStartTime: 2025-08-14 15:52:37
 EtlFinishTime: 2025-08-14 15:52:37
 LoadStartTime: 2025-08-14 15:52:37
LoadFinishTime: 2025-08-14 15:52:37
   TrackingSQL: 
    JobDetails: {"All backends":{"a4ec15ee-78e3-11f0-a387-260aca8fc881":[10001]},"FileNumber":3,"FileSize":627,"InternalTableLoadBytes":193,"InternalTableLoadRows":9,"ScanBytes":666,"ScanRows":9,"TaskNumber":1,"Unfinished backends":{"a4ec15ee-78e3-11f0-a387-260aca8fc881":[]}}
     Warehouse: 
10 rows in set (0.01 sec)
```

7. pipe 已知信息
    1. 以官网文档为准 [官网文档](https://docs.starrocks.io/zh/docs/sql-reference/sql-statements/loading_unloading/pipe/CREATE_PIPE/)
    2. 当 S3 一次新增多个文件时候
        1. 默认：文件个数不大于 BATCH_FILES = 256 个，文件大小不超过 BATCH_SIZE = 1GB 的时候，多个文件会占用 1 个 insert into select 导入能力加载到数据库
        2. 当新增文件体量超过如上任意条件时，会切分成多个 pipe 任务导入；参考：`fe/fe-core/src/main/java/com/starrocks/load/pipe/FilePipeSource.java`
            1. 假设新增 100 个文件，总加起来不超过 1G ，会使用 1 个 task 并发；path 可以设置一次导入多个文件
            2. 假设新增 50 个文件 && 超过 1G ，会使用多个 task 并发
            3. 假设新增 500 个文件 && 不超过 1G ，会使用多个 task 并发
            4. 并发个数被 FE 参数管理：`task_runs_concurrency=4` ；参考：`fe/fe-core/src/main/java/com/starrocks/scheduler/TaskRunScheduler.java`
    3. pipe 任务与数据导入
        1. 可查看：`fe/fe-core/src/main/java/com/starrocks/load/pipe/Pipe.java`
        2. 任务删除后，已经导入的记录会被删除
        3. 重新创建任务，会使用 files() 函数内的 path 信息重新匹配，数据可能存在重复导入
            1. 当 sink table 是 primary key 表时，该行为可以保证最终幂等性，但是数据加载占用的资源不会忽略
            2. 当 sink table 是 duplicate key 表时，该行为导致数据重复导入，数据会出现重复
        4. 数据表删除时，pipe 任务会联动删除
8. pipe 相关资料
    - [SHOW PIPES](https://docs.starrocks.io/zh/docs/sql-reference/sql-statements/loading_unloading/pipe/SHOW_PIPES/)
    - [CREATE PIPE](https://docs.starrocks.io/zh/docs/sql-reference/sql-statements/loading_unloading/pipe/CREATE_PIPE/)
    - [ALTER PIPE](https://docs.starrocks.io/zh/docs/sql-reference/sql-statements/loading_unloading/pipe/ALTER_PIPE/)
    - [DROP PIPE](https://docs.starrocks.io/zh/docs/sql-reference/sql-statements/loading_unloading/pipe/DROP_PIPE/)
    - [SUSPEND or RESUME PIPE](https://docs.starrocks.io/zh/docs/sql-reference/sql-statements/loading_unloading/pipe/SUSPEND_or_RESUME_PIPE/)
    - [RETRY FILE](https://docs.starrocks.io/zh/docs/sql-reference/sql-statements/loading_unloading/pipe/RETRY_FILE/)
