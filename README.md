### kubernetes下mysql集群的实现,galera-mysql

	docker pull index.tenxcloud.com/google_containers/mysql-galera:e2e

	官方镜像库：

	https://hub.docker.com/r/erkules/galera/

	https://github.com/erkules/codership-images



node1:

	docker run --detach=true --name node1 -h node1 erkules/galera:latest --wsrep-cluster-name=local-test --wsrep-cluster-address=gcomm://

node2:

	docker run --detach=true --name node2 -h node2 --link node1:node1 erkules/galera:latest --wsrep-cluster-name=local-test --wsrep-cluster-address=gcomm://node1

node3:

	docker run --detach=true --name node3 -h node3 --link node1:node1 erkules/galera:latest --wsrep-cluster-name=local-test --wsrep-cluster-address=gcomm://node1

查看：

	$ sudo docker exec -ti node1 mysql -e 'show status like "wsrep_cluster_size"'
	
	+--------------------+-------+
	| Variable_name      | Value |
	+--------------------+-------+
	| wsrep_cluster_size |     3 |
	+--------------------+-------+



### 具体实现：

We assume following 3 nodes
	
	nodea 10.10.10.10
	nodeb 10.10.10.11
	nodec 10.10.10.12

The following TCP port are used by Galera:

	3306-MySQL port
	4567-Galera Cluster
	4568-IST port
	4444-SST port

nodea：

	docker run -d -p 3306:3306 -p 4567:4567 -p 4444:4444 -p 4568:4568 --name nodea erkules/galera:latest --wsrep-cluster-address=gcomm:// --wsrep-node-address=10.10.10.10

nodeb:

	docker run -d -p 3306:3306 -p 4567:4567 -p 4444:4444 -p 4568:4568 --name nodeb erkules/galera:latest --wsrep-cluster-address=gcomm://10.10.10.10 --wsrep-node-address=10.10.10.11

nodec:

	docker run -d -p 3306:3306 -p 4567:4567 -p 4444:4444 -p 4568:4568 --name nodec erkules/galera:latest --wsrep-cluster-address=gcomm://10.10.10.10 --wsrep-node-address=10.10.10.12


测试：

	nodea$ docker exec -t nodea mysql -e 'show status like "wsrep_cluster_size"'
	+--------------------+-------+
	| Variable_name      | Value |
	+--------------------+-------+
	| wsrep_cluster_size |     3 |
	+--------------------+-------+



### BUILDING A MULTI-NODE CLUSTER USING NON-DEFAULT PORTS

In the long run, we may want to start more than one instance of Galera on a host in order to run more than one Galera cluster using the same set of hosts.

For the purpose, we set Galera Cluster to use non-default ports and then map MySQL’s default port to 4306:

	MySQL port 3306 is mapped to 4306
	Galera Cluster port 4567 is changed to 5567
	Galera IST port 4568 is changed to 5678
	Galera SST port 4444 is changed to 5444


nodea:

	docker run -d -p 4306:3306 -p 5567:5567 -p 5444:5444 -p 5568:5568 --name nodea erkules/galera:latest --wsrep-cluster-address=gcomm:// --wsrep-node-address=10.10.10.10:5567 --wsrep-sst-receive-address=10.10.10.10:5444 --wsrep-provider-options="ist.recv_addr=10.10.10.10:5568"


nodeb:

	docker run -d -p 4306:3306 -p 5567:5567 -p 5444:5444 -p 5568:5568 --name nodeb erkules/galera:latest --wsrep-cluster-address=gcomm://10.10.10.10:5567 --wsrep-node-address=10.10.10.11:5567 --wsrep-sst-receive-address=10.10.10.11:5444 --wsrep-provider-options="ist.recv_addr=10.10.10.11:5568"

