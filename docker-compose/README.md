### 一台服务器，多个mysql集群的实现

haproxy dockerfile:

	FROM index.tenxcloud.com/docker_library/haproxy:1.6.6
	COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg

运行：

	$ docker build -t my-haproxy .
	$ docker run -d --name my-running-haproxy my-haproxy


或者在运行时，添加cfg文件进docker,测试这种方式会出现问题，所以弃掉:

	docker pull index.tenxcloud.com/docker_library/haproxy:1.6.6

	docker run -d -it --name my-running-haproxy  -p 3307:3306 -p 4340:4399 -v /mnt/mnt/docker-compose/haproxy-config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg  --privileged=true index.tenxcloud.com/docker_library/haproxy:1.6.6

问题：

Q：出现权限拒绝

A：	

	方式1：docker run加--privileged=true

	方式2：临时关闭selinux：setenforce 0

Q2:haproxy.cfg挂载进去，变成了一个目录或者出现：

	[ALERT] 298/054910 (1) : [haproxy.main()] No enabled listener found (check for 'bind' directives) ! Exiting.

A:
	setenforce 0
	setsebool -P haproxy_connect_any=1

	弃用docker run -v挂载的方式，将文件直接复制进镜像，重新构建镜像

	docker build -t my-haproxy .

	docker run -d -it --name my-running-haproxy  -p 3307:3306 -p 4340:4399 my-haproxy

运行成功：

	Successfully built 57ad9fe84c93
	[root@localhost haproxy-config]# docker run -d -it --name my-running-haproxy  -p 3307:3306 -p 4340:4399 my-haproxy
	69500fa5e42950492d765619baebae14fc19cce82cc944406780647bf026ec1b
	[root@localhost haproxy-config]# docker ps -a
	CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                            NAMES
	69500fa5e429        my-haproxy          "/docker-entrypoint.s"   3 seconds ago       Up 1 seconds        0.0.0.0:3307->3306/tcp, 0.0.0.0:4340->4399/tcp   my-running-haproxy
	[root@localhost haproxy-config]# 


检测haproxy.cfg文件是否正确

	haproxy -f haproxy.cfg


### 1.环境安装


安装docker-compose

安装pip:

	wget https://pypi.python.org/packages/a9/23/720c7558ba6ad3e0f5ad01e0d6ea2288b486da32f053c73e259f7c392042/setuptools-36.0.1.zip#md5=430eb106788183eefe9f444a300007f0
	unzip setuptools-36.0.1.zip
	cd setuptools-36.0.1
	python setup.py install #安装

	
	wget https://pypi.python.org/packages/11/b6/abcb525026a4be042b486df43905d6893fb04f05aac21c32c638e939e447/pip-9.0.1.tar.gz#md5=35f01da33009719497f01a4ba69d63c9
	tar zxvf pip-9.0.1.tar.gz
	cd pip-9.0.1
	python setup.py install

安装docker-compose:	

	pip install docker-compose
	

### 2.使用

	
1.将docker-compose文件夹上传到服务器

	docker-compose up -d