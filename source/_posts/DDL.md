---
title: Hive 数据库与表操作全指南
date: 2025-07-12 00:00:00
tags:
  - Hive
  - 大数据
  - 数仓
categories:
  - 教程
---

> 🚀 本文是我在自建博客上的第一篇文章，旨在系统整理 Hive 中数据库与表的核心操作，适合初学者入门和开发者查阅。

------

# Hive 数据库与表操作全指南
## 📁 创建数据库

```
sql复制编辑-- 基础创建
CREATE DATABASE IF NOT EXISTS my_db
COMMENT '业务数据仓库';

-- 指定存储位置（默认路径：/user/hive/warehouse）
CREATE DATABASE my_db 
LOCATION '/path/on/hdfs';
```

> 💡 **提示**：使用 `IF NOT EXISTS` 可避免数据库已存在时抛出错误。

------

## 🔍 查询数据库

```
sql复制编辑-- 查看所有数据库
SHOW DATABASES;

-- 模糊匹配数据库名（如：以'test'开头）
SHOW DATABASES LIKE 'test*'; -- * 相当于 MySQL 中的 %

-- 查看数据库详情
DESCRIBE DATABASE EXTENDED my_db;
```

> `DESC` 是 `DESCRIBE` 的简写；`EXTENDED` 用于显示更丰富的元信息，如位置、参数等。

------

## ✏️ 修改数据库

```
sql复制编辑-- 修改数据库注释
ALTER DATABASE my_db SET DBPROPERTIES ('comment' = '新的描述信息');
```

> ⚠️ Hive 只允许修改数据库的 **描述信息和属性**，不支持重命名或移动数据库位置。

------

## ❌ 删除数据库

```
sql复制编辑-- 删除空数据库
DROP DATABASE IF EXISTS my_db;

-- 强制删除（连同所有表一起删除）
DROP DATABASE IF EXISTS my_db CASCADE;
```

> ⚠️ **谨慎使用 `CASCADE`，尤其是在生产环境中！**

------

## 🔄 切换当前数据库

```
sql复制编辑USE my_db;  -- 后续操作默认在该数据库中执行
SHOW TABLES; -- 显示 my_db 中的所有表
```

------

## 📊 表操作

### ✅ 创建表

#### 1. 创建普通表

```
sql复制编辑CREATE TABLE IF NOT EXISTS employees (
   id INT COMMENT '员工ID',
   name STRING COMMENT '姓名',
   salary FLOAT COMMENT '薪资',
   join_date DATE COMMENT '入职日期'
)
COMMENT '员工信息表'
ROW FORMAT DELIMITED 
   FIELDS TERMINATED BY ','    -- 列分隔符
   LINES TERMINATED BY '\n'    -- 行分隔符
STORED AS TEXTFILE             -- 存储格式（可选：ORC, PARQUET 等）
LOCATION '/user/hive/warehouse/my_db/employees';
```

#### 2. 创建分区表（高级）

```
sql复制编辑CREATE TABLE logs (
   log_time TIMESTAMP,
   message STRING
)
PARTITIONED BY (dt STRING, region STRING) -- 按日期和地区分区
STORED AS ORC;
```

#### 3. 复制已有表结构（不复制数据）

```
sql


复制编辑
CREATE TABLE new_table LIKE old_table;
```

------

### 🔎 查看表信息

```
sql复制编辑SHOW TABLES; -- 当前数据库下所有表
SHOW TABLES LIKE 'emp*'; -- 模糊匹配表名

DESCRIBE employees; -- 查看字段信息
DESCRIBE FORMATTED employees; -- 查看详细结构（含分区、存储格式等）
```

------

### 🛠️ 修改表结构

#### 重命名表

```
sql


复制编辑
ALTER TABLE employees RENAME TO staff;
```

#### 添加字段

```
sql复制编辑ALTER TABLE staff ADD COLUMNS (
   department STRING COMMENT '部门'
);
```

#### 修改字段名和类型

```
sql复制编辑ALTER TABLE staff 
CHANGE COLUMN salary salary_info DOUBLE COMMENT '税前薪资';
```

#### 添加分区

```
sql


复制编辑
ALTER TABLE logs ADD PARTITION (dt='2023-10-01', region='Asia');
```

#### 修改分区路径

```
sql复制编辑ALTER TABLE logs PARTITION (dt='2023-10-01', region='Asia')
SET LOCATION '/new/hdfs/path';
```

#### 删除分区

```
sql


复制编辑
ALTER TABLE logs DROP PARTITION (dt='2023-10-01', region='Asia');
```

------

### 🧹 删除/清空表

#### 删除表

```
sql


复制编辑
DROP TABLE IF EXISTS staff;
```

##### 外部表注意：

```
sql复制编辑-- 创建外部表（只删除元数据）
CREATE EXTERNAL TABLE ext_table (...);

-- 删除时不会删除 HDFS 上的数据文件
DROP TABLE ext_table;
```

#### 清空表数据

```
sql复制编辑TRUNCATE TABLE employees; -- 清空数据，保留表结构

-- 清空指定分区
TRUNCATE TABLE logs PARTITION (dt='2023-10-01');
```

------

### 📄 CTAS（通过查询创建表）

```
sql复制编辑CREATE TABLE [IF NOT EXISTS] table_name
AS SELECT ...;
```

用于将查询结果直接保存为一张新表。

------

## 📌 内部表 VS 外部表

| 特性             | 内部表（Managed Table）内部表（托管表） | 外部表（External Table）       |
| ---------------- | --------------------------------------- | ------------------------------ |
| **数据控制权**   | Hive 拥有并管理数据                     | Hive 只管理元数据              |
| **数据位置**     | 默认在仓库目录或自定义 LOCATION         | 必须使用 LOCATION 指定现有目录 |
| **删除操作**     | `DROP TABLE` 同时删除数据和元数据       | `DROP TABLE` 仅删除元数据      |
| **数据生命周期** | 随 Hive 表存在                          | 独立于 Hive，表删数据不删      |
| **典型场景**     | 临时表、中间结果表                      | 多方共享、外部系统写入数据     |



------

## 📘 说明：内部表与外部表详解

### 内部表（Managed Table）内部表（托管表） 

- **定义**：Hive 管理数据和元数据，创建表时数据会移动/写入 Hive 控制的路径。
- **删除时**：`DROP TABLE` 会删除 HDFS 上的实际数据。
- **适合场景**：
  - Hive 独立使用；
  - 中间或临时结果表；
  - 希望自动清理数据。

### 外部表（External Table）

- **定义**：Hive 仅管理元数据，数据路径由用户指定，不归 Hive 所有。
- **删除时**：`DROP TABLE` 只删除元数据，HDFS 上数据保留。
- **适合场景**：
  - Spark/Sqoop/Flume 等系统共同访问数据；
  - 数据需要长期存储或防止误删；
  - 多 Hive 应用共享同一数据源。