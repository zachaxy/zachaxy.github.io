---
title: docker基本使用
date: 2021-10-11 10:03:46
tags: 工具
---

>Docker 是一个开源的应用容器引擎，基于 Go 语言 并遵从 Apache2.0 协议开源。Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口（类似 iPhone 的 app）,更重要的是容器性能开销极低。
<!--more-->


镜像 vs 容器 区别?  镜像只读, 运行起来就是一个容器

docker 常用命令
- docker pull 镜像名<:tags>
- docker images 查看本地镜像
- docker run 镜像名<:tags>   ---> 创建容器(如果本地没有镜像,则会先进行拉取),启动应用
- docker ps ----> 查看正在运行中的容器, 要查看所有的,则使用 ps -a 
- docker rm <-f> 容器id   --->  删除容器
- docker rmi 镜像id  --> 直接删除镜像

以创建 tomcat 容器为例, 创建好了之后, 原本 Tomcat 启动的是 8080 端口,但是在物理机上直接访问是不通的,需要进行端口映射

// 8000: 物理机的端口;  容器里面 tomcat 本来监听的端口 
> docker run -p 8000:8080 tomcat

我没有启动起来...
有坑: https://blog.csdn.net/qq_45589050/article/details/104559125
需要改文件


创建后台容器,以便可以打开多个窗口
> docker run -p 8000:8080 -d tomcat 

停止后台容器
1. docker ps 
1. docker stop containerID
1. docker rm containerID (没有 stop, 直接 rm ,会报错, 可以直接 -f)



如何进入容器内部:
docker exec [-it] containerID  命令
-it: 采用交互式执行命令

eg: docker exec -it 13e2e5fe23f1  /bin/bash  
通过bash(容器内支持的,非 host 机器的命令)采用交互式命令,进入某容器内部

退出当前容器命令:
exit


docker run = docker create + docker start 
进入 running 状态
- docker kill : die -> kill 进程终止, docker start 会重新创建进程
- docker stop : die -> stop 进程还在, docker start 会恢复之前的进程
- docker restart : die -> start -> restart
- docker pause -> paused   
- docker unpause -> unpause -> running
- oom -> die (根据策略决定是否需要重启) -> start / stop 

基于以上的状态变化,好的习惯是使用 docker start/stop 启动/停止一个之前启动过的容器!!

这里要注意:如果你启动的镜像里面没有要开启运行的服务,那么该镜像在 start 后会立即 stop, eg: 纯 centos 镜像, 像 tomcat 这样的镜像,默认启动后会自动启动 tomcat,所以可以一直运行,

