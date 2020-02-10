# 杂记

## 0. 入门教程-网上学习的记录

> [【docker入门】10分钟，快速学会docker](https://www.bilibili.com/video/av58402749)
>
> [Play with Docker](https://labs.play-with-docker.com/)

—————————仓库

​				 		pull |      | push

Dockerfile--build--  镜像  --  load -- save -- tar文件

​				 		run |      | commit

—————————容器

1. 镜像
   + 介绍：类似用虚拟机时，创建虚拟机前需要下载的系统镜像文件，比如iso文件、img文件(ps:我装office就用到了img格式的)等这样一些镜像文件
2. 容器
   + 介绍：可以类比==正在运行==的一个虚拟机
3. tar文件
   + 介绍：类似于vm使用时的vmdk文件，它可以将一个镜像直接保存成一个tar文件传输给别人，别人通过load指令，可以将其重新加载成一个镜像，然后再通过run指令就起来一个正在运行的虚拟机了(容器)
4. Dockerfile
   + 介绍：相当一个配置文件，通过写"如何构建"的步骤，来指定一个镜像是如何构建的，然后通过`docker build`指令可以将dockerfile构建成一个镜像
5. 仓库
   + 介绍：远程仓库保存了很多镜像，包括一些共有的第三方做好的镜像比如ubuntu镜像、nginx镜像、MySQL镜像、tomcat镜像等等。可以通过`docker pull`指令下载这些镜像，也可以通过`docker push`上传自己的镜像

```shell
## 从远程服务器拉取(下载)取镜像
docker pull nginx # 不指定版本相当于 docker pull nginx:latest 通常约定latest为最新版本

## 查看本地拥有的镜像
docker images

## 将镜像运行成一个真正在运行的容器(虚拟机)
docker run -d -p 81:80 nginx #(镜像名称或用nginx镜像的IMAGE ID)，-d后台运行而不阻塞shell指令窗口,-p指定内外端口映射，外部81端口映射内部80端口

## 查看正在运行的容器
docker ps #-a 查看所有容器，包括没运行的

## 修改nginx容器
docker exec -it nginx容器id bash #(只要开头几位就好，能区分其他即可),bash是通过bash方式进入容器，能看到输入指令的开头变成root@容器ID:/# 光标位置

## 退出docker容器
exit

## 强制删除容器
docker rm -f 容器ID #(同样只要开头几位能识别就好了)

## 将本地的容器提提提交到本地形成一个指定名称的镜像，该镜像保留容器中所作的修改
docker commit 容器ID m1 #容器ID同样只要开头几位；m1指定镜像的名字，例如这里取名m1

## 编写Dockerfile文件
vim Dockerfile

## 编写一个示例的文件
vim index.html #随便写写就好了

## 通过docker build使用Dockerfile去构建一个定制的镜像
docker build -t m2 . # .点点表示指定当前目录下的Dockerfile文件去构建

## 将镜像保存到tar文件
docker save m2 > 1.tar # docker save [镜像名称] > tar文件名.tar

## 删除镜像
docker rmi m2 # 可以通过名称也可以通过IMAGE ID删除，如果镜像被运行成容器就删除不了

## 读取tar文件恢复一份镜像
docker load < 1.tar #这里运行后，发现images又重新出现了之前Dockerfile创建的m2镜像

## push操作需要自己去dockerhub或者其他官方仓库注册
## docker run -d -p 88:80 --name ，其中--name 指定容器运行后的名字NAMES，默认随机单词拼写的名字
## docker run -d -p 88:80 --name -v ，-v映射文件，比如可以把大哥前目录映射到内部的/usr/share/nginx/html
docker run -d -p 88:80 --name mynginx -v 'pwd':/usr/share/nginx/html nginx:latest# 这样就可以将一些静态文件放在外面，外面修改文件(因为是映射的)，里面的文件就会跟着变化
## 文件的映射也可用于一些数据的保存，比如MySQL的data目录可以映射到外面，防止数据丢失
## 最后接的参数就是镜像的名字，后面的版本如果知道具体的版本号就最好不要省略，省略就是latest最新版，而镜像如果一直在全新的构建的话，latest会不断的更新，如果有一个已知正在使用的稳定版本，最好指定这个版本
```

Dockerfile部分

```dockerfile
# 通过FROM指定docker构建的基础镜像，是基于nginx这个镜像
FROM nginx
# 将当前目录下所有文件拷贝到/usr/share/nginx/html/这个文件夹下
ADD ./ /usr/share/nginx/html/
```

## 1. Dockerfile

> [你必须知道的Dockerfile](https://www.cnblogs.com/edisonchou/p/dockerfile_inside_introduction.html)
>
> [Dockerfile文件详解](https://www.cnblogs.com/panwenbin-logs/p/8007348.html)
>
> [为什么docker上自己创建的mysql镜像特别大](https://www.imooc.com/wenda/detail/425720)

## 2. 定制镜像

> [Docker 定制镜像](https://www.jianshu.com/p/ff65a4db85ca)