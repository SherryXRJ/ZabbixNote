# Zabbix-(五)监控Docker容器与自定义jvm监控项

## 一.前言

前文中讲述了Zabbix对服务器硬件方面的监控功能，本文将讲述利用Zabbix监控Docker容器中的Java Web服务，并通过自定义监控项，监控JVM老年代使用情况以及GC信息。Zabbix其实提供了[JMX监控](<https://www.zabbix.com/documentation/4.4/manual/config/items/itemtypes/jmx_monitoring>)，自带了JMX模板能够直接监控JVM信息，本文主要侧重于自定义参数与自定义监控项，关于JMX会在之后的文章中介绍。

### 准备

* Zabbix Server (Zabbix 4.4)  (ip:192.168.152.140)
* 运行Java应用的主机 以下简称Server-A （已被Zabbix监控） (ip:192.168.152.142)

</br>

## 二.开启agent用户自定义参数配置

1. **修改配置**

   使用自定义参数时，首先需要修改Server-A的agent配置

   ```shell
   # vim /etc/zabbix/zabbix_agentd.conf
   ```

   修改配置 UnsafeUserParameters=1

   ```properties
   UnsafeUserParameters=1
   ```

2. **重启zabbix-agent**

   ```shell
   # systemctl restart zabbix-agent
   ```

</br>

## 三.运行tomcat容器

在Server-A运行tomcat容器

```shell
# docker run --name tomcat -p 8080:8080 -dit tomcat:jdk8-adoptopenjdk-hotspot
```

将zabbix账号添加到docker组。参考[部署问题](#questions)

```shell
# sudo gpasswd  -a zabbix docker
```

外部访问测试一下

![访问tomcat](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191111184512016-401955423.png)

</br>

## 四.创建自定义Docker模板

我们可以定义一个比较通用的Docker模板，有服务需要被监控时，直接链接该模板即可。

1. **创建群组**

   点击【配置】-【主机群组】-【创建主机群组】

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191111190943743-133590066.png)

   

   定义一个组名 Docker Group

   | 配置项 | 值           |
   | ------ | ------------ |
   | * 组名 | Docker Group |

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191111191128523-1207545470.png)

   

2. <div id="create_docker_template"><strong>创建模板</strong></div>

   创建一个自定义模板，模板名称**Docker Template**，选择上步骤创建的**Docker Group**群组

   | 配置项     | 值              |
   | ---------- | --------------- |
   | * 模板名称 | Docker Template |
   | * 群组     | Docker Group    |
   
   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191111191630039-1429178377.png)

</br>

## 五.编写脚本与自定义监控参数

我们需要编写一个脚本，用于发现当前正在运行的docker容器（这里使用容器名称）。

1. **在Server-A编写发现运行容器的python脚本**

   创建脚本

   ```shell
   # cd /data/zabbix
   # touch find_container.py
   # chmod a+x find_container.py
   # vim find_container.py
   ```

   脚本内容：

   ```python
   #!/usr/bin/env python
   import os
   import json
   
   # 查看当前运行的docker容器
   t=os.popen(""" docker ps  |grep -v 'CONTAINER ID'|awk {'print $NF'} """)
   container_name = []
   for container in  t.readlines():
           r = os.path.basename(container.strip())
           container_name += [{'{#CONTAINERNAME}':r}]
   # 转换成json数据
   print json.dumps({'data':container_name},sort_keys=True,indent=4,separators=(',',':'))
   ```

   <div id="json_format">运行脚本，查看一下json数据格式：</div>

   ```json
   {
       "data":[
           {
               "{#CONTAINERNAME}":"tomcat"
           }
       ]
   }
   ```

2. <div id="custom_key"><strong>在Server-A自定义容器发现参数</strong></div>

   我们需要自定义一个**键值对**的配置类型，以便Zabbix可以通过键读取到值。

   增加自定义参数

   ```shell
   # cd /etc/zabbix/zabbix_agentd.d
   # vim userparameter_find_container.conf
   ```

   | 键               | 值                                                |
   | ---------------- | ------------------------------------------------- |
   | docker.container | /data/zabbix/find_container.py （脚本的运行结果） |

   ```properties
   UserParameter=docker.container,/data/zabbix/find_container.py
   ```

   

3. **在Server-A创建查看容器JVM GC情况的脚本**

   我们可以使用``` jstat -gcutil ``` 命令查看GC情况

   <img src='https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191113113957406-1161438204.png' align='left'/>

   </br>

   创建python脚本

   ```shell
   # cd /data/zabbix
   # touch monitor_gc.py
   # chmod a+x monitor_gc.py
   # vim monitor_gc.py
   ```

   脚本内容

   ```python
   #!/usr/bin/python
   import sys
   import os
   
   def monitor_gc(container_name, keyword):
           cmd = ''' docker exec %s bash -c "jstat -gcutil 1" | grep -v S0 | awk '{print $%s}' ''' %(container_name, keyword)
           value = os.popen(cmd).read().replace("\n","")
           print value
           
   if __name__ == '__main__':
           # 参数1：容器的名称
           # 参数2：查看第几列（例如 Eden区在第3列传入3，Full GC次数在第9列传入9）
           container_name, keyword = sys.argv[1], sys.argv[2]
           monitor_gc(container_name, keyword)
   ```

   测试脚本，查看当前tomcat容器Full GC次数

   ```shell
   # /data/zabbix/monitor_gc.py 'tomcat' '9'
   ```

   <img src='https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191113173437789-129716856.png' align='left'/>

   </br>