如果是纯 centos, 需要在启动的时候一直运行在后台,[使用方法](https://www.chengxulvtu.com/docker-run-centos/):
```
docker run -d -it centos /bin/bash
或
docker run -itd --privileged centos /usr/sbin/init
使用--privileged参数进入后,好像可以在内部启动服务,不使用的话,内部无法启动服务


docker attach <container-id>
或
docker exec -it <container-id>  /bin/bash
```
进入 centos 的 shell

# Dockerfile 构建镜像
按照命令生成镜像

docker build -t 机构id(或个人id)/镜像名<:tags> Dockerfile_DIR

- FROM: 设置基准镜像,  也可以 FROM scratch, 不依赖任何镜像,直接不写不行吗?
- MAINTAINER:维护者
- LABEL version = "1.0"   无具体作用,就是用来做注释的,建议添加,以备使用
- LABEL description = "描述描述"
- WORKDIR: 工作目录,其实每一个 docker 镜像运行起来之后,底层都是一个裁减迷你 linux(如果是使用别人制作好的镜像,eg:tomcat,其Dockerfile 肯定有 FROM centos 之类的引用的,只不过我们不用关心具体是什么 os,如果要我们自己只做,也是需要手动引入的,Docker不会默认帮你添加一个 mini Linux),需要设置该镜像运行起来后的具体目录 本质使用 shell 登录进来后的默认路径...(不要被 workdir 字面意思给骗了,这个在 docker 里面就是一个 cd 命令,++)
- ADD: 复制文件用的,用于在构建的时候,复制物理机文件到 docker 运行起来的目录中
ADD 还具备解压缩功能, ADD test.tar.gz / 的含义是,将 test 解压到根路径
- COPY 纯拷贝命令,跟 ADD 类似, xxxx
- ENV JAVA_HOME /user/local/jdk8  设置环境变量
- EXPOSE 80 对外暴露容器的 80 端口,这样启动容器的时候,就不用加 -p 端口映射选项了


RUN: 在镜像构建时执行的命令,一个镜像一旦被构建成功后就是只读的,不能再修改了, 这个是为一可以在构建时修改内部文件的方式
RUN yum install -y vim #使用 shell命令格式,当前进程会创建子进程来执行该 shell 命令,执行完毕后子进程退出
RUN ["yum", "install", "-y", "vim"] #exec 命令格式, 创建进程替换原有的进程,同时集成原有的 pid,执行完毕后直接退出进程; 不清楚用哪种,则用这种...


ENTRYPOINT: 容器启动时执行的命令,省得你自己写脚本了...
写多个 ENTRYPOINT 命令,只有最后一个起作用..,也是推荐使用 exec 格式


CMD: 容器启动后执行默认的命令或者参数; 同样推荐使用 exec 方式执行; 但是如果你启动 docker image 的时候命令中带了命令,则该命令不会被执行...
eg: docker run xxx ls  # 表明启动 xxx 镜像,并执行 ls 命令,这样的话,xxx的 Dockerfile 中的 CMD 则不会被执行..

```shell
mkdir my_docker_dir
cd my_docker_dir
echo "hello">> index.html

code Dockerfile 
FROM tomcat:latest   # 拉取最新 tomcat 镜像
MAINTAINER zachaxy  # 维护者
WORKDIR /user/local/tomcat/webapps # 切换工作目录(shell 登录后默认路径),没有则自动创建,建议使用绝对路径!!!
ADD docker-web1 ./docker-web2  # 拷贝当前的 docker-web1 到以工作目录为 base_path 下的 docker-web2 目录下




docker build -t zachaxy/mywebapp:1.0 .   # 这里镜像名字你自己起,只要在 Dockerfile 目录中执行命令就行


docker image # 既可看到已创建的 image
```

启动 centos 的一个坑,必须要用 docker run -d  -it centos /bin/bash 的形式
否则 centos 一启动,容器就执行完自动退出了,我们必须以后台 + 命令行进入的方式,保证其稳定的运行,如果 centos 启动后默认启动了一个服务(eg:redies),则不用担心

# 镜像分层
- 镜像层 
- 容器层

Dockerfile 中的 FROM 设置基准镜像,回去远程下载,如果此时有另一个 Dockerfile 开头使用了同一个 docker 基准镜像,那么复用的是前一个



# docker 之间的通信

## 单向通信
eg: tomcat容器需要访问mysql容器来查询数据,而 mysql 则不需要访问 tomcat
docker 容器一旦创建,则由系统分配一个虚拟 ip, 仅仅是用来标识, 而且能确保同一台设备上,多个容器 ip 不同,默认多个容器之间是互通的(通过 ip 的方式互同),问题是容器每次启动后对应的 ip 不是固定的...


```
docker run -d --name web tomcat
docker run -d --name database mysql
```

启动容器的时候,通过--name 的方式,为容器命名,这样在 tomcat 内部访问 mysql 的时候,可以不用虚拟 ip,而是使用 database,就可以直接访问到 mysql 容器了


ps:怎么知道 docker 容器的 ip?
- 确实有这种机制,但是获取起来似乎非常麻烦 参考:[如何获取 Docker 容器的 IP 地址](https://chinese.freecodecamp.org/news/how-to-get-a-docker-container-ip-address-explained-with-examples/),放弃吧,还是给容器命名吧,简单点...

tomcat 单向访问 mysql 的方式:
```
docker run -d --name web --link database tomcat  
```
在启动 tomcat 的时候,额外添加一个 --link 的指示,就链接了mysql,这样在 tomcat 中就可以使用ping名字,可以直接 ping 通 mysql 

```
ping database
```


## 双向通信
使用网桥,完全虚拟出来的组件,每个 docker 容器都可以访问网络,正式因为网桥的存在.
那么只要将两个容器都绑定到网桥上,就可以互同了

```
docker network ls
```
类型: bridge/host/null

创建新的网桥:
```
docker network create -d bridge my-bridge
```


将新建的网桥与创建的容器进行绑定,那么绑定在同一个网桥下的容器默认就是互通的

```
docker network connect my-bridge web
docker network connect my-bridge database
```

每创建一个网桥,其实就是创建了一个虚拟网卡,该虚拟网卡的作用就是作为网关,挂载在该网桥下的容器以该虚拟网卡作为网关,同时改虚拟网卡还承担了地址转换的功能,从而实现容器向物理网卡发送数据的功能和接受数据的功能


## 使用 volume 进行容器的数据共享

使用 docker 部署 tomcat 文件,多实例但是代码文件都是一样的,如何一改都改?
在宿主机开辟磁盘空间,实现多个容器共享磁盘.


### 创建挂载宿主机目录
创建挂载宿主机目录,每次启动其它容器时挂载该目录

实例:`docker run --name 容器名 -v 宿主机共享路径:容器内映射路径 镜像名`
```
docker run --name web1 -v /usr/webapps:/usr/local/tomcat/webapps tomcat:latest
```

> 缺点: 每次启动都要这样设置有点麻烦啊...可以用entry_point 吗,不行,因为这个是 docker 本身的命令

### 创建共享容器
创建一个共享容器,每次启动其它容器时,挂载该容器
```
docker create --name webpage_share -v /usr/webapps:/usr/local/tomcat/webapps share_container /bin/true
```
创建容器,但是不运行,只是设置了挂载点,设置了容器名,作用是启动其它容器时挂载上去即可


启动其它容器挂载共享容器:
```
docker run  --volumes-from webpage_share --name web1 -d tomcat
```

> 优点:路径统一,如果修改,只要修改共享容器的路径即可
> 缺点:路径统一,满足不了定制需求...

# Docker-compose
> 多容器一键部署,容器编排工具,通过 yml 文件定义

docker-compose.yml
```
version: '3.3'  # : 后面必须有空格, yml 的版本,一般默认 3.3 版本,用来解析 yml
service:
  db:  # 子节点,使用空格缩进,一般是两个空格; 同时给容器命名,类似单独启动容器时 --name 指定的名字
    build: docker_file_path  # 表示对那个路径下的 Dockefile 进行创建
    restart: always # 一旦容器意外退出,那么自动重启
    environment: 
        xxxx: yyy   #配置环境变量,其实相当于单独启动容器时,额外的 -e 环境变量参数
  app:
    build: docker_file_path
    depends_on:         # 有先后依赖关系,同时默认是互通的,不用设置网桥了...
      - db
    ports:
      - "80:80"  # 这只端口映射, 宿主机端口:容器端口
    restart: always 

```


构建compose,直接执行下面命令,会默认从当前路径寻找 docker-compose.yml 文件进行构建, -d 参数后台运行, 查看日志:
```
docker-compose up -d logs  # 后台运行,但是打印日志
```


docker-compose down 移除容器