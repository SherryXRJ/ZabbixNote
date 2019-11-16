# Zabbix-(六) JMX监控

## 一.前言

Zabbix提供了JMX监控，它通过JMX API获取JVM信息，从而提供监控数据。本文讲述使用JMX监控Tomcat的JVM信息。

### 准备

* Zabbix Server 4.4 (ip: 192.168.152.140)
* 运行Java应用的主机 以下简称Server-A （已被Zabbix监控） (ip:192.168.152.142)

</br>

## 二.安装Zabbix-Java-gateway

Zabbix Server通过Zabbix Java gateway收集JMX监控数据，因此首先需要安装Zabbix-Java-gateway，同时修改Zabbix Server的配置。

1. **安装Zabbix-Java-gateway**

   可以在其他主机安装Zabbix-Java-gateway，只需要修改Zabbix-server配置文件，指定Zabbix-Java-gateway的地址和端口，这里就在部署Zabbix Server的主机上部署Zabbix-Java-gateway。

   ```shell
   # yum install zabbix-java-gateway
   ```

2. **配置Zabbix-Java-gateway**

   配置文件是```  /etc/zabbix/zabbix_java_gateway.conf```文件，文本采取默认配置，配置项详细信息可以参考下图或者参考[官方Zabbix-java-gateway配置项](<https://www.zabbix.com/documentation/4.4/manual/appendix/config/zabbix_java>)。

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191114142957193-1288065724.png)

3. **启动Zabbix-Java-gateway**

   ```shell
   # systemctl start zabbix-java-gateway
   ```

4. **修改Zabbix server配置**

   需要在zabbix server配置文件中增加zabbix-java-gateway相关配置

   ```shell
   # vim /etc/zabbix/zabbix_server.conf
   ```

   修改配置信息

   ```properties
   # zabbix-java-gateway地址
   JavaGateway=192.168.152.140
   
   # zabbix-java-gateway端口
   JavaGatewayPort=10052
   
   StartJavaPollers=5
   ```

   重启Zabbix server

   ```shell
   # systemctl restart zabbix-server
   ```

</br>

## 三.修改Java应用启动参数

本文是采用docker部署tomcat，因此本文中是修改tomcat的*catalina.sh*脚本，主要是添加以下几个jvm启动参数

```shell
-Dcom.sun.management.jmxremote
# java应用ip地址(docker部署 使用宿主机ip)
-Djava.rmi.server.hostname=192.168.152.142
# jmx端口
-Dcom.sun.management.jmxremote.port=12345
-Dcom.sun.management.jmxremote.rmi.port=12345
# 不开启认证
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
```

在tomcat catalina.sh脚本则可以在文件前面添加

```shell
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote"
JAVA_OPTS="$JAVA_OPTS -Djava.rmi.server.hostname=192.168.152.142"
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.port=12345"
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.rmi.port=12345"
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.authenticate=false"
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.ssl=false"
```

重新启动tomcat容器，暴露JMX 12345端口，并挂载数据卷（主要是配置tomcat的catalina.sh），这里可以先启动一个tomcat容器，将容器中的catalina.sh docker cp到宿主机上再做修改

```shell
# docker rm -f tomcat
# docker run --name tomcat -p 8080:8080 -p 12345:12345 -v /data/zabbix/catalina.sh:/usr/local/tomcat/bin/catalina.sh  -dit tomcat:jdk8-adoptopenjdk-hotspot
```

启动后可以通过jconsole工具测试一下能不能监控tomcat容器

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191114151616072-970530753.png)



![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191114151711889-629028225.png)

<br />

## 四.在Zabbix界面配置JMX

服务启动后，需要在zabbix界面为Server-A主机上增加JMX监控。

点击【配置】-【主机】-选择【Server-A】

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191114162938114-187826928.png)

增加JMX配置

| 配置项 | 值              |
| ------ | --------------- |
| IP地址 | 192.168.152.142 |
| 端口   | 12345           |



![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191114163028593-615627500.png)

<br/>



## 五.自定义JMX监控模板

实际上Zabbix 4.4自带了两个JMX模板***Template App Apache Tomcat JMX***和***Template App Generic Java JMX***模板一个可以监控Tomcat应用、一个可以监控普通Java应用。读者可以直接为被监控主机增加链接上述两个模板，也可以快速进行监控。

本文采用自定义的方式创建JMX监控模板

***注：Zabbix提供的 JMX模板不一定适配所有 JDK 版本，例如 Template App Generic Java JMX 模板中提供了Perm Gen 永久代的监控项，而 JDK 8中已经将 Perm Gen永久代替换为了 Metaspace元空间，而模板中没有元空间的监控项***



<br />