nodec:

	docker run -d -p 4306:3306 -p 5567:5567 -p 5444:5444 -p 5568:5568 --name nodec erkules/galera:latest --wsrep-cluster-address=gcomm://10.10.10.10:5567 --wsrep-node-address=10.10.10.12:5567 --wsrep-sst-receive-address=10.10.10.12:5444 --wsrep-provider-options="ist.recv_addr=10.10.10.12:5568"

测试：

	nodea$ docker exec -t nodea mysql -e 'show status like "wsrep_cluster_size"'
	+--------------------+-------+
	| Variable_name      | Value |
	+--------------------+-------+
	| wsrep_cluster_size |     3 |
	+--------------------+-------+

说明:

The following Galera Cluster configuration options are used to specify each port:

	4567 Galera Cluster is configured using `–wsrep-node-address`
	4568 IST port is configured using `–wsrep-provider-options=”ist.recv_addr=”`
	4444 SST port is configured using `–wsrep-sst-receive-address`


### ==================上面是官网的步骤，现在实战==========

VM虚拟机实现：

	nodea:192.168.122.138
	nodeb:192.168.122.139
	nodec:192.168.122.141

每台服务器都先安装docker,然后下载镜像,此镜像跟官网一样，只是在国内下载快一点：

	yum install -y docker 
	systemctl start docker

	docker pull registry.cn-hangzhou.aliyuncs.com/zhg_docker_ali_r/galera:latest

	docker tag registry.cn-hangzhou.aliyuncs.com/zhg_docker_ali_r/galera:latest  erkules/galera:latest


运行：

nodea:

	docker run -d -p 4306:3306 -p 5567:5567 -p 5444:5444 -p 5568:5568 --name nodea erkules/galera:latest --wsrep-cluster-address=gcomm:// --wsrep-node-address=192.168.122.138:5567 --wsrep-sst-receive-address=192.168.122.138:5444 --wsrep-provider-options="ist.recv_addr=192.168.122.138:5568"


nodeb:

	docker run -d -p 4306:3306 -p 5567:5567 -p 5444:5444 -p 5568:5568 --name nodeb erkules/galera:latest --wsrep-cluster-address=gcomm://192.168.122.138:5567 --wsrep-node-address=192.168.122.139:5567 --wsrep-sst-receive-address=192.168.122.139:5444 --wsrep-provider-options="ist.recv_addr=192.168.122.139:5568"

nodec:

	docker run -d -p 4306:3306 -p 5567:5567 -p 5444:5444 -p 5568:5568 --name nodec erkules/galera:latest --wsrep-cluster-address=gcomm://192.168.122.138:5567 --wsrep-node-address=192.168.122.141:5567 --wsrep-sst-receive-address=192.168.122.141:5444 --wsrep-provider-options="ist.recv_addr=192.168.122.141:5568"
	

在nodea上测试一下：

	docker exec -t nodea mysql -e 'show status like "wsrep_cluster_size"'


navicat上连接：

	登录：root,密码空,端口:4306


检测集群是否正常的方法：

mysql客户端查看集群数量状态，如果集群节点数等于状态显示节点数，则表示机群搭建正常。

	mysql > show global status like 'wsrep_cluster_size';


在其中一台（nodec）上创建一个数据库：

	create database hellodb;

再查看nodea,nodeb，发现马上出现了一个数据库hellodb。

	show databases; 

	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| hellodb            |                  //数据库复制成功
	| mysql              |
	| performance_schema |             |
	+--------------------+

nodeb上创建表，并添加数据：

	create table students (id int unsigned auto_increment not null primary key,name char(30) not null,age tinyint unsigned,gender enum('f','m'));

	insert into students (name) values ("Hello");


nodea,nodeb,nodec上分别查看：

	select * from students;

都显示：

	+----+-------+------+--------+
	| id | name  | age  | gender |
	+----+-------+------+--------+
	|  2 | Hello | NULL | NULL   | //如果字段时自动增加的话，则时按集群的节点数显示下一个数据
	+----+-------+------+--------+
	
解决办法：

	1、设定一个全局分配ID生成器，解决数据插入时ID顺序不一致
	2、手动指定id号，不能自动生成



