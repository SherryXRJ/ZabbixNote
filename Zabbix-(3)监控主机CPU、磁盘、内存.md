# Zabbix-(三)监控主机CPU、磁盘、内存并创建监控图形

## 一.前言

前文中已经讲述了两种方式对Zabbix的搭建，本文将讲述如何在zaibbx上添加需要监控的主机，以及使用Zabbix自带模板和自定义模板对主机的CPU、磁盘、内存进行监控，并触发问题，并且在Zabbix仪表盘创建实时监控图形。

### 准备

* Zabbix Server (Zabbix 4.4)  (ip:192.168.152.140)
* 被监控主机A  (Centos7.6)，下文简称 Server-A (ip:192.168.152.142) 
* 被监控主机B  (Centos7.6)，下文简称 Server-B (ip:192.168.152.143)

</br>

## 二.为被监控主机安装zabbix-agent

1. Server-A、Server-B分别安装zabbix-agent

   ```shell
   # rpm -Uvh https://repo.zabbix.com/zabbix/4.4/rhel/7/x86_64/zabbix-release-4.4-1.el7.noarch.rpm
   
   # yum install -y zabbix-agent
   ```

   </br>

2. Server-A、Server-B配置zabbix-agent

   ```shell
   # vim /etc/zabbix/zabbix_agentd.conf
   ```

   修改以下配置:

   * Server-A的zabbix_agentd.conf

   ```properties
   Server=192.168.152.140
   ServerActive=192.168.152.140
   
   # Hostname要与在Zabbix界面配置的Hostname(主机名称)保持一致
   Hostname=Server-A
   ```

   * Server-B的zabbix_agentd.conf

   ```properties
   Server=192.168.152.140
   ServerActive=192.168.152.140
   
   # Hostname要与在Zabbix界面配置的Hostname(主机名称)保持一致
   Hostname=Server-B
   ```

   </br>

3. 分别启动zabbix-agent

   ```shell
   # systemctl start zabbix-agent
   ```

   可以查看agent日志

   ```shell
   # tailf /var/log/zabbix/zabbix_agentd.log
   ```

   <div id="agent_host_not_found"></div>可能会出现以下内容，是由于zabbix界面上没有配置主机，接下来将在zabbix页面上进行主机配置

   ```txt
     6981:20191030:111132.151 no active checks on server [192.168.152.140:10051]: host [Server-A] not found
   ```

   </br>

## 三.Zabbix添加主机

通过页面操作，将需要监控的主机添加到zabbix中

</br>

1. **登录Zabbix，默认账号:Zabbix，默认密码:admin (可在zabbix数据库 users表查询)**

   ![Zabbix Dashboard](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030112702013-1417845140.png)

   </br>

2. <div id="create_host_base_info"><strong>点击【配置】-【主机】-【创建主机】，添加需要被监控的主机</strong></div>

   首先配置【主机】信息，添加Server-A，输入配置项

   | 配置项                | 值                                    |
   | --------------------- | ------------------------------------- |
   | * 主机名称            | Server-A                              |
   | 可见的名称            | Server-A                              |
   | * 群组                | Linux servers  (进行选择)             |
   | * agent代理程序的接口 | IP地址: 192.168.152.142   端口: 10050 |

   ![添加主机信息](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030113023989-959676857.png)

   

   <div id="link_template_os_linux_by_zabbix_agent">再配置【模板】信息，点击【添加】，选择群组<strong>Templates</strong>，勾选<strong>Template OS Linux by Zabbix agent</strong>，点击【选择】</div>

   ![选择链接模板](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030152929467-1795333932.png)

   ![添加主机链接模板](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030115004021-1613164389.png)

   最后点击【保存】

   </br>

