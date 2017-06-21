### kubernetes下mysql集群的实现,galera-mysql

	docker pull index.tenxcloud.com/google_containers/mysql-galera:e2e




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