1. **创建自定义JMX模板**

   创建模板过程就不贴出来了，不清楚如何创建自定义模板的读者可以参考之前的文章[如何创建自定义模板](<https://www.cnblogs.com/Sherry-XRJ/p/11763732.html#create_template>)。这里创建了一个群组***Java Server Group***和自定义模板***Custom JMX Template***

2. **创建监控项**

   JMX监控项的**键值**格式为```jmx[object_name,attribute_name]```，其中*object_name*是MBean的ObjectName，*attribute_name*为需要读取的属性值。如何确定需要监控的数据可以参考[文末的JMX问题](#jmx_question)。

   <br />

   官方对键值的说明：

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191114185215617-169618358.png)

   读者可以查看[JMX监控项的官方文档](<https://www.zabbix.com/documentation/4.4/manual/config/items/itemtypes/jmx_monitoring>)

   <br />

   **堆内内存监控项**

   | 配置项     | 值                                                           |
   | ---------- | ------------------------------------------------------------ |
   | * 名称     | 堆内内存监控项                                               |
   | 类型       | JMX agent代理程序                                            |
   | * 键值     | jmx["java.lang:type=Memory","HeapMemoryUsage.used"]          |
   | * JMX 端点 | service:jmx:rmi:///jndi/rmi://{HOST.CONN}:{HOST.PORT}/jmxrmi |
   | 单位       | B                                                            |
   | 其他配置项 | 根据需要配置                                                 |

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191115100853402-535159992.png)

   <br />

   **JVM线程总数监控**

   | 配置项     | 值                                                           |
   | ---------- | ------------------------------------------------------------ |
   | * 名称     | JVM线程总数监控                                              |
   | 类型       | JMX agent代理程序                                            |
   | * 键值     | jmx["java.lang:type=Threading","ThreadCount"]                |
   | * JMX 端点 | service:jmx:rmi:///jndi/rmi://{HOST.CONN}:{HOST.PORT}/jmxrmi |
   | 其他配置项 | 根据需要配置                                                 |

   <br />

   **Tomcat请求总数监控**

   | 配置项     | 值                                                           |
   | ---------- | ------------------------------------------------------------ |
   | * 名称     | Tomcat请求总数监控                                           |
   | 类型       | JMX agent代理程序                                            |
   | * 键值     | jmx["Catalina:type=GlobalRequestProcessor,name=\"http-nio-8080\"",requestCount] |
   | * JMX 端点 | service:jmx:rmi:///jndi/rmi://{HOST.CONN}:{HOST.PORT}/jmxrmi |
   | 其他配置项 | 根据需要配置                                                 |

   其中\\"http-nio-8080\\" 是使用tomcat的默认端口8080，如果修改了端口这里也要做对应调整。作为模板配置可以把这个参数在模板中配置成*自定义宏*，在主机中可以修改宏的值。

   <br />

   **Tomcat每分钟请求监控**

   | 配置项     | 值                                                           |
   | ---------- | ------------------------------------------------------------ |
   | * 名称     | Tomcat每分钟请求监控                                         |
   | 类型       | JMX agent代理程序                                            |
   | * 键值     | change("jmx[\"Catalina:type=GlobalRequestProcessor,name=\\"http-nio-8080\\"\",requestCount]") |
   | * JMX 端点 | service:jmx:rmi:///jndi/rmi://{HOST.CONN}:{HOST.PORT}/jmxrmi |
   | 其他配置项 | 根据需要配置                                                 |

   <br />

   **JVM老年代已使用内存监控**

   | 配置项     | 值                                                           |
   | ---------- | ------------------------------------------------------------ |
   | * 名称     | JVM老年代已使用内存监控                                      |
   | 类型       | JMX agent代理程序                                            |
   | * 键值     | jmx["java.lang:type=MemoryPool,name=Tenured Gen", "Usage.used"] |
   | * JMX 端点 | service:jmx:rmi:///jndi/rmi://{HOST.CONN}:{HOST.PORT}/jmxrmi |
   | 其他配置项 | 根据需要配置                                                 |

   <br />

   **JVM老年代总内存监控**

   | 配置项     | 值                                                           |
   | ---------- | ------------------------------------------------------------ |
   | * 名称     | JVM老年代已使用内存监控                                      |
   | 类型       | JMX agent代理程序                                            |
   | * 键值     | jmx["java.lang:type=MemoryPool,name=Tenured Gen", "Usage.committed"] |
   | * JMX 端点 | service:jmx:rmi:///jndi/rmi://{HOST.CONN}:{HOST.PORT}/jmxrmi |
   | 其他配置项 | 根据需要配置                                                 |

   </br >

   **老年代内存使用比例监控**

   | 配置项     | 值                                                           |
   | ---------- | ------------------------------------------------------------ |
   | * 名称     | JVM老年代已使用内存监控                                      |
   | 类型       | 可计算的                                                     |
   | * 键值     | jmx.old                                                      |
   | * 公式     | 100 * (last("jmx[\\"java.lang:type=MemoryPool,name=Tenured Gen\\", \\"Usage.used\\"]"))<br /> / <br />last("jmx[\\"java.lang:type=MemoryPool,name=Tenured Gen\\", \\"Usage.committed\\"]") |
   | * 信息类型 | 浮点数                                                       |
   | 单位       | %                                                            |
   | 其他配置项 | 根据需要配置                                                 |

</br >

## 六.DashBoard创建图形

创建图形的步骤本文就忽略了，添加图形步骤可以参考[Zabbix-(三)监控主机CPU、磁盘、内存并创建监控图形](<https://www.cnblogs.com/Sherry-XRJ/p/11763732.html>)

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191115185806650-1107244672.png)

<br />

## 七.其他

<h3 id="jmx_question">如何确定JMX的object_name和attribute_name</h3>

jmx监控项格式： ```jmx[object_name,attribute_name]```

1. 使用jconsole连接到JVM，选择MBean

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191115190155609-1877645225.png)

   <br />

2. 找到需要监控的MBean，查看Bean信息和它的属性信息

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191115190437053-1026659117.png)

   <br />

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191115190630660-168363389.png)

   <br/>

   !![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191115191329736-587644324.png)

   <br />

3. 结合上面的步骤，那么zabbix监控项就可以填写为

   ***jmx["java.lang:type=Memory","HeapMemoryUsage.used"]***