3. **在【主机】页面可以看到Server-A已经成功添加了**

   ![主机添加成功](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030115715303-410088702.png)

   同时,Server-A的zabbix-agent日志也不再打印 [host [Server-A] not found](#agent_host_not_found)

   注: 由于在之前在安装Zabbix server时，也在zabbix server上安装了zabbix-agent，因此图例上除了**Server-A**主机以外，还有**zabbix server**主机

   </br>

4. **通过**全克隆**添加主机Server-B**

   选择需要复制的主机Server-A

   ![选择复制的主机](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030144855672-183352368.png)

   点击【全克隆】(full clone)

   ![全克隆](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030145151114-1175887281.png)

   修改**主机名称**、**agent IP地址**等信息

   | 修改配置项 | 值              |
   | ---------- | --------------- |
   | *主机名称  | Server-B        |
   | *agent IP  | 192.168.152.143 |

   ![修改克隆主机信息](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030145305517-762375553.png)

   最后点击【添加】，等待Server-B与zabbix server建立通信

   ![Server-B添加成功](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030151014139-1643274410.png)
   
   </br>

## 四.创建自定义模板(Template)

在添加主机步骤中，添加了2台需要监控的主机，添加**监控项**时也可以给每台主机单独添加监控项，但是随着主机数量增多，就会出现过多重复的操作，因此可以使用zabbix的**Templates(模板)**将**Items(监控项**和**Triggers(触发器)**等众多配置定义在模板中，将主机链接到定义好的模板上，就可以免去重复的操作。

下面将自定义模板，定义监控磁盘剩余空间监控项，并配置触发器当磁盘剩余空间低于一定阈值时触发告警。

</br>

1. <div id="create_template"><strong>创建自定义模板</strong></div>

   点击【配置】-【模板】-【创建模板】

   ![创建模板页面](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030160719896-1127765354.png)

   </br>

 2. **输入模板信息，完成后点击【添加】**

    | 配置项     | 值                      |
    | :--------- | ----------------------- |
    | * 模版名称 | Template Disk Free Size |
    | 可见的名称 | Template Disk Free Size |
    | * 群组     | Linux servers (选择)    |
    | 描述       | 自定义磁盘剩余空间模板  |

    ***注: 读者也可以自定义一个群组，并在自定义群组中创建模板，这个步骤本文不再示范***

    ![自定义模板信息](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030162442693-2057176733.png)

    </br>

## 五.创建磁盘剩余空间监控项和触发器

1. **创建自定义磁盘监控项(Item)**

   </br>

   进入自定义模板的监控项模块

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030162847777-896096475.png)

   点击【创建监控项】

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030162941643-370094834.png)

   输入监控参数

   | 配置项       | 值                  |
   | ------------ | ------------------- |
   | * 名称       | 磁盘剩余空间监控项  |
   | 类型         | Zabbix 客户端       |
   | * 键值       | vfs.fs.size[/,free] |
   | 单位         | B                   |
   | ……其他配置项 | 根据需要填写        |

   这里的键值 vfs.fs.size[/,free]是指，监控根路径下，空余的磁盘大小

   ![vfs.fs.size](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030165601130-780517915.png)

   点击【添加】

   ![磁盘监控项](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030183543582-438510700.png)

   ***注:创建监控项(Items)可以参考[官方创建监控项](<https://www.zabbix.com/documentation/4.4/manual/config/items/item>)， 更多的键值(Keys)可以参考[官方监控项类型](<https://www.zabbix.com/documentation/4.4/manual/config/items/itemtypes?_blank>)***

   </br>

2. <div id="create_trigger"><strong>创建触发器(Trigger)</strong></div>

   触发器可以配置当监控项监控到的数据达到一定阈值，从而触发问题。

   </br>

   在Template Disk Free Size模板中选择【触发器】，点击【创建触发器】

   ![点击创建触发器](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030170049032-2115100797.png)

   </br>

   输入触发器参数

   | 配置项                | 值                                                           |
   | --------------------- | ------------------------------------------------------------ |
   | * 名称                | 磁盘剩余空间触发器                                           |
   | 严重性                | 严重(选择)                                                   |
   | * 表达式/问题表现形式 | {Template Disk Free Size:vfs.fs.size[/,free].last()}<15000000000 (可通过选择监控项) |
   | 事件成功迭代          | 恢复表达式(选择)                                             |
   | * 恢复表达式          | {Template Disk Free Size:vfs.fs.size[/,free].last()}>=15000000000 |
   | 问题事件生成模式      | 多重(选择)                                                   |

   </br>

   表达式/问题表示形式

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030171241246-1524685233.png)

   </br>

   选择已配置的**磁盘剩余空间监控项**

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030171337153-1691735306.png)

   </br>

   配置**结果 <  15000000000**， 监控项中单位为B，这里15GB换算成15000000000B

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030183212009-806965528.png)

   </br>

   点击【插入】，可以看到如下表达式，表达式意思是，当检测到磁盘弓箭剩余不足15GB时，将触发问题

   ```txt
   {Template Disk Free Size:vfs.fs.size[/,free].last()}<15000000000
   ```

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030183309348-1264715710.png)

   </br>

   因此可以直接输入问题**恢复表达式**，即磁盘剩余空间高于15GB时，恢复问题

   ```txt
   {Template Disk Free Size:vfs.fs.size[/,free].last()}>=15000000000
   ```

   </br>

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030183334622-1638898657.png)

   点击【添加】

   </br>

   再将该自定义模板，链接到Server-A、Server-B主机的模板中，参考[创建主机添加链接](#link_template_os_linux_by_zabbix_agent)，不过在筛选模板时，群组要选择**Linux servers**(与创建模板时群组保持一致)，添加后点击【更新】

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030180402250-1868921353.png)

   </br>

   进入【配置】-【主机】-【Server-A】(或者 Server-B)-【监控项】中，可以搜索到*磁盘剩余空间监控项*已经添加成功

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030180810410-1875476428.png)

   ***注：如果监控项状态不为【已启动】可以查看zabbix server日志进行排查***

   </br>

