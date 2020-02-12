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
docker run -d -p 81:80 nginx #(镜像名称或用nginx镜像的IMAGE ID)，-d后台运行而不阻塞shell指令窗口,-p指定内外端口映射，外部81端口映射内部80端口;如果直接run某个镜像，而本地没又该镜像，则会自动尝试从远程服务器pull下载然后继续run

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

## 0.1 docker入门2-如何组织一个多容器项目docker-compose

> [docker入门2-如何组织一个多容器项目docker-compose](https://www.bilibili.com/video/av61131351)
>
> [curl 的用法指南](https://www.ruanyifeng.com/blog/2019/09/curl-reference.html)

### 概述

​	宿主机安装docker后，生成一张docker网卡，docker网卡通过NAT方式为每个容器分配IP。(例如docker网卡IP为172.27.0.1，子网掩码255.255.255.0，给3个容器分配IP为172.27.0.2~172.27.0.4)。容器之间同一网段，可以直接通信，而宿主机和容器之间的通信则需要经过docker网卡

```shell
## 首先先创建一个自己的nginx容器
docker run -d -p 80:80 --name mynginx nginx

## 进入该容器
docker exec -it mynginx bash # 可以用容器ID也可以容器名字

## 没有ifconfig情况下，可以查看hosts文件得知IP信息
cat /etc/hosts # 可以看到已经默认为改容器ID配置了对应的IP

## 退出容器
exit

## -d后台运行 -it阻塞运行,alpine是最小的linux操作系统，大概5M左右
docker run -dit alpine

## 进入刚才运行的alpine容器中,通过sh进入
docker exec -it 容器ID sh 

## 通过apk add 安装curl
apk add curl

## 通过curl获取另一个容器的数据
curl 172.17.0.2 #这里之前进去的第一个容器mynginx的ip通过hosts文件查看就是这个

## 退出alpine，下面介绍实际生产中容器间的通信策略
exit

## 删除刚才创建的示范用的alpine容器
docker rm -f alpine容器的ID

## --link 将外部的mynginx映射到 当前要访问的容器alpine内部的myng，即在该alpine安装curl后，可以直接curl myng 代替直接写curl mynginx容器的IP，其原理其实就是自动帮我们在hosts文件按添加了IP和域名的映射关系
docker run -dit --link mynginx:myng alpine

## 通过sh方式进入后来新建的alpine容器中
docker exec -it alpine的容器ID sh

## 安装curl
apk add curl

## 访问之前运行的mynginx容器
curl ng #由于前面运行mynginx容器使用--link参数，自动帮忙配置了hosts的域名IP映射

## 查看hosts文件，验证--link参数自动配置的域名IP映射
cat /etc/hosts # 这里是172.17.0.2      myng ae4a8dd9aba8 mynginx，如果--link myng:myng则为172.17.0.2      myng ae4a8dd9aba8。

## 退出当前alpine容器
exit

## 如果使用--link关联每个容器之间的通讯，那么如果nginx->php->mysql，那么就必须按顺序先创建mysql容器，创建php容器且使用--link映射mysql容器，创建nginx且使用--link映射php，这样才能达到原本预定的访问效果。这样迁移烦琐、维护起来也麻烦

## 下面使用docker-compose，play with docker默认安装了，我们自行安装就完事了.下面新开一个INSTANCE实例来演示
docker-compose

## 需要创建2个目录，这里先创建conf目录，存放配置信息文件
mkdir conf

## 创建html目录存放html和php文件
mkdir html

## 进入html目录，编写index.html
cd html/
vim index.html # 随便写一点东西就好了，演示

## 编写test.php检测PHP是否运行成功
vim test.php

## 编写mysql.php用来进行mysql的数据库访问
vim mysql.php

## 返回conf目录，新建nginx配置文件
cd ../conf/
vim nginx.conf

## 回到根目录，编写docker-compose.yml配置文件
cd 
vim docker-compose.yml

## 启动 -d表示后台启动
docker-compose up -d

## 在自动拉取镜像启动完成后，查看运行的容器的NAMES等
docker ps
# 发现分别是root_php_1;root_mysql_1;root_nginx_1;其中root指docker-compose.yml在root目录下，php是刚才配置的service的名字，1表示第一个实例，因为docker-compose可以指定运行多个，就会有1234等编号

##点击访问80端口。发现访问到根目录显示index.html文字，即访问到宿主机的/root/html/index.html文件；访问/php.test确认php能访问成功

## 下面讲解过程，如果访问根目录，即index.html文件，会到nginx读取配置，其已经映射到外部配置，首先读取/的配置条件，然后匹配搭配/usr/share/nginx/html下读取，而其又映射到外部的/root/html目录下，所以最后等于读取到了/root/html目录下的文件
## 如果访问test.php,也是到nginx访问(外部80映射到内部80)，然后会匹配正则 ~ 以php$结尾的，转发到php:9000号端口，目录是/var/www/html,这时候到达php容器(因为php:9000会解析到php容器，通过9000端口过来)，而php容器内的目录又映射到外部的/root/html下，和前面读取html一样的位置，等于最后还是访问到了外部宿主机的目录
## mysql.php同样先经过nginx，解析php$这一条，最后同样到达外部宿主机的/root/html，和test.php一样，读取到/root/html/mysql.php，该文件中指定数据库的主机host就是mysql，docker-compose.yml中已经配置了，就会根据这个域名去解析，刚好解析到有个叫mysql的容器，到它的3306端口，密码123456，登录成功后，将这条登录成功数据原路返回回去
```

下面是test.php文件

```php
<?php
    phpinfo();
```

下面是mysql.php文件

```php
<?php
$dbhost = "mysql";
$dbuser = "root";
$dbpass = "123456";
 
// 创建连接
$conn = mysqli_connect($dbhost, $dbuser, $dbpass);
 
// 检测连接
if (! $conn) {
    die("Could not connect: " . mysqli_error());
} 
echo "mysql connected!!";
mysqli_close($conn);
?>
```

下面是nginx.conf文件

```none
worker_processes 1;

events {
	worker_connections 1024;
}
http {
	include			mine.types;
	default_type	application/octet-stream;
	
	sendfile		on;
	
	keepalive_timeout	65;
	server {
		listen 80;
		server_name		localhost;
		
		location /	{
			root	/ust/share/nginx/html;
			index	index.html	index.htm;
		}
		
		error_page	500	502	503	504	/50x.html
		location = /50x.html {
			root /usr/share/nginx/html;
		}
		
		location ~ \.php$ {
			fastcgi_pass	php:9000;
			fastcgi_index	index.php;
			fastcgi_param	SCRIPT_FILENAME	/var/www/html/$fastcgi_script_name;
			include			fastcgi_params;
		}
	}
}
```

下面是docker-compose.yml配置文件

```yaml
# version 指定版本
version: "3"
# services 指定服务
services:
	nginx:
		image: nginx:alpine # 配置使用该镜像
		ports: # 配置端口，相当于-p参数,外部80映射到内部80端口
		- 80:80
		volumes: # 目录，相当于-v参数
		- /root/html:/usr/share/nginx/html #外部html目录映射到内部目录，修改外部该目录文件里面的映射目录也会被对应的修改
		- /root/conf/nginx.conf:/etc/nginx/nginx.conf
	php: #image不能指定官方的php因为官方的php镜像缺少一些等下要用的扩展，可以去dockerhub搜索，找到php-fpm，这里使用 devilbox/docker-php-fpm,这里随便从Tag里挑一个版本
		image: devilbox/php-fpm:5.2-work-0.89
		volumes:
		- /root/html:/var/www/html # 这里映射了2此/root/html,nginx映射的那个如果访问的不是php文件，就会到/usr/share/nginx/html找文件，其映射出来就是/root/html，那如果外部存放一些非PHP文件就能访问到；如果访问PHP文件，就会转发到php容器中，到/var/www/html找文件，同样映射到外部/root/html找php文件
	mysql: # php文件的dbhost必须是mysql，因为已经映射过来了，不能写localhost
		image: mysql:5.6
		environment: # 设置环境变量：MYSQL_ROOT_PASSWORD,这是mysql容器启动必须配置的参数
		- MYSQL_ROOT_PASSWORD=123456 # 因为mysql容器里面默认的密码123456，所以这里这么配置	
```



## 1. Dockerfile

> [你必须知道的Dockerfile](https://www.cnblogs.com/edisonchou/p/dockerfile_inside_introduction.html)
>
> [Dockerfile文件详解](https://www.cnblogs.com/panwenbin-logs/p/8007348.html)
>
> [为什么docker上自己创建的mysql镜像特别大](https://www.imooc.com/wenda/detail/425720)
>
> [【Docker】Dockerfile用法全解析](https://www.bilibili.com/video/av85895204)

Dockerfile常用的5个指令

1. FROM：指定运行的镜像(只有该参数是必须的)
2. WORKDIR：指定接下去的shell语句运行的目录(目录不存在会自动创建)
3. COPY(ADD类似，下面讲区别)：把宿主机的文件拷贝到镜像中去
4. RUN：就是运行shell语句(构建时运行)
5. CMD(ENTRYPOINT类似，下面讲区别)：指定整个镜像运行起来时执行的脚本(容器真正运行时执行)，并且这个脚本运行完后，整个容器的生命周期也就结束了，所以通常可以指定一些阻塞式的脚本

+ EXPOSE：指定容器向外暴露的端口
+ VOLUME：指定映射文件，比如指定 /a/b，那就是把这个目录映射到宿主机的一个目录下(一般都是映射到匿名卷) ；docker run中的-p和-v是分别指定映射到外部的端口和目录

+ ENV：指定参数的几种方式之一，能指定当前容器的环境变量，比如ENV A=10（或写成A 10，中间有个空格），那么CMD可以改成CMD echo $A ;也可以通过docker run -e 参数来指定环境变量
+ ARG：本身就是参数的意思，与ENV不同的是，ENV在构建和运行时都生效，是系统环境变量；而ARG是构建参数，只有在构建时生效，前面说过CMD是运行时才生效,如果ARG B=10那么CMD echo $B是无法打印出10的，找不到参数，打印出空行。可以通过ARG B=10而ENV A $B而CMD echo $A把ARG指定的参数B打印出来。**构建指的是docker build**。因为ARG要被用到终究还是得靠CMD，而ARG指定的值可以当作默认值，那么使用 docker build -t test（这个是镜像名） --build-arg B=12则会将里面的B设置为12，而不指定B时，内部的参数则为默认的10.
+ LABEL：指定一些元数据信息，key="value"形式。比如`LABEL k="v" k1="v1"`。元素据其实没什么实质性作用，就起到标识性作用。
+ ONBUILD：当前镜像构建时不会执行，基于当前镜像的镜像构建的时候才会执行。ONBUILD后面接的参数可以是Dockerfile其他的任意参数，比如`ONBUILD ENV C=100`下一行`CMD echo $C`,构建当前镜像后直接运行，发现打印空行，因为当前是没有C这个环境变量的。这时候如果在别的目录下创建一个Dockerfile，内容只有一行`FROM test`，也就是以上一个镜像来构建镜像，然后执行build和run，发现打印环境变量值为100。
+ STOPSIGNAL：指定当前的容器以什么信号停止，很少使用，自己去了解
+ HEALTHCHECK：检查容器健康状态的配置，也是很少使用
+ SHELL：指定基于哪种shell，一般Linux默认是 /bin/sh，也是一般不用写这项

```dockerfile
FROM alpine
WORKDIR /app
COPY src/ /app # 复制宿主机src/目录下所有内容到容器的/app目录下
RUN echo 321 >> 1.txt # 此时目录即WORKDIR指定的 /app目录
CMD tail -f 1.txt
```

```shell
vim Dockerfile # 编写以上内容

## 构建镜像
docker build -t test .

## 运行创建的test镜像，CMD指令为监听文件的后面几行，阻塞
docker run test

## 修改CMD的指令为 cat 1.txt后，重新运行docker，发现输出文字后，直接运行完毕，进入stop状态，这时候要是执行docker ps会发现没有运行中的容器
vim Dockerfile
docker build -t test .
docker run test
```

+ COPY和ADD指令类似，COPY一般第一个参数源地址指的是宿主机的目录，而ADD的源地址不光可以是文件系统，还可以是url，一般推荐使用COPY而不用ADD。*够用就好原则，如果没有用到网络资源就直接使用COPY*

+ CMD和ENTRYPOINT类似，两者如果未指定则都是继承自父镜像，如果祖辈也没有指定CMD或者ENTRYPOINT，则镜像无法构建。两者如果同时适配，搭配不同时效果可能也不同，详情走官方文档。口诀：**如果ENTRYPOINT非json则以ENTRYPOINT为准，如果ENTRYPOINT和CMD都是JSON则ENTRYPOINT+CMD拼接成shell**，其他的情况都不会遇到，没必要记住

## 2. 定制镜像

> [Docker 定制镜像](https://www.jianshu.com/p/ff65a4db85ca)

## 3. 运行项目

> [Spring Boot 的项目打包成的 JAR 包，制作成 docker 镜像并运行](https://www.jianshu.com/p/faf7af05a808)
>
> [docker 容器之间互相访问（互联）](https://www.jianshu.com/p/c5e39d6a5307)
>
> [Docker网络管理之docker跨主机通信](https://blog.51cto.com/14157628/2458487?source=dra)