### 故障恢复

将其中一台（nodeb）关闭

	docker stop nodeb


再插入数据：

	insert into students (name) values ("World");  //一节点插入数据

nodea,nodec上显示：

	+----+-------+------+--------+
	| id | name  | age  | gender |
	+----+-------+------+--------+
	|  2 | Hello | NULL | NULL   | 
	+----+-------+------+--------+
	|  4 | World | NULL | NULL   | 
	+----+-------+------+--------+


开启nodeb上的mysql：

	docker start nodeb

再次查看数据，发现已经同步过来了。

	 select * from students;

	+----+-------+------+--------+
	| id | name  | age  | gender |
	+----+-------+------+--------+
	|  2 | Hello | NULL | NULL   | 
	+----+-------+------+--------+
	|  4 | World | NULL | NULL   | 
	+----+-------+------+--------+


### 搭配haproxy使用

nodea上安装,也可在随意一台服务器上:

		yum install -y haproxy #安装

		systemctl start haproxy #启动
	
将本库文件haproxy.cfg上传并替换/etc/haproxy/haproxy.cfg：

重启haproxy：
	
	systemctl restart haproxy

浏览器访问haproxy:

	http://192.168.122.138:4399/admin?stats

	用户名:zouhuigang

	密码:zouhuigang123456


navicat访问：

	链接：192.168.122.138 端口:3306

	用户名:root 密码：空


插入一条数据：

	insert into students (name) values ("haproxy"); 


查看全部数据，发现已全部同步到三台服务器中。



### 一台服务器上，多个docker-mysql集群的实现
	
	docker-compose的实现



问题：

Q1：当导入数据时，一张表有很多数据的时候，不能及时同步过去。

A1:MySQL/Galera集群只支持InnoDB存储引擎。如果你的数据表使用的InnoDB，需要转换为InnoDB，否则记录不会在多台复制。

	修改表类型:
	ALTER TABLE `pingan` ENGINE = INNODB;  #pingan表名字

	查看表类型:
	SHOW table STATUS FROM yy_sys;  #yy_sys数据库名字

	查看表创建的信息：
	SHOW CREATE TABLE `pingan`;

导出去掉表类型：

压缩：

	mysqldump  -P 3306 -h 127.0.0.1 -u 'tywyadmin' -p'tywyADMIN@20)^'  -R yy_sys --skip-create-options | gzip  > /mnt2/mysql-backup/yy_sys_2017_6_21.sql.gz


不压缩：

	mysqldump  -P 3306 -h 127.0.0.1 -u 'tywyadmin' -p'tywyADMIN@20)^'  -R yy_sys --skip-create-options  > /mnt2/mysql-backup/yy_sys_2017_6_21.sql




Q2：导入问题，[Err] 2006 - MySQL server has gone away,[Err] INSERT INTO `m_userinfo` VALUES 

查看mysql的运行时长:

	show global status like 'uptime';

mysql链接超时,查看

	show global variables like '%timeout';

	wait_timeout 是28800秒，即mysql链接在无操作28800秒后被自动关闭

查看是mysql允许最大的数据包，也就是你发送的请求：

	show global variables like 'max_allowed_packet';

	修改为100M：
	set global max_allowed_packet=1024*1024*100;

Q3:[Err] 1047 - WSREP has not yet prepared node for application use

A3:带处理



一些命令：

	MariaDB [(none)]> show variables like 'wsrep_on';
	+---------------+-------+
	| Variable_name | Value |
	+---------------+-------+
	| wsrep_on      | ON    |
	+---------------+-------+
	 
	MariaDB [(none)]> show status like 'wsrep_connected';
	+-----------------+-------+
	| Variable_name   | Value |
	+-----------------+-------+
	| wsrep_connected | ON    |
	+-----------------+-------+
	 
	MariaDB [(none)]> show status like 'wsrep_ready';
	+---------------+-------+
	| Variable_name | Value |
	+---------------+-------+
	| wsrep_ready   | ON    |
	+---------------+-------+

	wsrep_on 值为ON则说明启动成功。
	wsrep_connected值为ON说明连接到了集群。
	wsrep_ready值为ON说明已经准备好接受SQL请求了。该值最关键