3. **测试一下**

   </br>

   当前Server-A主机磁盘剩余空间，为15G

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030181945821-549127108.png)

   上传一些文件到Server-A，此时磁盘剩余空间为14G

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030182302111-1641322233.png)

   等待Zabbix监控到Server-A磁盘变化，查看仪表盘，出现**问题**，配置成功

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030184136152-606110359.png)

   删除Server-A大文件，等待Zabbix监控到主机磁盘恢复，仪表盘**问题**恢复

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030184433467-1824205768.png)

</br>

## 六.监控CPU空闲率

在添加主机时，由于已经链接了[**Template OS Linux by Zabbix agent**](#link_template_os_linux_by_zabbix_agent)模板（该模板还链接了**Template Module Linux CPU by Zabbix agent**等若干个其他模板），Template Module Linux CPU by Zabbix agent模板自带了许多监控项，其中包括**CPU idle time 监控项**，因此可以直接使用该监控项监控主机CPU空闲率数值，无需自定义监控项，只需要添加一个**触发器(Trigger)**来读取监控项触发告警即可。

*注: zabbix自带模板中，有许多监控项可以直接利用起来，无需再单独创建监控项，使用时可先在已有模板中查找下可用的监控项。*

</br>

1. **使用自带模板中监控项**

   直接使用**CPU idle time 监控项**即可，可以在【配置】-【主机】，【Server-A】的【监控项】中搜索到该监控项(在下图中可以看到该监控项链接了模板)

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191030154803420-1960990520.png)

   </br>

2. **在已有模板中添加触发器(trigger)**

   这里在模板**Template Module Linux CPU by Zabbix agent**添加一个触发器。

   </br>

   点击【配置】-【模板】搜索模板*Template Module Linux CPU by Zabbix agent*，并进入【触发器】配置

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031104333929-1682888328.png)

   </br>

   创建触发器操作流程参考上面步骤中的[创建自定义触发器](#create_trigger)，这里说明一下配置参数

   | 配置项            | 值                                                           |
   | ----------------- | ------------------------------------------------------------ |
   | * 名称            | CPU空闲率触发器                                              |
   | 严重性            | 严重 (选择)                                                  |
   | 表达式/问题表现式 | {Template Module Linux CPU by Zabbix agent:system.cpu.util[,idle].avg(5m)}>=80 |
   | 事件成功迭代      | 恢复表达式(选择)                                             |
   | * 恢复表达式      | {Template Module Linux CPU by Zabbix agent:system.cpu.util[,idle].avg(5m)}<80 |

   表达式/问题表现式：表示在5分钟内CPU平均空闲率如果高于80%，那么将触发问题

   </br>

   添加表达式示例:

   ![问题表现式](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031114036817-1648216691.png)

   </br>

   system.cpu.util[,idle]官方说明

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031105523589-1198913628.png)

   

   <div id="create_template_warning"><font style="color:red;font-weight:bold;font-style:italic;">注：这里修改了zabbix自带的模板(Template Module Linux CPU by Zabbix agent)，为其添加了一个新的触发器，在实际使用中，要谨慎操作，因为链接了该模板的主机触发器都会被修改，因此实际使用中需要对这种操作进行评估。</font></div>

