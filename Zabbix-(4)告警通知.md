# Zabbix-(四)邮件、钉钉告警通知

## 一.前言

在之前的文章里，通过Zabbix对主机的磁盘、CPU以及内存进行了监控，并在首页Dashboard里创建了监控图形，但是只有当我们登录到Zabbix后才能看到监控到的**问题(Problem)**，因此在本篇文章里，将利用**触发器(Trigger)**，以及**媒介(Media)**等配置项，实现当**触发器**触发时，通过不同**媒介**，如：邮件、钉钉，发送**动作(Action)**，实现实时通知告警功能。

### 准备

* Zabbix Server (Zabbix 4.4) 
* 在Zabbix中已配置一些监控项和触发器(这些配置可以参考我的[上一篇文章](<https://www.cnblogs.com/Sherry-XRJ/p/11763732.html>))



## 二.安装相关环境

由于使用到脚本告警媒介，本文中通过调用Python脚本触发告警，因此需要在Zabbix Server主机上安装pip以及相关模块。(这里Python使用Centos7自带的Python2.7.5)

1. **安装pip**

   ```shell
   # yum install -y epel-release
   
   # yum install -y python-pip
   ```

2. **安装requests模块**

   ```shell
   # pip install requests
   ```

</br>

## <div id="create_media_type">三.配置告警媒介类型</div>

Zabbix默认自带了2种**报警媒介类型(Media Type)**，*电子邮件*以及*短信*，我们将修改电子邮件类型配置，并新建*脚本*类型和*Webhook*类型。希望通过*脚本、Webhook*告警媒介发送钉钉消息。

***注：Webhook告警媒介是Zabbix 4.4的新特性***



1. **修改电子邮件告警媒介**

   点击【管理】-【报警媒介类型】-【Email】

   ![](https://img2018.cnblogs.com/blog/1697941/201910/1697941-20191031183635522-1294953703.png)
   
   </br>
   
   修改Email配置，我这里用的是Outlook邮箱，具体SMTP服务器可以参考[Outlook官网 SMTP设置](https://support.office.com/zh-cn/article/outlook-com-%E7%9A%84-pop%E3%80%81imap-%E5%92%8C-smtp-%E8%AE%BE%E7%BD%AE-d088b986-291d-42b8-9564-9c414e2aa040)。使用其他邮箱也可以去对应官网查询SMTP配置。
   
   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101104247915-530808683.png)
   
   </br>
   
   测试发送邮箱，点击【测试】
   
   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101112215655-936873407.png)
   
   </br>
   
   输入收件人邮箱
   
   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101112245454-2052086324.png)
   
   </br>
   
   收到邮件
   
   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101111700091-2057408288.png)
   
   </br>

2. **新增脚本告警媒介**

   新建Python脚本告警媒介，用户钉钉告警

   </br>

   点击【创建媒体类型】

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101114052469-1153568048.png)

	</br>
	
	进行配置
	
	| 配置项          | 值              |
	| --------------- | --------------- |
	| * 名称          | Python脚本      |
	| 类型            | 脚本            |
	| * 脚本名称      | pythonScript.py |
	| 脚本参数(参数1) | {ALERT.MESSAGE} |
	| 脚本参数(参数2) | {ALERT.SENDTO}  |
	| 脚本参数(参数3) | {ALERT.SUBJECT} |
	
	![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101114440619-1411831305.png)
	
	</br>
	
	接下来新建Python脚本，Zabbix Server配置文件中可以配置告警脚本路径，默认为 /usr/lib/zabbix/alertscripts
	
	```shell
	# 查看告警脚本路径
	# cat zabbix_server.conf | grep AlertScriptsPath
	```
	
	编写告警脚本
	
	```shell
	# cd /usr/lib/zabbix/alertscripts
	# vim pythonScript.py
	```
	
	脚本内容
	
	```python
	#!/usr/bin/env python
	#coding:utf-8
	
	import requests,json,sys,os,datetime
	
	# 钉钉机器人地址
	webhook="https://oapi.dingtalk.com/robot/send?access_token=your_dingding_robot_access_token"
	
	# 对应{ALERT.SENDTO}, Zabbix告警媒介配置界面第2个参数
	user=sys.argv[2]
	
	# 对应{ALERT.MESSAGE}, Zabbix告警媒介配置界面第1个参数
	text=sys.argv[1]
	data={
	    "msgtype": "text",
	    "text": {
	        "content": text
	    },
	    "at": {
	        "atMobiles": [
	            user
	        ],
	        "isAtAll": False
	    }
	}
	headers = {'Content-Type': 'application/json'}
	x=requests.post(url=webhook,data=json.dumps(data),headers=headers)
	```
	
	给脚本可执行权限
	
	```shell
	# chmod uo+x /usr/lib/zabbix/alertscripts/pythonScript.py
	```
	
	测试脚本
	
	![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101144900400-723121653.png)
	
	</br>
	
	钉钉收到消息
	
	![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101180947650-1794626710.png)
	
	</br>