注意事项：

1、使用Galera必须要给MySQL-Server打wsrep补丁。可以直接使用官方提供的已经打好补丁的MySQL安装包，如果服务器上已经安装了标准版MYSQL，需要先卸载再重新安装。卸载前注意备份数据。

2、MySQL/Galera集群只支持InnoDB存储引擎。如果你的数据表使用的InnoDB，需要转换为InnoDB，否则记录不会在多台复制。可以在备份老数据时，为mysqldump命令添加–skip-create-options参数，这样会去掉表结构的声明信息，再导入集群时自动使用InnoDB引擎。不过这样会将AUTO_INCREMENT一并去掉，已有AUTO_INCREMENT列的表，必须在导入后重新定义。

3、MySQL 5.5及以下的InnoDB引擎不支持全文索引（FULLTEXT indexes），如果之前使用InnoDB并建了全文索引字段的话，只能安装MySQL 5.6 with wsrep patch。

4、所有数据表必须要有主键（PRIMARY）,如果没有主键可以建一条AUTO_INCREMENT列。

5、MySQL/Galera集群不支持下面的查询：LOCK/UNLOCK TABLES，不支持下面的系统变量：character_set_server、utf16、utf32及ucs2。

6、数据库日志不支持保存到表，只能输出到文件（log_output = FILE），不能设置binlog-do-db、binlog-ignore-db。

7、跟其他集群一样，为了避免节点出现脑裂而破坏数据，建议Galera集群最低添加3个节点。

8、在高并发的情况下，多主同时写入时可能会发生事务冲突，此时只有一个事务请求会成功，其他的全部失败。可以在写入/更新失败时，自动重试一次，再返回结果。

9、节点中每个节点的地位是平等的，没有主次，向任何一个节点读写效果都是一样的。实际可以配合VIP/LVS或HA使用，实现高可用性。

10、如果集群中的机器全部重启，如机房断电，第一台启动的服务器必须以空地址启动：mysqld_safe –wsrep_cluster_address=gcomm:// >/dev/null &

参考文档：

[http://galeracluster.com/downloads/](http://galeracluster.com/downloads/)

[http://dockone.io/article/2016](http://dockone.io/article/2016)

[https://github.com/kubernetes/kubernetes/tree/685e421b89d8c67f680b15e5b4894263c8017da5/test/e2e/testing-manifests/statefulset/mysql-galera](https://github.com/kubernetes/kubernetes/tree/685e421b89d8c67f680b15e5b4894263c8017da5/test/e2e/testing-manifests/statefulset/mysql-galera)

[http://blog.csdn.net/cloud952788/article/details/39643921](http://blog.csdn.net/cloud952788/article/details/39643921)

[http://www.openskill.cn/article/400](http://www.openskill.cn/article/400)

[http://lanxianting.blog.51cto.com/7394580/1787391/](http://lanxianting.blog.51cto.com/7394580/1787391/)

[https://www.nginx.com/blog/mysql-high-availability-with-nginx-plus-and-galera-cluster/](https://www.nginx.com/blog/mysql-high-availability-with-nginx-plus-and-galera-cluster/)

官方文档：

[http://galeracluster.com/2015/05/getting-started-galera-with-docker-part-1/](http://galeracluster.com/2015/05/getting-started-galera-with-docker-part-1/)

[http://galeracluster.com/2015/05/getting-started-galera-with-docker-part-2-2/](http://galeracluster.com/2015/05/getting-started-galera-with-docker-part-2-2/)

[http://galeracluster.com/documentation-webpages/docker.html](http://galeracluster.com/documentation-webpages/docker.html)