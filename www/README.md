### 2台服务器+kubernetes部署方案


如果将整个集群部署在一台机器上，可能由于服务器占用已用的太多，所以，磁盘读写很高。现在将它拆开，分别放置到www放nodea,qa放nodeb,nodec。

### 1.环境：

www:139.196.16.67(10.174.113.12) 

qa:139.196.48.36(10.174.155.169)

注： ip地址为：外网（内网）


### 2.部署

#### www

nodea:

	docker run -d -p 4306:3306 -p 5567:5567 -p 5444:5444 -p 5568:5568  -e MYSQL_ROOT_PASSWORD=password  --name nodea erkules/galera:latest --wsrep-cluster-address=gcomm:// --wsrep-node-address=10.174.113.12:5567 --wsrep-sst-receive-address=10.174.113.12:5444 --wsrep-provider-options="ist.recv_addr=10.174.113.12:5568"



#### qa

nodeb:

	docker run -d -p 4306:3306 -p 5567:5567 -p 5444:5444 -p 5568:5568 -e MYSQL_ROOT_PASSWORD=password  --name nodeb erkules/galera:latest --wsrep-cluster-address=gcomm://10.174.113.12:5567 --wsrep-node-address=10.174.155.169:5567 --wsrep-sst-receive-address=10.174.155.169:5444 --wsrep-provider-options="ist.recv_addr=10.174.155.169:5568"


nodec:

	docker run -d -p 4307:3306 -p 5570:5570 -p 5445:5445 -p 5571:5571 -e MYSQL_ROOT_PASSWORD=password --name nodec erkules/galera:latest --wsrep-cluster-address=gcomm://10.174.155.169:5567 --wsrep-node-address=10.174.155.169:5570 --wsrep-sst-receive-address=10.174.155.169:5445 --wsrep_provider_options="base_port=5570;" --wsrep-provider-options="ist.recv_addr=10.174.155.169:5571"


数据持久化：

	docker run -d -p 4307:3306 -p 5570:5570 -p 5445:5445 -p 5571:5571 -v /mnt2/mysql-data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=password --name nodec erkules/galera:latest --wsrep-cluster-address=gcomm://10.174.155.169:5567 --wsrep-node-address=10.174.155.169:5570 --wsrep-sst-receive-address=10.174.155.169:5445 --wsrep_provider_options="base_port=5570;" --wsrep-provider-options="ist.recv_addr=10.174.155.169:5571"



nodea故障的处理，将docker删除之后，不要再用

	
	docker run -d -p 4306:3306 -p 5567:5567 -p 5444:5444 -p 5568:5568  -e MYSQL_ROOT_PASSWORD=password  --name nodea erkules/galera:latest --wsrep-cluster-address=gcomm:// --wsrep-node-address=10.174.113.12:5567 --wsrep-sst-receive-address=10.174.113.12:5444 --wsrep-provider-options="ist.recv_addr=10.174.113.12:5568"

因为这个会创建新的集群，不会链接之前的nodeb,nodec。要用这个：

	docker run -d -p 4306:3306 -p 5567:5567 -p 5444:5444 -p 5568:5568  -e MYSQL_ROOT_PASSWORD=password  --name nodea erkules/galera:latest --wsrep-cluster-address=gcomm://10.174.155.169:5567,10.174.155.169:5570 --wsrep-node-address=10.174.113.12:5567 --wsrep-sst-receive-address=10.174.113.12:5444 --wsrep-provider-options="ist.recv_addr=10.174.113.12:5568"
	



注：使用时，将上述password替换为自己想要的mysql登录密码。

### 3.负载均衡说明
	
	10.174.113.12：4306

	10.174.155.169：4306

	10.174.155.169：4307


### 4.数据备份,需要先安装garbd

	garbd --address gcomm://10.174.155.169:5567 gmcast.listen_addr=tcp://0.0.0.0:4444 \  --group example_cluster --donor example_donor --sst backup


mysqldump:

	mysqldump -h[server-ip-address] -u[username] -p[password] --all-databases --single-transaction > backup.sql


	mysqldump -P 端口 -h 127.0.0.1 -u 'root' -p'密码' -R 数据库  > backup.sql

可配合定时任务备份，凌晨3点备份：

	crontab -e

	0 3 * * * mysqldump -P 4306 -h 127.0.0.1 -u 'root' -p'password' -R 数据库 | gzip > /mnt2/mysql-backup/skiper-WWW-`date +\%Y-\%m-\%d_\%H.\%M.\%S`.sql.gz


### 5.恢复数据,恢复的时候..不要用mysqldump

	mysql -h 127.0.0.1 -P 4306 -u root -p yy_sys<backup.sql


[https://my.oschina.net/buptwangjing/blog/549976](https://my.oschina.net/buptwangjing/blog/549976)

[http://chuansong.me/n/1890505352823](http://chuansong.me/n/1890505352823)
	

说明：

	wsrep_cluster_address=gcomm://mysqlnode2,mysqlnode3
	集群中的其他节点地址，可以使用主机名或IP，一个集群成员之一的URL地址新的节点只需要连接到现有成员之一，就会自动检索集群地图和重新连接到其他节点

	wsrep_node_address='mysqlnode1'
	本机节点地址，可以使用主机名或IP


	wsrep_sst_donor='node3,'
	一个逗号分割的节点串作为状态转移源，比如wsrep_sst_donor=node5,node3, 如果node5可用，用node5,不可用用node3,如果node3不可用，最后的逗号表明让提供商自己选择一个最优的。

	wsrep_slave_threads=16
	线程数量。参考设置：1.CPU内核数*2以上;2.其它写节点连接总数的1/4.

	wsrep_sst_auth=tvmining:linuxidc.com
	xtrabackup使用的用户名密码


端口说明：

	3306，这个是MariaDB/MySQL的服务端口，这个都不开那就不用跑MariaDB/MySQL服务了。
	4567，Galera做数据复制的通讯和数据传输端口，需要在防火墙放开TCP和UDP
	4568，Galera做增量数据传输使用的端口（Incremental State Transfer, IST），需要防火墙放开TCP
	4444，Galera做快照状态传输使用的端口（State Snapshot Transfer, SST），需要防火墙放开TCP

修改端口：

	修改4567端口：wsrep_provider_options=’base_port=5567;’，同时wsrep_cluster_address也需要相应调整
	修改4568端口：wsrep_provider_options=’ist.recv_addr=192.168.1.102:5568;’
	修改4444端口：wsrep_sst_receive_address=’192.168.1.102:5569’
		