3. **新增Webhook告警媒介**

   | 配置项         | 值              |
   | -------------- | --------------- |
   | * 名称         | Webhook         |
   | 类型           | Webhook         |
   | 参数: （名称） | 值              |
   | user           | {ALERT.SENDTO}  |
   | subject        | {ALERT.SUBJECT} |
   | message        | {ALERT.MESSAGE} |

   脚本:

   ```javascript
   try {
       Zabbix.Log(4, 'params= '+value);
    
       params = JSON.parse(value);
       req = new CurlHttpRequest();
       data = {};
       result = {};
    
       req.AddHeader('Content-Type: application/json');
    
       data.msgtype = "text";
       //	对应 message参数
       data.text = {"content" : params.message};
       //	对应 user参数
       data.at = {"atMobiles": [params.user], "isAtAll": "false"};
   
       //	钉钉机器人
       resp = req.Post('https://oapi.dingtalk.com/robot/send?access_token=your_access_token',
           JSON.stringify(data)
       );
   } catch (error) {
       result = {};
   }
    
   return JSON.stringify(result);
   ```

   </br>

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101162152998-1082779532.png)

   </br>

   测试Webhook

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101171638637-1748802669.png)

   </br>

​		![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101180839426-1692916362.png)

</br>

## 四.为用户添加告警媒介

需要将新增的告警媒介添加给用户



点击【用户】-【告警媒介】

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101175413827-2086326352.png)

将上述步骤添加的告警媒介(Python脚本、Webhoob、Email)，进行添加(收件人根据告警媒介类型填写*邮箱*或*手机号*)，严重性也根据需要勾选。

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101175921916-1457460650.png)

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101180152031-1106695095.png)



## 五.配置动作

完成上述配置完成后，需要创建**动作(Action)**，将**触发器(Trigger)**和**告警媒介(Media Type)**进行关联，一旦触发器触发，那么Zabbix会执行动作，再去执行告警媒介。

1. **添加动作**

   点击【配置】-【动作】-【创建动作】

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101163705948-1562311444.png)

   </br>

2. **配置【动作】相关信息**

   | 配置项       | 值                                                           |
   | ------------ | ------------------------------------------------------------ |
   | * 名称       | 告警动作                                                     |
   | 新的触发条件 | 【触发器】【等于】【Template Disk Free Size: 磁盘剩余空间触发器】 |

   操作步骤如下图：

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101165319838-1750798604.png)

   </br>

   群组选择 ->Linux servers

   主机选择 -> Template Disk Free Size 模板([上一篇文章中定义的模板](<https://www.cnblogs.com/Sherry-XRJ/p/11763732.html#create_template>))

   勾选触发器 -> 磁盘剩余空间触发器 ([上一篇文章中创建的触发器](<https://www.cnblogs.com/Sherry-XRJ/p/11763732.html#create_trigger>))

   勾选后点击【选择】

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101165535835-588172634.png)

   </br>

3. **配置【操作】相关信息**

   点击【操作】

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101170141650-1742111746.png)

   </br>

   先配置以下信息

   | 配置项                 | 值                                                           |
   | ---------------------- | ------------------------------------------------------------ |
   | * 默认操作步骤持续时间 | 1h（保持默认）                                               |
   | 默认标题               | 告警: {EVENT.NAME}                                           |
   | 消息内容               | 【磁盘空间不足告警】<br/>告警事件: {EVENT.DATE} {EVENT.TIME}  <br/>告警问题: {EVENT.NAME}<br/>告警主机: {HOST.IP} {HOST.NAME}<br/>告警级别: {EVENT.SEVERITY}<br/>磁盘剩余:{ITEM.VALUE} |

   上述配置表格【默认标题】和【消息内容】值中形如{EVENT.NAME}的内容是Zabbix中的**宏(Marco)**，宏是一个变量，例如 {HOST.IP} 表示告警主机的IP地址，Zabbix自带的宏可以参考[Zabbix 4.4自带宏](<https://www.zabbix.com/documentation/4.4/manual/appendix/macros/supported_by_location>)

   </br>

   继续配置操作

   <div id="new_operate">点击【新的】</div>

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101171855767-571009836.png)

   </br>

   【操作类型】选择发送消息，【发送到用户】添加Admin

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101172107858-1429109527.png)

   </br>

   【仅送到】根据需要选择之前配置的[告警媒介](#create_media_type)，本文选择Email和Python脚本(这里只能单选或全选，所以需要先选择一个，因此需要多次[添加](#new_operate))

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101172846402-1013568127.png)

   </br>

   添加完成后点击【添加】

   ![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101173124322-592629852.png)

   </br>

## 六.测试

向被监控主机拷贝或下载大文件，使其磁盘剩余空间低于触发器监控阈值，等待触发器触发问题，查看仪表盘、邮件等。

仪表盘

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101180445231-2072889764.png)

</br>

钉钉

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101180623132-1648275229.png)

</br>

邮件

![](https://img2018.cnblogs.com/blog/1697941/201911/1697941-20191101180714780-211451885.png)

## 七.参考文档

* [Zabbix 4.4 自定义脚本媒介](<https://www.zabbix.com/documentation/4.4/manual/config/notifications/media/script>)
* [Zabbix 4.4 Webhook](<https://www.zabbix.com/documentation/4.4/manual/config/notifications/media/webhook>)
* [Zabbix 4.4宏](<https://www.zabbix.com/documentation/4.4/manual/config/macros>)