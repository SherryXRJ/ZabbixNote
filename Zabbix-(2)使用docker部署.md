# Zabbix-(二)使用docker部署

## 一.前言

前文记录了在服务器上搭建zabbix平台，本文记录使用docker部署zabbix 4.4

### 准备

* Centos7.6 虚拟机，并安装了docker 版本为18.09.6

  

## 二.docker部署

​        可以通过拉取官方镜像，修改启动参数，部署**zabbix-server-mysql、zabbix-web-nginx-mysql、zabbix-web-nginx-mysql、zabbix-agent**模块。本文记录拉取各个模块的镜像，并启动容器部署。     

​        除此之外，也可以拉取**zabbix-appliance**镜像，进行部署，该镜像内置了Mysql数据库、Zabbix server、基于Nginx和Java gateway的Zabbix web页面。可以参考官方部署示例:[通过Zabbix appliance部署](https://www.zabbix.com/documentation/4.4/manual/installation/containers#usage_examples)        



### 1.官方镜像

各模块镜像地址:

* [zabbix-server-mysql](<https://hub.docker.com/r/zabbix/zabbix-server-mysql>)
* [zabbix-agent](<https://hub.docker.com/r/zabbix/zabbix-agent>)
* [zabbix-web-nginx-mysql](<https://hub.docker.com/r/zabbix/zabbix-web-nginx-mysql>)
* [zabbix-java-gateway](<https://hub.docker.com/r/zabbix/zabbix-java-gateway>)
* [mysql](<https://hub.docker.com/_/mysql?tab=description>)

[zabbix官方所有镜像地址](<https://hub.docker.com/u/zabbix>)



### 2.容器部署

1. **mysql部署**

   *注:如果已有数据库，那么可以省去这个步骤*					

   ```shell
   docker run --name mysql-server -t \
         -e MYSQL_DATABASE="zabbix" \
         -e MYSQL_USER="zabbix" \
         -e MYSQL_PASSWORD="zabbix" \
         -e MYSQL_ROOT_PASSWORD="root" \
         -d mysql:8.0 \
         --character-set-server=utf8 --collation-server=utf8_bin
   ```

2. **zabbix java gateway部署**

   ```shell
   docker run --name zabbix-java-gateway -t \
         -d zabbix/zabbix-java-gateway:centos-4.4-latest
   ```

   

3. **zabbix-server-mysql部署**

   ​        这里link了上面部署的mysql容器和zabbix java gateway容器，并且指定数据库相关参数，和修改时区。更多的参数可以参考[dockerhub zabbix-server-msql启动参数](<https://hub.docker.com/r/zabbix/zabbix-server-mysql>) 以及 [zabbix-server 4.4官方参数 ](<https://www.zabbix.com/documentation/4.4/manual/appendix/config/zabbix_server>)

   **注:**如果启动日志无法连接到mysql，可以查看[部署问题](#questions)

   ```shell
   docker run --name zabbix-server-mysql -t \
         -e DB_SERVER_HOST="mysql-server" \
         -e MYSQL_DATABASE="zabbix" \
         -e MYSQL_USER="zabbix" \
         -e MYSQL_PASSWORD="zabbix" \
         -e MYSQL_PORT="3306" \
         -e ZBX_JAVAGATEWAY="zabbix-java-gateway" \
         -e ZBX_TIMEOUT="20" \
         -e PHP_TZ="Asia/Shanghai" \
         -v /etc/localtime:/etc/localtime \
         --link zabbix-java-gateway:zabbix-java-gateway \
         --link mysql-server:mysql \
         -p 10051:10051 \
         -d zabbix/zabbix-server-mysql:centos-4.4-latest
   ```

   

4. **zabbix-web-nginx-mysql部署**

   需要配置数据库相关参数，并暴露容器80端口供外部18080端口访问

   ```shell
   docker run --name zabbix-web-nginx-mysql -t \
         -e DB_SERVER_HOST="mysql-server" \
         -e MYSQL_DATABASE="zabbix" \
         -e MYSQL_USER="zabbix" \
         -e MYSQL_PASSWORD="zabbix" \
         -e MYSQL_PORT="3306" \
         -e ZBX_JAVAGATEWAY="zabbix-java-gateway" \
         -e ZBX_SERVER_HOST="zabbix-server-mysql" \
         -e PHP_TZ="Asia/Shanghai" \
         -v /etc/localtime:/etc/localtime \
         --link zabbix-server-mysql:zabbix-server-mysql \
         --link mysql-server:mysql \
         -p 18080:80 \
         -d zabbix/zabbix-web-nginx-mysql:centos-4.4-latest
   ```

   

5. **zabbix-agent部署**

   ```shell
   docker run --name zabbix-agent \
              -e ZBX_HOSTNAME="zabbix-server-mysql" \
              -e ZBX_SERVER_HOST="zabbix-server-mysql" \
              -v /etc/localtime:/etc/localtime \
              -p 10050:10050 \
              --link zabbix-server-mysql:zabbix-server-mysql \
              -d zabbix/zabbix-agent:centos-4.4-latest
   ```

   



## 三.访问zabbix界面

访问 <http://192.168.152.140:18080/>

![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191029175300360-1714629051.png)



## 四.其他

### <div id="questions">部署问题</div>

1. 无法连接到mysql，zabbix-server-mysql和zabbix-web-nginx-mysql容器打印日志

   ```text
   **** MySQL server is not available. Waiting 5 seconds...
   ```

   **原因:**由于mysql镜像使用的是mysql 8.0，而zabbix-server-mysql和zabbix-web-nginx-mysql镜像中安装的mysql-client是5版本的，所以这两个容器连接mysql时会报以下错误

   ```text
   ERROR 2059 (HY000): Authentication plugin 'caching_sha2_password' cannot be loaded: /usr/lib64/mysql/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory
   ```

   **解决:**修改zabbix账号

   ```shell
   # 进入mysql-server容器
   docker exec -it mysql-server /bin/bash -c 'mysql -uroot -proot'
   
   # 修改zabbix账号
   ALTER USER 'zabbix'@'%' IDENTIFIED WITH mysql_native_password BY 'zabbix';
   ```

   

