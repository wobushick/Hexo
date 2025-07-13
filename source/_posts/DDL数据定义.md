---
title: Hive 数据库与表操作全指南
date: 2025-07-13
tags:
  - Hive
  - 大数据
  - 数据仓库
  - HQL
categories:
  - 数据平台
  - Hive
description: 系统整理 Hive 中数据库和表的创建、查询、修改、删除、分区操作、内部表与外部表差异等常见操作指南。
---
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Hive 数据库与表操作全指南</title>
  <meta name="description" content="系统整理 Hive 中数据库和表的创建、查询、修改、删除、分区操作、内部表与外部表差异等常见操作指南。" />
  <style>
    body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif; background: #fff; color: #333; padding: 2rem; line-height: 1.6; }
    h1, h2, h3 { color: #1a73e8; }
    pre, code { background-color: #f5f5f5; padding: 0.5em; border-radius: 4px; display: block; overflow-x: auto; }
    table { width: 100%; border-collapse: collapse; margin: 1em 0; }
    th, td { border: 1px solid #ccc; padding: 0.75em; text-align: left; }
    th { background: #f0f0f0; }
    .tip, .warn { border-left: 5px solid; padding: 1em; margin: 1em 0; border-radius: 4px; }
    .tip { border-color: #4CAF50; background: #e8f5e9; }
    .warn { border-color: #f44336; background: #ffebee; }
    .highlight { background: #f5f5f5; border-left: 3px solid #ddd; padding: 0.5em 1em; }
  </style>
</head>
<body>

<h1>Hive 数据库与表操作全指南</h1>

<p>🚀 本文旨在系统整理 Hive 中关于数据库与表的常见操作，帮助你快速查阅并避免常见误区。</p>

<h2>数据库操作</h2>

<h3>创建数据库</h3>
<pre><code>-- 基础创建
CREATE DATABASE IF NOT EXISTS my_db 
COMMENT '业务数据仓库';

-- 指定存储位置
CREATE DATABASE my_db 
LOCATION '/path/on/hdfs';
</code></pre>

<div class="tip">
  💡 <strong>提示：</strong> 使用 <code>IF NOT EXISTS</code> 可避免数据库已存在时报错。
</div>

<h3>查询数据库</h3>
<pre><code>SHOW DATABASES;
SHOW DATABASES LIKE 'test*';  -- 支持模糊匹配
DESCRIBE DATABASE EXTENDED my_db;
</code></pre>

<h3>修改数据库</h3>
<pre><code>ALTER DATABASE my_db SET DBPROPERTIES ('comment' = '新描述');
</code></pre>

<div class="warn">
  ⚠️ <strong>注意：</strong> Hive 只能修改注释和属性，不能重命名或更改位置。
</div>

<h3>删除数据库</h3>
<pre><code>DROP DATABASE IF EXISTS my_db;
DROP DATABASE IF EXISTS my_db CASCADE;
</code></pre>

<div class="warn">
  ⚠️ <strong>警告：</strong> 生产环境请慎用 <code>CASCADE</code>，防止误删所有表数据。
</div>

<h3>切换数据库</h3>
<pre><code>USE my_db;
SHOW TABLES;
</code></pre>

<hr />

<h2>表操作</h2>

<h3>创建表</h3>

<h4>基础建表</h4>
<pre><code>CREATE TABLE IF NOT EXISTS employees (
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
</code></pre>

<h4>创建分区表</h4>
<pre><code>CREATE TABLE logs (
   log_time TIMESTAMP,
   message STRING
)
PARTITIONED BY (dt STRING, region STRING)
STORED AS ORC;
</code></pre>

<h4>复制表结构（不含数据）</h4>
<pre><code>CREATE TABLE new_table LIKE old_table;
</code></pre>

<h3>查看表</h3>
<pre><code>SHOW TABLES;
SHOW TABLES LIKE 'emp*';
DESCRIBE employees;
DESCRIBE FORMATTED employees;
</code></pre>

<h3>修改表</h3>

<h4>重命名表</h4>
<pre><code>ALTER TABLE employees RENAME TO staff;
</code></pre>

<h4>添加列</h4>
<pre><code>ALTER TABLE staff ADD COLUMNS (
  department STRING COMMENT '部门'
);
</code></pre>

<h4>修改列名/类型</h4>
<pre><code>ALTER TABLE staff 
CHANGE COLUMN salary salary_info DOUBLE COMMENT '税前薪资';
</code></pre>

<h4>添加/删除/修改分区</h4>
<pre><code>ALTER TABLE logs ADD PARTITION (dt='2023-10-01', region='Asia');
ALTER TABLE logs DROP PARTITION (dt='2023-10-01', region='Asia');
ALTER TABLE logs PARTITION (dt='2023-10-01', region='Asia') 
SET LOCATION '/new/hdfs/path';
</code></pre>

<h3>删除表</h3>
<pre><code>DROP TABLE IF EXISTS staff;

-- 外部表（只删除元数据）
CREATE EXTERNAL TABLE ext_table (...);
DROP TABLE ext_table;
</code></pre>

<h3>清空表数据</h3>
<pre><code>TRUNCATE TABLE employees;
TRUNCATE TABLE logs PARTITION (dt='2023-10-01');
</code></pre>

<h3>使用 CTAS 创建表</h3>
<pre><code>CREATE TABLE IF NOT EXISTS new_table
AS SELECT ...;
</code></pre>

<hr />

<h2>内部表 vs 外部表 对比</h2>

<table>
  <thead>
    <tr>
      <th>特性</th>
      <th>内部表 (Managed Table)</th>
      <th>外部表 (External Table)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>数据所有权</td>
      <td>Hive 拥有并完全控制数据</td>
      <td>Hive 只拥有元数据，数据归外部控制</td>
    </tr>
    <tr>
      <td>LOCATION 指定</td>
      <td>可选</td>
      <td>必须指定</td>
    </tr>
    <tr>
      <td>DROP TABLE 行为</td>
      <td>删除元数据 + 删除 HDFS 数据</td>
      <td>只删除元数据，数据不动</td>
    </tr>
    <tr>
      <td>生命周期</td>
      <td>表删 → 数据删</td>
      <td>表删 → 数据保留</td>
    </tr>
    <tr>
      <td>典型用途</td>
      <td>Hive 专用表、中间/临时表</td>
      <td>多工具共享数据源、避免误删</td>
    </tr>
  </tbody>
</table>

<hr />

<h2>📌 总结</h2>
<ul>
  <li><strong>内部表</strong>：适合 Hive 自主管理的数据表，便于自动清理。</li>
  <li><strong>外部表</strong>：适合共享、保留关键数据的分析场景。</li>
  <li><strong>慎用 DROP</strong>：内部表删除会连数据一起删，务必确认！</li>
</ul>

<p>如果这篇文章对你有帮助，欢迎点赞、收藏或留言交流 🙌</p>

</body>
</html>
