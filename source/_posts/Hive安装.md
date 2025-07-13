---
title: Hive安装
date: 2025-07-13
tags:
  - Hive
  - HQL
categories:
  - 数据平台
  - Hive
description: Hive安装大致流程问题记录
---
## 一、准备工作


> ​**​注意**:​​ 以下操作默认您已配置好Hadoop环境

### 1\. 下载Hive

*   下载地址：[Hive 3.1.3](https://archive.apache.org/dist/hive/hive-3.1.3/apache-hive-3.1.3-bin.tar.gz)
*   上传至Linux的`/opt/software`目录

### 2\. 解压安装

```
tar -zxvf /opt/software/apache-hive-3.1.3-bin.tar.gz -C /opt/module
mv /opt/module/apache-hive-3.1.3-bin /opt/module/hive
```

## 二、环境配置

### 1\. 配置环境变量

编辑`/etc/profile`文件：`sudo vim /etc/profile`

在文件底部添加：

```
# HIVE_HOME
export HIVE_HOME=/opt/module/hive
export PATH=$PATH:$HIVE_HOME/bin
```

刷新配置文件：`source /etc/profile`

### 2\. 同步Guava版本

```
rm $HIVE_HOME/lib/guava-*.jar
cp /opt/hadoop-3.2.2/share/hadoop/common/lib/guava-*.jar $HIVE_HOME/lib/
```

## 三、MySQL数据库配置

### 1\. 安装MySQL

```
sudo apt install -y mysql-server-8.0
sudo systemctl start mysql
sudo systemctl enable mysql
```

### 2\. 连接远程MySQL

`mysql -uroot -h192.168.1.3 -p123456`

### 3\. 安装MySQL JDBC驱动

```
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-8.0.28.tar.gz
tar -xzvf mysql-connector-java-8.0.28.tar.gz
cp mysql-connector-java-8.0.28/mysql-connector-java-8.0.28.jar $HIVE_HOME/lib/
```

## 四、Hive配置

### 1\. 创建hive-site.xml

```
cd /opt/module/hive/conf
vim hive-site.xml
```

### 2\. 添加配置内容

```
<configuration>
    <!-- 指定Hive Metastore服务的URI（统一资源标识符） -->
    <property>
        <name>hive.metastore.uris</name>
        <value>thrift://192.168.1.6:9083</value>  <!-- Metastore服务地址和端口 -->
    </property>
    
    <!-- 指定是否使用本地Metastore服务（false表示使用远程服务） -->
    <property>
        <name>hive.metastore.local</name>
        <value>false</value>  <!-- 设置为false启用远程Metastore -->
    </property>
    
    <!-- 定义JDBC连接URL用于Metastore数据库 -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://192.168.1.3:3306/hive_metastore?createDatabaseIfNotExist=true&amp;useSSL=false&amp;characterEncoding=UTF-8</value>
        <!-- 
            jdbc:mysql://[数据库地址]:[端口]/[数据库名]
            createDatabaseIfNotExist=true: 自动创建数据库（首次启动时）
            useSSL=false: 禁用SSL连接（生产环境建议启用）
            characterEncoding=UTF-8: 设置字符编码
        -->
    </property>
    
    <!-- 指定JDBC驱动类名 -->
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.cj.jdbc.Driver</value>  <!-- MySQL JDBC驱动类 -->
    </property>
    
    <!-- 定义连接Metastore数据库的用户名 -->
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>  <!-- 数据库用户名 -->
    </property>
    
    <!-- 定义连接Metastore数据库的密码 -->
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>123456</value>  <!-- 数据库密码 -->
    </property>
    
    <!-- 指定Hive数据仓库的HDFS存储路径 -->
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>  <!-- 表数据默认存储位置 -->
    </property>
    
    <!-- 指定Hive临时文件的HDFS存储路径 -->
    <property>
        <name>hive.exec.scratchdir</name>
        <value>/user/hive/tmp</value>  <!-- 作业执行时的临时文件目录 -->
    </property>
</configuration>
```

## 五、初始化与启动

### 1\. 初始化元数据

```
cd /opt/module/hive/bin/
schematool -dbType mysql -initSchema --verbose
```

### 2\. 启动Metastore服务

`hive --service metastore > /tmp/metastore.log 2>&1 &`

### 3\. 验证端口监听

`netstat -tuln | grep 9083`

## 六、连接模式示意图  
![enter image description here](https://i.postimg.cc/pL7H7mC6/Pix-Pin-2025-07-13-21-51-39.png)