# Zabbix-(七)分布式监控

## 一.前言

Zabbix提供了一套分布式监控的方案，即使用[Zabbix Proxy](<https://www.zabbix.com/documentation/4.4/manual/distributed_monitoring>)，本文记录使用Zabbix Proxy进行分布式监控。

官方所述Proxy的使用场景如下：

* 监控远程区域设备
* 监控本地网络不稳定区域
* 当 zabbix 监控上千设备时,使用它来减轻 server 的压力
* 简化分布式监控的维护

![Zabbix Proxy架构图](<https://www.zabbix.com/documentation/4.4/_media/manual/proxies/proxy.png>)

### 准备

* Zabbix Server 4.4 (ip 192.168.152.140)
* Centos 7， 用于安装 Zabbix Proxy (ip 192.168.152.144) 以下简称Proxy-Server
* mysql 8 (Zabbix Server 和 Zabbix Proxy 需要使用独立的数据库， ip 192.168.152.1) 
* 被Zabbix Proxy监控的主机 Centos 7 (ip 192.168.152.145) 以下简称Server-C



## 二.安装Zabbix Proxy

1. **在Proxy-Server安装Zabbix Proxy**

   ```shell
   # rpm -Uvh https://repo.zabbix.com/zabbix/4.4/rhel/7/x86_64/zabbix-release-4.4-1.el7.noarch.rpm
   
   # yum install zabbix-proxy-mysql
   ```

2. **配置Zabbix Proxy**

   ```shell
   # vim /etc/zabbix/zabbix_proxy.conf
   ```

   修改以下配置

   ```properties
   # Zabbix Server地址
   Server=192.168.152.140
   
   # Proxy的Hostname (默认Zabbix proxy)
   Hostname=Proxy-Server
   
   # 数据库配置
   DBName=zabbix_proxy
   DBUser=zabbix
   DBPassword=zabbix
   DBPort=3306
   
   ########### Proxy 特有参数 ############
   # Proxy已经将数据同步给Server后，数据保留时间(小时)
   ProxyLocalBuffer=0
   
   # Proxy与Server失去连接后，数据保留时间(小时)
   ProxyOfflineBuffer=1
   
   # 心跳包频率(秒)
   HeartbeatFrequency=60
   #####################################
   
   StatsAllowedIP=0.0.0.0/0
   ```

   更多配置项可以参考[官方配置](<https://www.zabbix.com/documentation/4.4/manual/appendix/config/zabbix_proxy>)

3. **配置Mysql**

   ***注: Zabbix Server和 Zabbix Proxy的数据库必须是分开独立的！！！***

   ```mysql
   # 新建zabbix_proxy数据库
   CREATE DATABASE zabbix_proxy DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_bin;
   
   # 给zabbix账号赋予权限
   GRANT ALL PRIVILEGES ON zabbix_proxy.* TO 'zabbix'@'%';
   ```

   初始化schema

   ```shell
   # zcat /usr/share/doc/zabbix-proxy-mysql*/schema.sql.gz | mysql -uzabbix -pzabbix -Dzabbix_proxy  -h192.168.152.1 -Dzabbix_proxy
   ```

4. **启动Zabbix Proxy**

   ```shell
   # systemctl start zabbix-proxy
   ```

</br>

## 三.Zabbix Server页面配置Proxy

点击【管理】-【agent代理程序】-【创建代理】

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191118152337202-1083142733.png)

| 配置项              | 值              |
| ------------------- | --------------- |
| * agent代理程序名称 | Proxy-Server    |
| 系统代理程序模式    | 主动式          |
| 代理地址            | 192.168.152.144 |

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191118152542594-1425130749.png)

</br >

Server与Proxy保持连接

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191118152805883-143917767.png)

</br >



## 四.利用Proxy监控主机

至此Zabbix Proxy已经启动完成，接下来就将利用Proxy-Server来监控Server-C。和使用Zabbix Server监控类似，被监控主机安装Zabbix agent，只步过agent需要proxy来监控。

1. **Server-C安装Zabbix agent**

   ```shell
   # rpm -Uvh https://repo.zabbix.com/zabbix/4.4/rhel/7/x86_64/zabbix-release-4.4-1.el7.noarch.rpm
   
   # yum install -y zabbix-agent
   ```

2. **配置Server-C的agent**

   ```shell
   # vim /etc/zabbix/zabbix_agentd.conf
   ```

   配置项

   ```properties
   # Server连接到Proxy的地址
   Server=192.168.152.144
   ServerActive=192.168.152.144
   
   # Server-C的hostname
   Hostname=Server-C
   
   ```

3. **启动Server-C的agent**

   ```shell
   # systemctl start zabbix-agent
   ```

4. **在Zabbix Server界面增加Server-C**

   增加【主机】

   | 配置项                       | 值              |
   | ---------------------------- | --------------- |
   | * 主机名称                   | Server-C        |
   | * 群组                       | Linux servers   |
   | agent代理程序的接口 (IP地址) | 192.168.152.145 |
   | agent代理程序的接口 (端口)   | 10050           |
   | 由agent代理程序监测          | Proxy-Server    |

   

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191118155230871-1507345214.png)

   </br>

   链接模板

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191118171308291-269353283.png)

   </br>

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191118171409396-1716019516.png)

</br>

至此，Server-C已经通过Zabbix Proxy进行监控，Proxy定时发送监控数据给Server，实现了分布式监控。新增监控项或者JMX监控可以参考我之前的文章。