3. **测试一下**

   等待5分钟，Zabbix server、Server-A、Server-B的CPU空闲率都高于80%，Dashboard界面触发了问题，由于Zabbix server主机也链接了[**Template OS Linux by Zabbix agent**](#link_template_os_linux_by_zabbix_agent)模板，因此修改**Template Module Linux CPU by Zabbix agent**模板，Zabbix server的CPU空闲率也被监控，所以在修改模板时要[注意](#create_template_warning)。

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031142521492-406939475.png)

</br>

## 七.监控内存占用率

在上面的步骤中添加了磁盘剩余空间、CPU空闲率监控，直接使用了**Zabbix 客户端**类型的监控项的键值，但是有些监控项可能不能直接获取，需要通过计算的方式来获取，例如监控内存占用率，虽然可以使用**vm.memory.size**这个键值，但是得到值并不是我们所期望的，参考下面官方的解释，虽然**mode**中有**pused (used, percentage)**，但是**"used"="total - free"** 而 **“available"="free + buffers + cached"**(*内核版本Linux<3.14*)，实际是想要的值：

```txt
(available - total) / total
```

因此需要使用**可计算的**键值类型

</br>

官方对vm.memory.size以及参数解释：

![官方vm.memory.size解释](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031150330230-835900575.png)

![参数说明](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031150416606-262286699.png)

​	</br>

1. **在Template OS Linux by Zabbix agent模板新增监控项**

   </br>

   | 配置项       | 值                                                           |
   | ------------ | ------------------------------------------------------------ |
   | * 名称       | 内存占用率监控项                                             |
   | 类型         | 可计算的                                                     |
   | * 键值       | memory.utilization  (自定义)                                 |
   | * 公式       | 100*(last("vm.memory.size[total]")-last("vm.memory.size[available]"))/last("vm.memory.size[total]") |
   | 信息类型     | 浮点数                                                       |
   | 单位         | %                                                            |
   | ……其他配置项 | 默认即可                                                     |

   自定义键值可自己输入，具体规则参考官方[键值规则](<https://www.zabbix.com/documentation/4.4/manual/config/items/item/key>)

   </br>

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031160924288-328553996.png)

</br>

这里就不再创建**触发器**了，感兴趣的读者可以自行创建，可参考上面的[触发器创建步骤](#create_trigger)



## 八.Dashboard创建图形

可以在首页仪表盘里创建图形，实时查看监控项的数据值。

</br>

1. **回到zabbix首页，点击【编辑仪表盘】-【添加构件】**

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031162443383-1488476685.png)

   </br>

2. <div id="create_image"><strong>创建磁盘剩余空间图形</strong></div>

   输入基本信息

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031162939483-1583728792.png)

   </br>

   添加【主机】和【监控项】

   左边一栏选择主机Server-A，右边一栏选择Server-A的磁盘监控项

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031163127514-276091078.png)

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031163421168-136121925.png)

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031163451432-775249713.png)

   ​	</br>

   再【添加新数据集】，同样操作将Server-B的磁盘监控也添加到图形中

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031163658471-1920432157.png)

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031163752746-784268469.png)

   </br>

3. **添加CPU空闲率图形**

   按照[上面的步骤](#create_image)，添加Server-A，Server-B的CPU空闲率图形

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031164156973-1441064175.png)

   </br>

4. **添加内存占用率图形**

   同样按照[上面的步骤](#create_image)，添加Server-A，Server-B的内存占用率图形

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031164521578-453610010.png)

   </br>

5. **保存设置并在仪表盘中查看**

   点击【保存设置】

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031164634413-8701487.png)

   </br>

   在仪表盘页面查看图形

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031164750983-1475112363.png)

   

## 九.参考文档

* [Zabbix4.4官方创建监控项](<https://www.zabbix.com/documentation/4.4/manual/config/items/item#unit_blacklisting>)
* [Zabbix4.4键值类型](<https://www.zabbix.com/documentation/4.4/manual/config/items/itemtypes>)
* [Zabbix4.4键值类型](<https://www.zabbix.com/documentation/4.4/manual/config/items/itemtypes>)
* [Zabbix4.4 vm.memory.size](<https://www.zabbix.com/documentation/4.4/manual/appendix/items/vm.memory.size_params>)