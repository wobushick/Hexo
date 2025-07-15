---
title: DML数据操作
date: 2013/7/15
categories: [数据库,DML,HQL]
tags: [数据库,Hive]
description: HQL、DML、数据平台相关操作
---
{% note info::数据操作即数据的增删改%}
### Load

*   将文件导入到hive表中,并不需要查询
    *   ```
        ​​LOAD DATA LOCAL INPATH '/local/file/system/path' [OVERWRITE] INTO TABLE tablename [PARTITION (part1=val1, part2=val2 ...)]​​
    *   `LOCAL`: 表示源文件在客户端本地文件系统。
    *   ​`OVERWRITE`**:​​ 覆盖目标表/分区中的数据。
    *   ​​`INTO`:​​ 追加数据。
    *   文件会被移动到 Hive 的数据仓库目录中。
    *   
            LOAD DATA INPATH 'hdfs://namenode/path/to/hdfs/file' [OVERWRITE] INTO TABLE ..​​
        *   源文件在 HDFS 上（无 `LOCAL` 关键字）。
        *   文件会被移动（而不是复制）到目标 Hive 位置。源文件将不存在于原来路径。
*   ​**​注意:​**​
    
    *   文件内容必须与表的定义（字段分隔符、集合分隔符等）相匹配。
    *   不会对数据进行任何转换（如 CSV 转 ORC），仅做移动或复制操作。
    *   加载数据后建议运行 `ANALYZE TABLE ... COMPUTE STATISTICS` 或 `msck repair table ...` (修复分区元数据) 以优化查询计划。

### SELECT

*   **核心**:检索表中的数据
*   ​**​关键特性：​**​
    
    *   ​**​过滤：​**​ 使用 `WHERE` 子句。
    *   ​**​聚合：​**​ 使用 `GROUP BY` 和聚合函数（如 `SUM`, `AVG`, `COUNT`, `MAX`, `MIN`）。
    *   ​**​排序：​**​ 使用 `ORDER BY`（全局排序）或 `SORT BY`（Reducer 内排序）。
    *   ​**​限制：​**​ 使用 `LIMIT`。
    *   ​**​连接：​**​ 支持 `INNER JOIN`, `LEFT OUTER JOIN`, `RIGHT OUTER JOIN`, `FULL OUTER JOIN`, `LEFT SEMI JOIN`, `CROSS JOIN`。
    *   ​**​视图：​**​ 查询结果可以持久化为视图。
    *   ​**​子查询：​**​ 支持在 `FROM`、`WHERE` 和 `HAVING` 子句中使用子查询（部分旧版本有限制）。
    *   ​**​分区裁剪：​**​ 查询指定分区可以显著提高性能（`WHERE part_column = 'value'`）。
    *   ​**​**`**DISTRIBUTE BY**` **/** `**CLUSTER BY**`**:​**​ 控制数据如何分布在 Reducer 之间进行后续处理（如 `SORT BY`）。

### INSERT​

*   ​**​核心：​**​ 将查询结果或数据写入 Hive 表。
*   ​**​主要形式:​**​
    
    *   ​`INSERT OVERWRITE TABLE target_table [PARTITION (part_col1=val1, part_col2=val2 ...)] SELECT ... FROM ...`​
        
        *   覆盖：_完全替换_目标表或指定分区中的数据。
    *   ​`INSERT INTO TABLE target_table [PARTITION (part_col1=val1, part_col2=val2 ...)] SELECT ... FROM ...`​
        
        *   追加：将查询结果_追加_到目标表或指定分区中已有数据之后。
    *   ​`INSERT OVERWRITE [LOCAL] DIRECTORY 'path' [ROW FORMAT ...] [STORED AS ...] SELECT ...`​
        
        *   将查询结果写入 HDFS 或本地文件系统的目录。
        *   `LOCAL` 表示写入本地文件系统。
        *   可以指定输出格式和分隔符。
*   ​**​动态分区插入：​**​
    
    *   在 `INSERT` 语句中，分区列的值由 `SELECT` 子句的结果动态确定。
    *   需要设置 `hive.exec.dynamic.partition.mode=nonstrict`（默认为 `strict`）。
    *   例如：`INSERT OVERWRITE TABLE sales PARTITION (country, state) SELECT ..., se_country, se_state FROM source_table;`
*   ​**​多表插入 (Multi Table Insert - MTI):​**​
    
    *   通过一个查询和多个 `INSERT` 子句，将结果同时写入多个目标表。
    *   语法：
        
        ```
        FROM source_table
        INSERT OVERWRITE TABLE target_table1 SELECT col1, col2 ...
        INSERT INTO TABLE target_table2 SELECT col3, col4 ...
        ```

### UPDATE

*   ​**​核心：​**​ 修改表中现有行。
*   ​**​关键点：​**​
    
    *   ​**​仅适用于事务表：​**​ 表必须配置为支持 ACID（原子性、一致性、隔离性、持久性）事务。
    *   ​**​前提条件：​**​
        
        *   表必须是分桶表 (`CLUSTERED BY`)。
        *   表文件格式必须是 ORC（推荐，性能最佳）。
        *   必须设置 `hive.support.concurrency=true`, `hive.enforce.bucketing=true`, `hive.exec.dynamic.partition.mode=nonstrict`, `hive.txn.manager=org.apache.hadoop.hive.ql.lockmgr.DbTxnManager` (或其等效替代品)。
    *   ​**​语法：​**​ `UPDATE tablename SET column = value [, column = value ...] [WHERE expression];`
    *   ​**​注意：​**​ 对于海量数据的更新操作，传统上更高效的方法是使用 `INSERT OVERWRITE` 重写整个分区或表。

### DELETE

*   ​**​核心：​**​ 删除表中的行。
*   ​**​关键点：​**​
    
    *   ​**​仅适用于事务表：​**​ 要求与 `UPDATE` 操作完全相同（分桶表、ORC 格式、ACID 配置）。
    *   ​**​语法：​**​ `DELETE FROM tablename [WHERE expression];`
    *   ​**​注意：​**​ 如果只想删除所有行，`TRUNCATE TABLE tablename;` 是一个更高效的选项（但它删除整个表数据，不能带 `WHERE`），但事务表上也要求类似配置。
    *   同样，大规模删除通常更倾向于重写分区。

### MERGE(Hivw2.2+引入)

*   ​**​核心：​**​ 在一条语句中根据条件执行 `INSERT`, `UPDATE`, `DELETE` 操作。也称为 "Upsert"。主要用于变更数据捕获处理。
*   ​**​关键点：​**​
    
    *   ​**​仅适用于事务表：​**​ 要求与 `UPDATE` 相同。
    *   ​**​语法：​**​ 相对复杂，需要指定源表、连接条件以及在匹配和不匹配情况下执行的操作。
    *   基本结构：
        
        ```
        MERGE INTO target_table AS T
        USING source_table AS S
        ON T.key = S.key
        WHEN MATCHED [AND some_condition] THEN UPDATE SET ... [DELETE WHERE ...] -- 可选 DELETE 子句
        WHEN MATCHED [AND some_condition] THEN DELETE -- 删除匹配行（可选）
        WHEN NOT MATCHED [AND some_condition] THEN INSERT VALUES (...) -- 插入新行
        WHEN NOT MATCHED BY SOURCE [AND some_condition] THEN ... -- (可选) 处理目标有但源没有的行
        ```

###   
📌 重要注意事项

1.  ​**​模式写/读 (Schema-on-Read)：​**​ Hive 在数据加载时不强制检查数据模式与表定义是否完全匹配。仅在查询时验证。错误的格式在加载过程中可能不会被发现，但在查询时会引发解析错误。
2.  ​**​不可变性 (历史限制)：​**​ 早期 Hive 设计（非事务表）中 HDFS 文件的不可变性限制了 `UPDATE` 和 `DELETE` 的效率。事务表（基于 ORC）通过引入`delta`文件等机制部分解决了这个问题。
3.  ​**​事务表的开销：​**​ 虽然事务表支持 ACID，但它们引入了额外的写开销（管理事务、压缩 `delta` 文件等）。仅当确实需要精细的行级操作时才启用事务功能。
4.  ​**​批量操作优先：​**​ Hive 最擅长的是批处理操作（大规模的 `SELECT` 和 `INSERT OVERWRITE`）。对于频繁的单行 `UPDATE/DELETE`，传统关系型数据库或 NoSQL 数据库可能更合适。
5.  ​**​分区优化：​**​ 在大型表上，`WHERE` 子句包含分区键可以极大地减少需要扫描的数据量（分区裁剪），显著提高查询性能。合理设计分区策略至关重要🔧。