4. <div id="custom_gc_param"><strong>在Server-A自定义Zabbix JVM GC参数</strong></div>

   同样，增加一个conf文件，表示自定义参数

   ```shell
   # cd /etc/zabbix/zabbix_agentd.d
   # touch userparameter_gc_status.conf
   # vim userparameter_gc_status.conf
   ```

   | 键               | 值                               |
   | ---------------- | -------------------------------- |
   | jvm.gc.status[*] | /data/zabbix/monitor_gc.py $1 $2 |

   ```properties
   UserParameter=jvm.gc.status[*], /data/zabbix/monitor_gc.py $1 $2
   ```

   jvm.gc.status[*] 表示可以使用参数。其中$1表示参数1，即容器名称；$2表示参数2，需要查看哪项GC信息，$1 $2都是通过Zabbix配置时传递的。[Zabbix自定义参数](https://www.zabbix.com/documentation/4.4/manual/config/items/userparameters)

5. **在Zabbix server上测试自定义参数**

   为zabbix sever安装zabbix-get

   ```shell
   # yum install -y zabbix-get
   ```

   测试自定义参数，如果有权限问题，可以参考[部署问题](#questions)

   ```shell
   # zabbix_get -s 192.168.152.142 -p 10050 -k docker.container
   # zabbix_get -s 192.168.152.142 -p 10050 -k "jvm.gc.status['tomcat', 9]"
   ```

   <img src="https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191113174258189-497831388.png" align="left"/>

</br>

</br>

## 六.Zabbix模板增加自动发现规则

上述配置中，已经可以通过脚本获取到已运行的容器信息，此步骤将通过Zabbix配置界面，在模板中添加自动发现规则，以发现被监控主机中正在运行的docker容器，并利用这些获取的数据进一步监控容器中jvm数据。

1. **创建自动发现规则**

   点击【配置】-【模板】-【Docker Template】

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191112153920454-777309989.png)

   点击【自动发现规则】-【创建发现规则】

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191112154000925-2051346880.png)

   先配置【自动发现规则】

   | 配置项   | 值                                                  |
   | -------- | --------------------------------------------------- |
   | * 名称   | 发现正在运行的Docker容器规则                        |
   | 类型     | Zabbix 客户端                                       |
   | * 键值   | docker.container （这是我们上述步骤中自定义的键值） |
   | 其他配置 | 根据需要配置                                        |

   **键值**配置项是之前[自定义的监控键值](#custom_key)

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191112154314459-1413074682.png)

   再配置【过滤器】

   **宏**则配置自定义脚本返回json数据中的[key值](#json_format)

   | 配置项 | 值               |
   | ------ | ---------------- |
   | 宏     | {#CONTAINERNAME} |

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191112170817329-365489678.png)

2. **添加监控项原型**

   点击新建的自动发现规则的【监控项原型】-【创建监控项原型】

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191112172005842-601652591.png)

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191112172035749-1094526787.png)

   </br>

   输入参数

   | 配置项     | 值                                  |
   | ---------- | ----------------------------------- |
   | * 名称     | Tomcat Full GC次数监控项            |
   | 类型       | Zabbix 客户端                       |
   | * 键值     | jvm.gc.status[{#CONTAINERNAME} , 9] |
   | 其他配置项 | 根据需要填写                        |

   键值是[自定义jvm gc参数步骤中](#custom_gc_param)定义的参数，*{#CONTAINERNAME}* 是jvm.gc.status的参数1，使用了自动发现规则，发现到的docker容器名称(本文中即是 tomcat)；参数2 9 则是表示需要查看FullGC次数，FGC列（第9列）

   <img src='https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191113113957406-1161438204.png' align='left'/>

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191113173833733-1069961292.png)

   除此之外，还可以添加Old老年代(对应第4列)，Full GC时间(对应第10列)等监控项，这里就不一一添加了，和上述过程基本一致，只需修改参数2即可（也可以利用刚新建的监控项原型进行【克隆】）。

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191113173900877-1423596568.png)

   </br>

## 七.链接模板

将上述[自定义的模板](#create_docker_template)链接到Server-A主机

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191113151852022-1473994800.png)

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191113151917775-458816489.png)



## 八.DashBoard添加可视化图形

回到Zabbix首页可以为新增的自定义监控项，增加图形（添加图形步骤可以参考[Zabbix-(三)监控主机CPU、磁盘、内存并创建监控图形](<https://www.cnblogs.com/Sherry-XRJ/p/11763732.html>)）

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191113181746070-427000859.png)

## 九.其他

<h3 id="questions">部署问题</h3>

* **zabbix在执行脚本时，是使用的*zabbix*账户，因此可能要注意要给zabbix账号赋予权限。**

  例如，zabbix账户无法使用docker命令，将zabbix添加到docker组

  ```shell
  # sudo gpasswd -a zabbix docker
  ```

* **zabbix server无法执行agent自定义参数中的脚本**

  为agent主机设置

  ```shell
  # setenforce 0
  ```

  
