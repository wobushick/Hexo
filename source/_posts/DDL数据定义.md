---
title: DDL数据定义
date: 2025-07-13
tags: [数据库,Hive]
categories: [数据库,DDL,HQL]
description: HQL、DDL、数据平台相关操作
---

##  数据库操作

创建数据库

```
-- 基础创建
CREATE DATABASE IF NOT EXISTS my_db 
COMMENT '业务数据仓库';
-- 指定存储位置
CREATE DATABASE my_db 
LOCATION '/path/on/hdfs';
```

💡 ​**​提示：​**​ 使用 `IF NOT EXISTS` 可避免数据库已存在时报错。

### 查询数据库

```
SHOW DATABASES;
SHOW DATABASES LIKE 'test*';  -- 支持模糊匹配
DESCRIBE DATABASE EXTENDED my_db;
```

### 修改数据库

```
ALTER DATABASE my_db SET DBPROPERTIES ('comment' = '新描述');
```

⚠️ ​**​注意：​**​ Hive 只能修改注释和属性，不能重命名或更改位置。

### 删除数据库

```
DROP DATABASE IF EXISTS my_db;
DROP DATABASE IF EXISTS my_db CASCADE;
```

⚠️ ​**​警告：​**​ 生产环境请慎用 `CASCADE`，防止误删所有表数据。

### 切换数据库

```
USE my_db;
SHOW TABLES;
```

## 表操作

### 创建表

#### 基础建表

```
CREATE TABLE IF NOT EXISTS employees (
   id INT COMMENT '员工ID',
   name STRING COMMENT '姓名',
   salary FLOAT COMMENT '薪资',
   join_date DATE COMMENT '入职日期'
)
COMMENT '员工信息表'
ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
  LINES TERMINATED BY '\n'
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/my_db/employees';
```

#### 创建分区表

```
CREATE TABLE logs (
   log_time TIMESTAMP,
   message STRING
)
PARTITIONED BY (dt STRING, region STRING)
STORED AS ORC;
```

#### 复制表结构（不含数据）

```
CREATE TABLE new_table LIKE old_table;
```

### 查看表

```
SHOW TABLES;
SHOW TABLES LIKE 'emp*';
DESCRIBE employees;
DESCRIBE FORMATTED employees;
```

### 修改表

#### 重命名表

```
ALTER TABLE employees RENAME TO staff;
```

#### 添加列

```
ALTER TABLE staff ADD COLUMNS (
  department STRING COMMENT '部门'
);
```

#### 修改列名/类型

```
ALTER TABLE staff 
CHANGE COLUMN salary salary_info DOUBLE COMMENT '税前薪资';
```

#### 添加/删除/修改分区

```
ALTER TABLE logs ADD PARTITION (dt='2023-10-01', region='Asia');
ALTER TABLE logs DROP PARTITION (dt='2023-10-01', region='Asia');
ALTER TABLE logs PARTITION (dt='2023-10-01', region='Asia') 
SET LOCATION '/new/hdfs/path';
```

### 修改表

#### 重命名表

```
ALTER TABLE employees RENAME TO staff;
```

#### 添加列

```
ALTER TABLE staff ADD COLUMNS (
  department STRING COMMENT '部门'
);
```

#### 修改列名/类型

```
ALTER TABLE staff 
CHANGE COLUMN salary salary_info DOUBLE COMMENT '税前薪资';
```

#### 添加/删除/修改分区

```
ALTER TABLE logs ADD PARTITION (dt='2023-10-01', region='Asia');
ALTER TABLE logs DROP PARTITION (dt='2023-10-01', region='Asia');
ALTER TABLE logs PARTITION (dt='2023-10-01', region='Asia') 
SET LOCATION '/new/hdfs/path';
```

### 删除表

```
DROP TABLE IF EXISTS staff;

-- 外部表（只删除元数据）
CREATE EXTERNAL TABLE ext_table (...);
DROP TABLE ext_table;
```

### 清空表数据

```
TRUNCATE TABLE employees;
TRUNCATE TABLE logs PARTITION (dt='2023-10-01');
```

### 使用 CTAS 创建表

```
CREATE TABLE IF NOT EXISTS new_table
AS SELECT ...;
```

## 内部表 vs 外部表 对比

<figure class="table" style="width:96.01%;"><table class="ck-table-resized"><colgroup><col style="width:18.4%;"><col style="width:37.62%;"><col style="width:43.98%;"></colgroup><tbody><tr><td><span style="background-color:rgb(237,237,237);color:rgba(0,0,0,0.9);"><strong>特性</strong></span></td><td><span style="background-color:rgb(237,237,237);color:rgba(0,0,0,0.9);"><strong>内部表 (Managed Table)</strong></span></td><td style="background-color:var(--yb-md-th-bg-color);padding:10px 12px;"><strong>外部表 (External Table)</strong></td></tr><tr><td><strong>数据所有权/控制权</strong></td><td><strong>Hive拥有并完全控制数据。​</strong></td><td><strong>Hive只拥有元数据，数据由外部控制。​</strong></td></tr><tr><td><strong>创建表时数据位置​</strong></td><td><span style="background-color:rgb(255,255,255);color:rgba(0,0,0,0.9);">可指定(</span><code>LOCATION</code><span style="background-color:rgb(255,255,255);color:rgba(0,0,0,0.9);">)，默认在仓库目录下创建。</span></td><td style="background-color:var(--yb-md-td-bg-color-odd);padding:10px 12px;"><br>​<strong>​必须指定(</strong><code><strong>LOCATION</strong></code><strong>)，指向已有数据目录。​</strong></td></tr><tr><td><span style="background-color:rgb(255,255,255);color:rgba(0,0,0,0.9);">​</span><strong>​</strong><code><strong>DROP TABLE</strong></code><strong>操作</strong></td><td><span style="background-color:rgb(255,255,255);color:rgba(0,0,0,0.9);">​</span><strong>​删除元数据 AND 删除HDFS上的数据文件</strong></td><td><strong>只删除元数据，HDFS上的数据文件保持不动。​</strong></td></tr><tr><td><strong>数据生命周期​</strong></td><td><span style="background-color:rgb(255,255,255);color:rgba(0,0,0,0.9);">由Hive管理，表删数据失。</span></td><td style="background-color:var(--yb-md-td-bg-color-odd);padding:10px 12px;"><br>独立于Hive存在，表删数据留。</td></tr><tr><td><strong>典型使用场景​</strong></td><td><span style="background-color:rgb(255,255,255);color:rgba(0,0,0,0.9);">Hive专用中间/临时表；最终结果表(需删数据)。</span></td><td><span style="background-color:rgb(255,255,255);color:rgba(0,0,0,0.9);">共享数据源；外部分析；防止误删源数据；多应用共享。</span></td></tr></tbody></table></figure>