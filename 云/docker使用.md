# docker

镜像：模板，可以通过这个模板来创建容器服务。通过镜像可以创建多个容器

容器：独立运行一个或多个应用，通过镜像来创建

仓库：存放镜像的地方，分为公有仓库和私有仓库



docker run：docker会在本机寻找镜像，本机上有就用本机上的，本机上没有就去docker hub上寻找镜像，找到就下载镜像到本机，找不到返回错误

docker是client-server架构，docker的守护进程运行在宿主机上，通过socket从client访问

docker server 接收到docker client的指令，连接docker守护进程，可以启动容器，容器内也有自己的端口号等（可以理解为小型linux虚拟机）

docker比虚拟机快的原因：

![img](https://img2.baidu.com/it/u=1484240172,1702752071&fm=26&fmt=auto&gp=0.jpg)

docker有比宿主机更少的抽象层；docker用的是宿主机的内核（新建容器时，docker不需要像虚拟机一样重新加载操作系统内核）



帮助命令

```
docker version
docker info
docker --help
```

镜像命令

```
docker images
REPOSITORY                     TAG            IMAGE ID       CREATED         SIZE
docker images --help
-a, --all 列出所有镜像
--digests 现实摘要
-f, --filter filter 过滤
-q, --quiet 只显示镜像的id
```

REPOSITORY 镜像仓库源 TAG 镜像标签 IMAGE ID 镜像ID CREATE 镜像的创建时间 SIZE 镜像的大小

```
docker search
docker search --help
-f, --filter filter 过滤
```

```
docker pull 镜像名:[tag]
docker pull ethereum/client-go 不写tag，默认就是latest
Using default tag: latest
latest: Pulling from ethereum/client-go 分层下载，docker image核心 联合文件系统
5843afab3874: Pull complete 
8f76e50b0265: Pull complete 
e6c5c6091d64: Pull complete 
Digest: sha256:cfd62706bb424b1b80f3d541d52b5c74b67952b7908a7d5a1ea8c9c980baee8e 签名
Status: Downloaded newer image for ethereum/client-go:latest
docker.io/ethereum/client-go:latest 真实下载
```

```
docker rmi -f 镜像ID
docker rmi -f $(docker images -aq)
docker rmi -f $(docker images | grep mcct | awk '{print $3}')
```

容器命令

有了镜像才能创建容器

```
docker run [可选参数] image
--name="Name" 容器名字
-d 后台方式运行
-it 使用交互方式运行，进入容器查看内容
-p 指定容器的端口
	-p 主机端口:容器端口
	-p 容器端口
-P 随机指定端口
```

```
docker run -it centos /bin/bash
exit 容器停止并退出
ctrl + P + Q 容器退出不停止
```

```
docker ps 当前在运行的容器
docker ps -a 当前在运行的容器+历史运行过的容器
-n 显示最近创建的容器
-q 只显示容器的编号
```

```
docker rm 容器id 不能删除正在运行的容器，强制删除使用rm -f
docker rm -f $(docker ps -aq) 删除所有容器
```

```
docker start 容器id
docker restart 容器id
docker stop 容器id
docker kill 容器id
```

其他命令

```
docker run -d 镜像名
docker ps 发现centos停止了
```

常见的坑，docker容器使用后台运行，就必须要有一个前台进程，docker发现没有前台进程应用，就会自动停止

```
docker logs -tf 容器id
docker run -d centos /bin/sh -c "while true;do echo test;sleep 1;done"
```

```
docker top 容器id 查看容器内部进程
```

```
docker inspect 容器id 查看容器元数据
```

```
一般使用后台方式运行的容器，现在要进入容器
docker exec -it 容器id /bin/bash 开启新的窗口
docker attach 容器id 进入正在运行的命令行，而不是开启新的窗口
```

```
docker cp 容器id:容器内路径 宿主机路径 将容器内数据拷贝到宿主机
```

 端口暴露

```
-p 3344:80 通过宿主机3344端口访问docker容器中80端口
```

````
docker stats 查看cpu状态
````



docker镜像，包含运行某个软件所需的所有内容，包括代码、运行时、库、环境变量和配置文件。应用可直接打包docker镜像，就可以直接跑起来。

docker镜像是由一层层文件系统组成，docker镜像最底层是bootfs，主要包含bootloader和kernel，bootloader主要是引导加载kernel。

rootfs，在bootfs之上，包含典型Linux系统中的/dev，/proc，/bin，/etc等标准目录和文件，rootfs是不同操作系统发行版，Ubuntu、Centos等

一个镜像pull下来时有6个层级；执行run时，只分为两层，原来pull的镜像是镜像层tomcat，而自己的所有操作都是容器层的



commit生成自己的新版本

```
docker commit 提交容器成为一个新的副本
docker commit -a="GSG" -m "add some webapps" 3v52t5it2 tomcat02:1.0
```



容器数据卷

数据在容器中，容器删除，数据丢失。这样是行不通的，需要将数据持久化。容器之间可以有一个数据共享的技术，docker容器中产生的数据，同步到本地。目录的挂载，将容器中的目录，挂载到linux上。

```
docker run -it -v 主机目录:容器内目录
```

可以在本地修改，容器内会自动同步；将容器删除，挂载在本地的数据卷还在



具名和匿名挂载

```
docker run -d -P --name nginx01 -v /etc/nginx nginx 只指定容器内的，没有指定容器外的卷
docker volume ls 查看所有的卷的情况，会出现一堆hash，属于匿名挂载
docker run -d -P --name nginx02 -v juming-nginx:/etc/nginx nginx 指定了容器内外，但是容器外没有具体路径
docker volume ls 查看所有的卷的情况，会出现容器的名字
docker volume inspect juming-nginx 可以查看具体的目录挂载地址
```

区分具名、匿名、指定路径挂载

```
-v 容器内路径 匿名
-v 卷名:容器内路径 具名
-v /宿主机路径:容器内路径 指定路径挂载
```

-v后可加:ro/:rw改变读写权限；ro只能通过宿主机操作，而不能在容器内操作。



dockerfile用来构建docker镜像的文件

通过脚本可以生成镜像，镜像是一层一层的，脚本一个个的命令，每个命令都是一层

```
FROM centos
VOLUME ["volume01","volume02"]
CMD echo "------end------"
CMD /bin/bash

docker build -f /home/test/dockerfile1 -t koo/centos:1.0 .
```

```
Sending build context to Docker daemon  2.048kB
Step 1/4 : FROM centos
 ---> 300e315adb2f
Step 2/4 : VOLUME ["volume01","volume02"]
 ---> Running in 4795600df3e4
Removing intermediate container 4795600df3e4
 ---> 52ac2840749d
Step 3/4 : CMD echo "------end------"
 ---> Running in a5eaf169c8fc
Removing intermediate container a5eaf169c8fc
 ---> b9eed86659c4
Step 4/4 : CMD /bin/bash
 ---> Running in 6c132197bf5e
Removing intermediate container 6c132197bf5e
 ---> 213fcec9a50b
Successfully built 213fcec9a50b
Successfully tagged koo/centos:1.0
```

会把挂载的卷放在 / 根目录下



数据卷容器，多个容器实现数据共享

```
docker run -it --name docker02 --volumes-from docker01 koo/centos:1.0 docker02共享docker01里数据卷
docker run -it --name docker03 --volumes-from docker01 koo/centos:1.0 docker03共享docker01里数据卷
```

删除docker01，docker02与docker03依然可以访问，说明文件是拷贝的概念，而不是三个容器共享的概念



环境配置

```
docker run -it -e 环境
```



dockerfile构建docker镜像可以分为四步

1. 编写一个dockerfile文件
2. docker build构建成为一个镜像
3. docker run运行镜像
4. docker push发布镜像（dockerhub）



dockerfile：构建文件，定义了一切步骤，源代码

docker images：通过dockerfile构建生成的镜像，最终发布和运行的产品

```
FROM 基础镜像，一切从这里开始构建
MAINTAINER 镜像是谁写的，姓名+邮箱
RUN 镜像构建的时候需要运行的命令
ADD 需要添加的包之类的内容
WORKDIR 镜像的工作目录
VOLUME 挂载卷，挂载的目录位置
EXPOSE 暴露端口配置
CMD 指定容器启动时需要执行的命令，只有最后一个会生效，可被替代
ENTRYPOINT 指定容器启动时需要执行的命令，可以追加命令
ONBUILD 当构建一个被继承dockerfile时，会运行ONBUILD指令
COPY 类似ADD，将文件拷贝到镜像中
ENV 构建时设置环境变量
```



自己加强centos版本

```
FROM centos
MAINTAINER koo<352326510@qq.com>
ENV MYPATH /usr/local
WORKDIR $MYPATH
RUN yum -y install vim
RUN yum -y install net-tools

EXPOSE 80

CMD echo $MYPATH
CMD echo "------end------"
CMD /bin/bash
```

```
docker build -f dockerfile -t mycentos .
docker run -it mycentos
```



可以拿到fabric-peer之后，通过docker history操作一下

```
docker history 镜像id 查看镜像的历史操作记录
```



CMD和ENTRYPOINT区别

```
FROM centos
CMD ["ls","-a"]

docker build -f dockerfile -t cmdtest .
docker run 容器id

但是想要追加-l命令 docker run 容器id -l，会报错
因为cmd的清理下 -l 替换了CMD ["ls","-a"]，而-l不是命令，所以会报错

而使用ENTRYPOINT ["ls","-a"]，再使用docker run 容器id -l，会执行成功，追加命令会追加在ENTRYPOINT后面
```



实战tomcat镜像

```
FROM centos
MAINTAINER koo<352326510@qq.com>
COPY readme.txt /usr/local/readme.txt
ADD XXX.tar.gz /usr/local/
ADD YYY.tar.gz /usr/local/
RUN yum install vim
ENV MYPATH /usr/local
WORK $MYPATH
ENV JAVA_HOME XXXXXXXXXXX
ENV CLASSPATH XXXXXXXXXXX
ENV PATH $PATH:JAVA_HOME/bin
EXPOSE 8080
CMD /usr/local/apache-tomcat-XXX/bin/startup.sh && tail -F /usr/local/XXXX/log.out
```

```
docker build -f dockerfile -t diytomcat .
docker run -d -p 9090:8080 --name tomcat1 -v /home/koo/build/tomcat/test:/usr/local/tomcat/webapps/test -v /home/koo/build/tomcat/logs/:/usr/local/tomcat/logs diytomcat
docker exec -it g473df7347 /bin/bash
```



发布自己的镜像

dockerhub

1. hub.docker注册自己的账号
2. 登录此账号
3. 在服务器上提交自己的镜像

```
docker login -p 密码 -u 用户名
```

4. 登录完毕后就可以提交镜像了，一步`docker push 用户名/镜像名:版本号`

提交时也按照层级一层层提交



docker save/docker load

可以打包成backup.tar，发送给别人



docker网络

理解docker0

```
ip addr

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 52:54:00:fa:6c:56 brd ff:ff:ff:ff:ff:ff
    inet 9.134.80.129/22 brd 9.134.83.255 scope global noprefixroute eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fefa:6c56/64 scope link 
       valid_lft forever preferred_lft forever
3: br0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether b6:8e:f9:5b:a3:d8 brd ff:ff:ff:ff:ff:ff
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:0c:62:ad:5c brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:cff:fe62:ad5c/64 scope link 
       valid_lft forever preferred_lft forever
6: br-9f8fa260cdf1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:22:1e:91:a8 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.1/16 brd 172.19.255.255 scope global br-9f8fa260cdf1
       valid_lft forever preferred_lft forever
7: br-f6d2ba951412: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:7e:ed:60:5e brd ff:ff:ff:ff:ff:ff
    inet 172.20.0.1/16 brd 172.20.255.255 scope global br-f6d2ba951412
       valid_lft forever preferred_lft forever
547: br-236c9f0d0c75: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:67:d9:05:93 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-236c9f0d0c75
       valid_lft forever preferred_lft forever
601: br-158f70a59d41: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:f9:ba:37:06 brd ff:ff:ff:ff:ff:ff
    inet 172.24.0.1/16 brd 172.24.255.255 scope global br-158f70a59d41
       valid_lft forever preferred_lft forever
    inet6 fe80::42:f9ff:feba:3706/64 scope link 
       valid_lft forever preferred_lft forever
```

lo是本机回环地址

eth1是腾讯个人服务器内网地址

docker0是docker地址

>docker是如何处理容器网络访问

```
docker exec -it tomcat01 ip addr
```

发现容器启动会得到一个eth0@XXX的地址，是docker分配的

宿主机可以ping通容器内部，容器是宿主机创建的，所以可以被宿主机ping通

docker0 172.19.0.1，相当于家中路由器的地址

每次启动docker容器，docker会给docker容器分配一个ip，只要安装了docker，就会有一个网卡docker0，使用桥接模式，使用的技术是evth-pair技术

启动一个容器后再次执行`ip addr`，我们发现容器带来的网卡都是一对对的

evth-pair是一对虚拟设备接口，都是成对出现了，一端连着协议，一端彼此相连，充当一对桥梁，连接各种虚拟网络设备的

```
docker exec -it tomcat02 ping 172.18.0.2 可以通过tomcat01 ping tomcat02 可以互相ping通
```

两个docker容器并不是直连的，docker02首先通过evth-pair技术，将请求发送给docker0，docker0要么广播，要么在路由表中寻找docker01的地址，tomcat01与tomcat02公用一个路由器docker0。



直接通过`docker exec -it tomcat02 ping tomcat01`是不能ping通的，但是可以`docker run -d -P --name tomcat03 --link tomcat02 tomcat` 这样可以正向tomcat03连接tomcat02，但反向不可以

可以通过`docker network inspect 容器id`实现



自定义网络

查看所有的docker网络模式

bridge：桥接

none：不配置网络

host：与宿主机共享网络

container：容器网络联通

```
docker run -d -P --name tomcat01 --net bridge tomcat 默认bridge
docker network create --driver bridge --subnet 192.168.0.0/16 --gateway 192.168.0.1 mynet 自定义一个网络
```



两个docker自定义网络连通

```
docker network connect mynet tomcat01
```

连通之后是将tomcat01加入到mynet，tomcat01就可以拥有两个网络，双虚拟网卡



docker退出状态码

| 状态码 | 描述                                                    |
| ------ | ------------------------------------------------------- |
| 0      | 表示正常退出                                            |
| 非0    | 表示异常退出（退出状态码采用chroot标准）                |
| 125    | Docker守护进程本身的错误                                |
| 126    | 容器启动后，要执行的默认命令无法调用                    |
| 127    | 容器启动后，要执行的默认命令不存在                      |
| 137    | 表明容器收到了SIGKILL信号，进程被杀掉，对应kill -9      |
| 139    | 表明容器收到了SIGSEGV信号，无效的内存引用，对应kill -11 |
| 143    | 表明容器收到了 SIGTERM 信号，终端关闭，对应kill -15     |

| 字符           | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| no             | 默认策略，在容器退出时不重启容器                             |
| no-failure     | 在容器非正常退出时（退出状态非 0），才会重启容器             |
| no-failure:3   | 在容器非正常退出时重启容器，最多重启 3 次                    |
| always         | 在容器退出时总是重启容器                                     |
| unless-stopped | 在容器退出时总是重启容器，但不考虑在 docker 守护进程启动时就已经停止了的容器 |

# docker-compose

批量容器编排，定义、运行多个容器

通过YAML配置文件

dockerfile——docker-compose.yaml——docker-compose up

服务services，容器、应用（web、redis、mysql...）

项目project，一组关联的容器

docker-compose up启动项目的流程：创建网络——执行docker-compose.yaml文件——启动服务

默认规则： 默认的服务名 文件名 _ 服务名 _ num，未来可能有多个服务器组成集群，num代表副本数量

网络规则：10个服务——>项目（项目中的内容都在同个网络下，域名访问）。如果在同一个网络下，我们可以直接通过域名访问

```
version 版本
services 服务
    服务1
        images
        build
        network
        ...
    服务2
        ...
其他配置，网络/卷，全局规则
volumes
ntworks
configs
```

```
version: '2'

volumes:
  orderer.example.com:
  peer0.org1.example.com:
  peer0.org2.example.com:

networks:
  test:

services:

  orderer.example.com:
    container_name:自定义容器名称，而不是生成默认名称
    image:指定容器运行的镜像
    environment:添加环境变量
    working_dir:工作目录路径
    command:容器启动后默认执行的命令
    volumes:数据卷挂载路径
    ports:暴露端口信息
    networks:
    entrypoint: /code/entrypoint.sh 指定服务容器启动后执行的入口文件

  peer0.org1.example.com:
    container_name:
    image:
    environment:
    volumes:
    working_dir:
    command:
    ports:
    networks:

  peer0.org2.example.com:
    container_name: peer0.org2.example.com
    image: hyperledger/fabric-peer:$IMAGE_TAG
    environment:
      #Generic peer variables
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # the following setting starts chaincode containers on the same
      # bridge network as the peers
      # https://docs.docker.com/compose/networking/
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=${COMPOSE_PROJECT_NAME}_test
      - FABRIC_LOGGING_SPEC=INFO
      #- FABRIC_LOGGING_SPEC=DEBUG
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_PROFILE_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
      # Peer specific variabes
      - CORE_PEER_ID=peer0.org2.example.com
      - CORE_PEER_ADDRESS=peer0.org2.example.com:9051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:9051
      - CORE_PEER_CHAINCODEADDRESS=peer0.org2.example.com:9052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:9052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org2.example.com:9051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org2.example.com:9051
      - CORE_PEER_LOCALMSPID=Org2MSP
    volumes:
        - /var/run/docker.sock:/host/var/run/docker.sock
        - ../organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/msp:/etc/hyperledger/fabric/msp
        - ../organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls:/etc/hyperledger/fabric/tls
        - peer0.org2.example.com:/var/hyperledger/production
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    ports:
      - 9051:9051
    networks:
      - test
  
  cli:
    container_name:
    image:
    tty:模拟一个伪终端
    stdin_open:打开标准输入，可以接受外部输入
    environment:
    working_dir:
    command:
    volumes:
    depends_on:设置依赖关系，先启动depends_on定义的服务，再启动cli
    networks:
```

