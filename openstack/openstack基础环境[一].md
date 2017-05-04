## openstack
**OpenStack是什么?**  OpenStack是一个云操作系统，通过数据中心可控制大型的计算、存储、网络等资源池。所有的管理通过前端界面管理员就可以完成，同样也可以通过web接口让最终用户部署资源。    
本文希望通过提供必要的指导信息，帮助大家利用OpenStack前端来设置及管理自己的公共云或私有云。

![](http://i.imgur.com/lHi9gla.png)  

[**openstack官方网站**](http://www.openstack.org)  
[**中文文档**](http://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/)

### Openstack服务介绍  
- MySQL：为各个服务提供数据存储   
- RabbitMq：为各个服务之间通信提供认证和服务注册 
- Keystone：为各个服务器之间通讯提供认证和服务注册 
- Glance：为虚拟机提供镜像管理 
- Nova：为虚拟机提供计算资源 
- Neutron：为虚拟机提供网络资源  
### OpenStack基础环境 [一]
*提示：1、生产环境中必须保证openstack节点时间同步，如果时间不同步是无法创建虚拟机的。*   
*2、主机名解析。*  
#### openstack基础软件包安装  
基础软件包需要在所有的Openstack节点上进行安装，包括控制节点和计算节点。 
 
    rpm -ivh http://mirrors.aliyun.com/epel/epel-release-latest-6.noarch.rpm  
	yum install -y python-pip gcc gcc-c++ make libtool patch automake python-devel libxslt-devel MySQL-python openssl-devel libudev-devel git wget libvirt-python libvirt qemu-kvm gedit python-numdisplay python-eventlet device-mapper bridge-utils libffi-devel libffi

	[root@linux-node1 my.cnf.d]# vim /etc/my.cnf
	[mysqld]
	#bind-address = 192.168.21.128  # 监听的IP地址（也可以写0.0.0.0）
	default-storage-engine = innodb     # 默认存储引擎[innodb]
	innodb_file_per_table               # 使用独享表空间
	#max_connections = 4096        	    # 最大连接数是4096 （默认是1024）
	collation-server = utf8_general_ci  # 数据库默认校对规则
	character-set-server = utf8 	    # 默认字符集  
	init-connect = 'SET NAMES utf8'
##### 创建数据库 MySQL
	mysql> create database keystone;
	Query OK, 1 row affected (0.00 sec)

	mysql> grant all on keystone.* to keystone@'192.168.21.%/255.255.255.0' identified by 'keystone';
	Query OK, 0 rows affected (0.00 sec)

	mysql> create database glance;
	Query OK, 1 row affected (0.00 sec)

	mysql> grant all on glance.* to glance@'192.168.21.%/255.255.255.0' identified by 'glance';
	Query OK, 0 rows affected (0.00 sec)

	mysql> create database nova;
	Query OK, 1 row affected (0.00 sec)

	mysql> grant all on nova.* to nova@'192.168.21.%/255.255.255.0' identified by 'nova';
	Query OK, 0 rows affected (0.00 sec)

	mysql> create database neutron;
	Query OK, 1 row affected (0.00 sec)

	mysql> grant all on neutron.* to neutron@'192.168.21.%/255.255.255.0' identified by 'neutron';
	Query OK, 0 rows affected (0.00 sec)

	mysql> flush privileges;
	Query OK, 0 rows affected (0.00 sec)

	mysql> create database cinder;
	Query OK, 1 row affected (0.00 sec)

	mysql> grant all on cendir.* to cendir@'192.168.21.%/255.255.255.0' identified by 'cinder';
	Query OK, 0 rows affected (0.00 sec)

##### rabbitMQ
OpenStack使用消息队列来协调服务的操作和状态信息。消息队列服务通常在控制器节点上运行。   
OpenStack支持多消息队列服务包括RabbitMQ，QPID，和ZeroMQ。然而，大多数分布包OpenStack支持一个特定的消息队列服务。   

	[root@linux-node1 mysql]#rpm -ivh http://mirrors.aliyun.com/epel/epel-release-latest-6.noarch.rpm
	[root@linux-node1 mysql]#yum install rabbitmq-server -y
	[root@linux-node1 mysql]# /etc/init.d/rabbitmq-server start  
	Starting rabbitmq-server: SUCCESS  
	rabbitmq-server.
	
	[root@linux-node1 mysql]# /usr/lib/rabbitmq/bin/rabbitmq-plugins  list
	[ ] amqp_client                       3.1.5
	[ ] cowboy                            0.5.0-rmq3.1.5-git4b93c2d
	[ ] eldap                             3.1.5-gite309de4
	[ ] mochiweb                          2.7.0-rmq3.1.5-git680dba8
	[ ] rabbitmq_amqp1_0                  3.1.5
	[ ] rabbitmq_auth_backend_ldap        3.1.5
	[ ] rabbitmq_auth_mechanism_ssl       3.1.5
	[ ] rabbitmq_consistent_hash_exchange 3.1.5
	[ ] rabbitmq_federation               3.1.5
	[ ] rabbitmq_federation_management    3.1.5
	[ ] rabbitmq_jsonrpc                  3.1.5
	[ ] rabbitmq_jsonrpc_channel          3.1.5
	[ ] rabbitmq_jsonrpc_channel_examples 3.1.5
	[ ] rabbitmq_management               3.1.5   # web管理.
	[ ] rabbitmq_management_agent         3.1.5
	[ ] rabbitmq_management_visualiser    3.1.5
	[ ] rabbitmq_mqtt                     3.1.5
	[ ] rabbitmq_shovel                   3.1.5
	[ ] rabbitmq_shovel_management        3.1.5
	[ ] rabbitmq_stomp                    3.1.5
	[ ] rabbitmq_tracing                  3.1.5
	[ ] rabbitmq_web_dispatch             3.1.5
	[ ] rabbitmq_web_stomp                3.1.5
	[ ] rabbitmq_web_stomp_examples       3.1.5
	[ ] rfc4627_jsonrpc                   3.1.5-git5e67120
	[ ] sockjs                            0.3.4-rmq3.1.5-git3132eb9
	[ ] webmachine                        1.10.3-rmq3.1.5-gite9359c7

	[root@linux-node1 mysql]# /usr/lib/rabbitmq/bin/rabbitmq-plugins  enable rabbitmq_management
	The following plugins have been enabled:
	mochiweb
    webmachine
    rabbitmq_web_dispatch
    amqp_client
    rabbitmq_management_agent
    rabbitmq_management
    Plugin configuration has changed. Restart RabbitMQ for changes to take effect.
	[root@linux-node1 mysql]# /etc/init.d/rabbitmq-server restart
	Restarting rabbitmq-server: SUCCESS
	rabbitmq-server.
	
	[root@linux-node1 mysql]# lsof  -i:5672              
	COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
	beam    19968 rabbitmq   15u  IPv6 107726      0t0  TCP *:amqp (LISTEN)

	[root@linux-node1 mysql]# netstat -anput | grep 5672 
	tcp        0      0 0.0.0.0:15672               0.0.0.0:*                   LISTEN      19968/beam          
	tcp        0      0 0.0.0.0:55672               0.0.0.0:*                   LISTEN      19968/beam          
	tcp        0      0 :::5672                     :::*                        LISTEN      19968/beam   

	[root@linux-node1 mysql]# rabbitmqctl list_users
	Listing users ...
	guest	[administrator]
	...done.
![](http://i.imgur.com/OZa9krV.png)
![](http://i.imgur.com/uSPYNHZ.png)




