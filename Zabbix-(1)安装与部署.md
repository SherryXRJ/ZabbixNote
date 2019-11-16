# Zabbix-(一)安装与部署

## 一.前言

本文记录在Centos7.6平台 通过yum安装部署Zabbix 4.4

### 准备

* Centos7.6 虚拟机一台(ip: 192.168.152.140)

* Mysql 8.0.12数据库(ip: 192.168.152.1)

  

## 二.安装

### 1.安装php

yum安装php

```shell
# yum install -y php
```

### 2.安装httpd

yum安装httpd

```shell
# yum install -y httpd
```

### 3. 安装zabbix各组件

1. 添加rpm包

   ``` bash
   # rpm -Uvh https://repo.zabbix.com/zabbix/4.4/rhel/7/x86_64/zabbix-release-4.4-1.el7.noarch.rpm
   ```

2. 安装zabbix-server-mysql

   ```shell
   # yum install -y zabbix-server-mysql
   ```


3. 安装zabbix-web-mysql

   ```shell
   # yum install -y zabbix-web-mysql
   ```

4. 安装zabbix-agent

   ```shell
   # yum install -y zabbix-agent
   ```

   

## 三.初始化zabbix数据库

1. mysql创建zabbix用户，密码为zabbix

   ```mysql
   CREATE USER 'zabbix'@'%' IDENTIFIED BY 'zabbix';
   ```

2. 创建zabbix数据库，并为zabbix用户赋予权限

   ```mysql
   CREATE DATABASE zabbix DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_bin;
   
   GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'%';
   ```

3. 初始化schema

   注: zabbix sql在下面的这个路径

   ```txt
   /usr/share/doc/zabbix-server-mysql-4.4.0/create.sql.gz
   ```

   a. 如果zabbix主机安装了mysql-client那么可以向mysql写入初始化sql

   ```shell
   # zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -h192.168.152.1 -uzabbix -p zabbix
   ```

   b. 如果zabbix没有安装mysql-client，那么则可以将**create.sql.gz**文件导出，再执行sql，这里就不在赘述

   

## 四.配置zabbix

1. 配置zabbix-server

   ```shell
   # vim /etc/zabbix/zabbix_server.conf
   ```

   可以修改server相关配置，例如：端口，日志，SSL，数据库，告警脚本路径等

   

   这里修改数据库配置和允许的ip

   ```properties
   DBHost=192.168.152.1
   DBName=zabbix
   DBUser=zabbix
   DBPassword=zabbix
   DBPort=3306
   
   StatsAllowedIP=0.0.0.0/0
   ```

2. 配置zabbix前端

   ```shell
   # vim /etc/httpd/conf.d/zabbix.conf
   ```

   ```properties
   # 修改时区
   php_value date.timezone Asia/Shanghai
   ```

3. SELinux 配置

   ```shell
   # setsebool -P httpd_can_network_connect on
   # setsebool -P zabbix_can_network on
   # service httpd restart
   ```

4. zabbix-agent配置

   ```shell
   # vim /etc/zabbix/zabbix_agentd.conf
   ```

   ```properties
   # zabbix server地址
   Server=192.168.152.140
   
   ServerActive=192.168.152.140
   
   Hostname=Zabbix-server
   ```

   

## 五.启动zabbix

1. 启动zabbix-server和httpd

   ```shell
   # systemctl restart zabbix-server httpd
   ```

2. 启动zabbix-agent

   ```shell
   # systemctl start zabbix-agent
   ```



## 六.访问zabbix界面

访问 <http://192.168.152.140/zabbix/>

![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191029180241675-1208336289.png)




## 七.其他

### 部署问题	

1. mysql zabbix 账号问题，启动zabbix-server时，出现了

   ```text
     9213:20191029:144309.734 [Z3001] connection to database 'zabbix' failed: [2059] Authentication plugin 'caching_sha2_password' cannot be loaded: /usr/lib64/mysql/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory
   ```

   **解决: 修改zabbix账号**

   ```mysql
   ALTER USER 'zabbix'@'%' IDENTIFIED WITH mysql_native_password BY 'zabbix';
   ```

2. 未关闭selinux，出现

   ```text
   10947:20191029:145011.030 cannot start preprocessing service: Cannot bind socket to "/var/run/zabbix/zabbix_server_preprocessing.sock": [13] Permission denied.
   ```

   **解决:临时关闭selinux**

   ```shell
   # setenforce 0
   ```



### 参考文档

[官方文档](<https://www.zabbix.com/documentation/4.4/manual/installation/install_from_packages/rhel_centos>)