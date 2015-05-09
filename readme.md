#cSphere入群docker基础：

这篇基础文章是方便cSphere用户在使用cSphere平台之前了解的基础。
针对的用户是已经有一定的Linux基础知识。

## Docker是什么
Docker是一个改进的容器技术。具体的“改进”体现在，Docker为容器引入了镜像，使得容器可以从预先定义好的模版创建出来，并且这个模版还是分层的，后面再细说分层。

Docker经常被提起的特点：
轻量，体现在内存占用小，高密度
快速，毫秒启动
隔离，沙盒技术更像虚拟机

Docker技术的基础：
namespace，容器隔离的基础，保证A容器看不到B容器. 6个名空间：User,Mnt,Network,UTS,IPC,Pid
cgroups，容器资源统计和隔离。主要用到的cgroups子系统：cpu,blkio,device,freezer,memory
unionfs，典型：aufs/overlayfs

Docker组件：
docker客户端  向docker服务器进程发起请求，如创建容器，销毁容器等
docker服务器进程 处理所有docker的请求，管理所有容器
docker镜像仓库Registry  镜像存放的中央仓库，可看作是存放二进制的scm

## Docker安装
Docker的安装非常简单，支持目前所有主流操作系统，从Mac到Windows到各种Linux发行版
具体参考： https://docs.docker.com/installation/

## Docker常见命令

#### 容器相关操作
docker create   # 创建一个容器但是不启动它
docker run      # 创建并启动一个容器
docker stop     # 停止容器运行，发送信号SIGTERM
docker start    # 启动一个停止状态的容器
docker restart  # 重启一个容器
docker rm       # 删除一个容器
docker kill     # 发送信号给容器，默认SIGKILL
docker attach   # 连接到一个正在运行的容器
docker wait     # 阻塞到一个容器，直到容器停止运行

#### 获取相关信息
docker ps       # 显示运行的容器或所有容器
docker inspect  # 深入容器内部获取容器所有信息
docker logs     # 查看容器的日志(stdout/stderr)
docker events   # 得到docker服务器的实时的事件
docker port     # 显示容器的端口映射
docker top      # 显示容器的进程信息
docker diff     # 显示容器文件系统的前后变化

#### 导出容器
docker cp       # 从容器里向外拷贝文件或目录
docker export   # 将容器整个文件系统导出为一个tar包，不带layers、tag等信息

#### 执行
docker exec     # 在容器里执行一个命令，可以执行bash进入交互式

#### 镜像操作
docker images   # 显示所有的镜像列表
docker import   # 从一个tar包创建一个镜像，往往和export结合使用
docker build    # 从一个Dockerfile创建一个镜像
docker commit   # 从一个容器创建一个镜像
docker rmi      # 删除镜像
docker load     # 从一个tar包创建一个镜像，和save配合使用
docker save     # 将一个镜像保存为一个tar包，带layers和tag信息
docker history  # 显示一个镜像生成的历史
docker tag      # 为镜像起一个别名

#### 镜像仓库(registry)操作
docker login    # 登录到一个registry
docker search   # 从registry仓库搜索镜像
docker pull     # 从仓库下载镜像到本地
docker push     # 将一个镜像push到registry仓库中

#### 获取IP地址
docker inspect id | grep IPAddress | cut -d '"' -f 4

#### 获取端口映射
docker inspect -f '{{range $p, $conf := .NetworkSettings.Ports}} {{$p}} -> {{(index $conf 0).HostPort}} {{end}}' id

#### 获取环境变量
docker exec id env

#### 杀掉所有正在运行的容器
docker kill $(docker ps -q)

#### 删除老的容器
docker ps -a | grep 'weeks ago' | awk '{print $1}' | xargs docker rm

#### 删除已经停止的容器
docker rm `docker ps -a -q`

#### 删除所有镜像，小心
docker rmi $(docker images -q)

## Dockerfile

Dockerfile是docker构建的基础，也是docker区别于其他容器的重要特征，正是有了Dockerfile，docker的自动化和可移植姓才成为可能。

不论是开发还是运维，学会编写Dockerfile几乎是必备的，这有助于你理解整个容器的运行。

#### FROM <image name>, 从一个基础镜像构建新的镜像
FROM ubuntu 

#### MAINTAINER <author name>, 维护者信息
MAINTAINER William <wlj@nicescale.com>

#### RUN <command>, 非交互式运行shell命令
RUN apt-get -y install nginx

#### ADD <src> <dst>, 将外部文件拷贝到镜像里,src可以为url
ADD http://nicescale.com/  /data/nicescale.tgz

#### CMD [“param1","param2"]
CMD ["nginx"]

docker运行时执行的命令，如果设置了ENTRYPOINT，则CMD将作为参数

#### EXPOSE <port>, 暴露哪些端口
EXPOSE 443 80

#### ENTRYPOINT [‘executable’, ‘param1’,’param2’]
ENTRYPOINT [ "nginx" ]

#### WORKDIR /path/to/workdir, 设置工作目录
WORKDIR /var/www

#### ENV <key> <value>, 设置环境变量的kv
ENV TEST 1

#### USER <uid>, 设置用户ID
USER nginx

#### VULUME <dir>, 设置volume
VOLUME [‘/data’]

#### Dockerfile最佳实践
A 尽量将一些常用不变的指令放到前面
B CMD和ENTRYPOINT尽量使用json数组方式

```
docker build csphere/nginx:1.7 .
```
## 镜像仓库Registry

镜像从Dockerfile build生成后，需要将镜像推送(push)到镜像仓库。企业内部都需要构建一个docker registry，这个registry可以看作二进制的scm，CI/CD也需要围绕registry进行。

#### 部署registry

```
mkdir /registry
docker run  -p 80:5000  -e STORAGE_PATH=/registry  -v /registry:/registry  registry:0.9.1
```

#### 推送镜像保存到仓库

```
docker tag  csphere/nginx:1.7 192.168.1.2/csphere/nginx:1.7
docker push 192.168.1.2/csphere/nginx:1.7
```

#### 拉取镜像创建容器

```
docker pull 192.168.1.2/csphere/nginx:1.7
docker tag 192.168.1.2/csphere/nginx:1.7 csphere/nginx:1.7
docker run -d csphere/nginx:1.7
```


