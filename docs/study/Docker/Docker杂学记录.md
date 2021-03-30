# 杂记

## 0. 入门教程-网上学习的记录

> [【docker入门】10分钟，快速学会docker](https://www.bilibili.com/video/av58402749)
>
> [Play with Docker](https://labs.play-with-docker.com/)
>
> [Docker 命令详解（run篇）](https://www.cnblogs.com/shijunjie/p/10488603.html)
>
> [docker与虚拟机性能比较](https://blog.csdn.net/cbl709/article/details/43955687)
>
> 安装和配置
>
> [如何在 CentOS 上安装 RPM 软件包](https://www.linuxidc.com/Linux/2019-08/159875.htm)
>
> [centos8 安装 docker](https://blog.csdn.net/yucaifu1989/article/details/103111317)
>
> [RHEL8-安装Docker Problem: package docker-ce-3:19.03.6-3.el7.x86_64 requires containerd.io](https://www.cnblogs.com/leoshi/p/12324827.html)
>
> [docker_清华源国内的选择](http://www.mamicode.com/info-detail-2862950.html)
>
> [安装docker-compose的两种方式](https://blog.csdn.net/LUCKWXF/article/details/96131392)

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

## 0.1 docker入门2-如z何组织一个多容器项目docker-compose

> [docker入门2-如何组织一个多容器项目docker-compose](https://www.bilibili.com/video/av61131351)
>
> [curl 的用法指南](https://www.ruanyifeng.com/blog/2019/09/curl-reference.html)
>
> [docker-compose简介(一)](https://www.cnblogs.com/maduar/p/10777355.html)
>
> [docker-compose安装](https://www.jianshu.com/p/f323aa0416da)
>
> [解决“ImportError: 'module' object has no attribute 'check_specifier'”](https://blog.csdn.net/yanghx9013/article/details/79496527)
>
> [docker-compose.yml配置文件详解](https://blog.csdn.net/Aria_Miazzy/article/details/89326829)
>
> [/usr/lib/python2.7/site-packages/requests/__init__.py:80: RequestsDependencyWarning: urllib3 (1.22) or chardet (2.2.1) doesn't match a supported version! RequestsDependencyWarning)](https://www.cnblogs.com/insane-Mr-Li/p/10914716.html)
>
> [docker-compose up使用自定义的网段的两种方式（从其根源指定）](https://www.cnblogs.com/lemon-le/p/10531449.html)
>
> [docker-compose关于网络名的定义](https://www.jianshu.com/p/d70c61d45364)
>
> [docker network基础](https://www.cnblogs.com/jsonhc/p/7823286.html)

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

下面举例docker-compose使用自己创建的bridge网桥网络

```yaml
version: '3.3'
services:
  http-config-n0-3344:
    container_name: http-config-n0-3344
    build: .
    image: http-config-n0-3344
    restart: always
    hostname: http-config-n0-3344
    networks: 
      ash-http-bridge:
        ipv4_address: 172.20.0.2
networks: 
  ash-http-bridge:
    external:
      name: ash-http-bridge
```

ps：需要实现创建ash-http-bridge网络

```shell
docker network create --subnet=172.20.0.0/16 --gateway=172.20.0.1 ash-http-bridge
```

创建后可以通过`docker network ls`查看创建的网络，发现创建了ash-http-bridge网络

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
>
> [dockerfile运行mysql并初始化数据](https://www.cnblogs.com/UniqueColor/p/11150314.html)

## 3. 运行项目

> [Spring Boot 的项目打包成的 JAR 包，制作成 docker 镜像并运行](https://www.jianshu.com/p/faf7af05a808)
>
> [docker 容器之间互相访问（互联）](https://www.jianshu.com/p/c5e39d6a5307)
>
> [Docker网络管理之docker跨主机通信](https://blog.51cto.com/14157628/2458487?source=dra)
>
> [外部访问docker容器(docker run -p/-P 指令)](https://www.cnblogs.com/williamjie/p/9915019.html)
>
> [在docker下运行mysql](https://www.cnblogs.com/jasonboren/p/11362342.html)
>
> [主机无法访问容器映射的端口：Connection reset by peer](https://www.jianshu.com/p/964b268e1fc8)
>
> [docker容器内存占用过高(例如mysql)](https://www.cnblogs.com/bingogo/p/12144873.html)
>
> [使用docker-compose配置mysql数据库并且初始化用户](https://www.cnblogs.com/mmry/p/8812599.html)
>
> [Docker：MySQL连接慢问题解决](https://blog.csdn.net/bacteriumx/article/details/82792984)
>
> [Docker方式启动tomcat,访问首页出现404错误](https://blog.csdn.net/qq_40891009/article/details/103898876?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)
>
> [Docker中启动Tomcat外部访问报HTTP Status 404 – 未找到](https://blog.csdn.net/mah666/article/details/104055180/)

## 4. 项目运行示例

### 1. cloud config、eureka+security（3个不同公网ip、不同局域网环境的服务器）

每个服务器分别有一个eureka（整合security）和一个config，config只在本地局域网中，eureka向互联网暴露端口。

1. config的启动类使用@EnableConfigServer注解，eureka的启动类用@EnableEurekaServer

```java
//Config的Application启动类
@SpringBootApplication
@EnableConfigServer // 启动Cloud Config服务端服务，获取远程git/gitee的配置
public class HttpConfigN03344Application {

    public static void main(String[] args) {
        SpringApplication.run(HttpConfigN03344Application.class, args);
    }

}

//Eureka的Applicatoin启动类
@SpringBootApplication
@EnableEurekaServer // EnableEurekaSever 服务端的启动类，可以接收别人注册进来~
public class HttpEurekaN17001Application {

    public static void main(String[] args) {
        SpringApplication.run(HttpEurekaN17001Application.class, args);
    }

    @EnableWebSecurity //用到了security，如果不写这个，会发现服务之间没法相互注册，明明有开放端口
    static class WebSecurityConfig extends WebSecurityConfigurerAdapter {
        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http.csrf().ignoringAntMatchers("/eureka/**");
            super.configure(http);
        }
    }
}
```

2. IDEA中编写Config项目的application.yml以及Eureka的application.yml和bootstrap.yml

```yaml
###### Config项目的配置文件 application.yml
server:
  port: 3344 # 服务的端口，只用来docker网络网桥内访问，docker中不用另外配置-p 3344:3344来映射到宿主机，因为我们只要本地的eureka访问网桥bridge下的该Cloud Config服务的3344端口即可

spring:
  application:
    name: http-config-n0-3344 #应用名
    # 连接远程仓库
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/XXXXXX.git # https，不是git，https才能使用下面的账号密码形式，如果用ssh，那么需要用rsa密钥访问。本来我打算用ssh方式，但是尝试了之后失败了，可能我配置下面参数的rsa出错了，但是我按照网络上各种文章以及官方示例，都没整好，就不折腾了
          username: gitee账号
          password: gitee密码

# 通过 config-server可以连接到git，访问其中的资源以及配置~



###### Eureka项目的application.yml
spring:
  application:
    name: http-eureka-n1-7001
    


##### Eureka项目的bootstrap.yml
spring:
  cloud:
    config:
      name: config-eureka-n1
      label: master
      profile: dev
      uri: http://http-config-n0-3344:3344
```

3. gitee上的几个配置文件

   + application.yml

     ```yaml
     # 选择启动的环境 dev test prod
     spring:
       profiles:
         active: dev
     
     ---
     spring:
       profiles: dev
       application:
         name: http-config-dev-n0-3344
     
     ---
     spring:
       profiles: test
       application:
         name: http-config-test-n0-3344
     
     ---
     spring:
       profiles: prod
       application:
         name: http-config-prod-n0-3344
     ```

   + config-eureka-n1.yml（我配置了3个节点，其他2个文件类似，就是n1变成n2和n3，defaultZone把n2、n3对应改成n1、n3以及n1、n2）

     ```yaml
     # eureka节点n1的配置 ip: 你的第一个服务器公网ip
     # 选择启动的环境 dev test prod
     spring:
       profiles:
         active: dev
     
     ---
     # 服务启动项
     server:
       port: 7001
     
     #spring配置
     spring:
       profiles: dev
       application:
         name: http-eureka-dev-n1-7001
       security: # 使得Eureka需要账号密码才能访问
         user:
           name: 账号
           password: 密码
           roles: SUPERUSER
     
     #eureka配置
     eureka:
       instance:
         hostname: http-eureka-n1-7001 
         appname: http-eureka-7001
         instance-id: n1-服务器1的公网ip
         prefer-ip-address: true
         ip-address: 服务器1的公网ip
       client: 
         service-url: 
           #单机 defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
           #设置与Eureka Server交互的地址查询服务和注册服务都需要依赖这个地址（单机）。
           defaultZone: http://账号:密码@服务器2公网ip:7001/eureka/,http://账号:密码@服务器3公网ip:7001/eureka/
     
     ---
     # 服务启动项
     server:
       port: 7001
     
     #spring配置
     spring:
       profiles: test
       application:
         name: http-eureka-test-n1-7001
       security: # 使得Eureka需要账号密码才能访问
         user:
           name: 账号
           password: 密码
           roles: SUPERUSER
     
     #eureka配置
     eureka:
       instance:
         hostname: http-eureka-n1-7001 
         appname: http-eureka-7001
         instance-id: n1-服务器1公网ip
         prefer-ip-address: true
         ip-address: 服务器1的公网ip
       client: 
         service-url: 
           #单机 defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
           #设置与Eureka Server交互的地址查询服务和注册服务都需要依赖这个地址（单机）。
           defaultZone: http://账号:密码@服务器2公网ip:7001/eureka/,http://账号:密码@服务器3公网ip:7001/eureka/
     
     ---
     # 服务启动项
     server:
       port: 7001
     
     #spring配置
     spring:
       profiles: prod
       application:
         name: http-eureka-prod-n1-7001
       security: # 使得Eureka需要账号密码才能访问
         user:
           name: 账号
           password: 密码
           roles: SUPERUSER
     
     #eureka配置
     eureka:
       instance:
         hostname: http-eureka-n1-7001 
         appname: http-eureka-7001
         instance-id: n1-服务器1的公网ip
         prefer-ip-address: true
         ip-address: 服务器1的公网ip
       client: 
         service-url: 
           #单机 defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
           #设置与Eureka Server交互的地址查询服务和注册服务都需要依赖这个地址（单机）。
           defaultZone: http://账号:密码@服务器2公网ip:7001/eureka/,http://账号:密码@服务器3公网ip:7001/eureka/
     ```
   
4. pom依赖不贴了，就贴其中一个要点，不配置会发现maven执行package操作后生成的jar文件很小，docker上运行不起来。eureka的pom还需加security的依赖，自己查

   ```xml
   <build>
       <plugins>
           <plugin>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-maven-plugin</artifactId>
               <version>${springboot.version}</version>
               <configuration>
                   <!-- 指定该Main Class为全局的唯一入口 -->
                   <mainClass>com.ash.springcloud.HttpConfigN03344Application</mainClass>
                   <layout>ZIP</layout>
               </configuration>
               <executions>
                   <execution>
                       <goals>
                           <goal>repackage</goal><!--可以把依赖的包都打包到生成的Jar包中-->
                       </goals>
                   </execution>
               </executions>
           </plugin>
       </plugins>
   </build>
   ```

   5. 重点来了，Dockerfile和docker-compose.yml的编写
      
      1. Config项目的Dockerfile
      
         ```dockerfile
         FROM java:8-alpine
         ADD http-config-n0-3344-0.0.1-SNAPSHOT.jar http-config-n0-3344-0.0.1-SNAPSHOT.jar
         EXPOSE 3344
         ENTRYPOINT ["java","-jar","/http-config-n0-3344-0.0.1-SNAPSHOT.jar"]
         ```
      
      2. Config项目的docker-compose.yml
      
         ```yaml
         version: '3.3'
         services:
           http-config-n0-3344:
             container_name: http-config-n0-3344
             build: .
             image: http-config-n0-3344
             restart: always
             hostname: http-config-n0-3344
             networks: 
               ash-http-bridge:
                 ipv4_address: 172.20.0.2
         networks: 
           ash-http-bridge:
             external:
               name: ash-http-bridge
         ```
      
      3. Eureka项目的Dockerfile
      
         ```dockerfile
         FROM java:8-alpine
         ADD http-eureka-n1-7001-0.0.1-SNAPSHOT.jar http-eureka-n1-7001-0.0.1-SNAPSHOT.jar
         EXPOSE 7001
         ENTRYPOINT ["java","-jar","/http-eureka-n1-7001-0.0.1-SNAPSHOT.jar"]
         ```
      
      4. Eureka项目的docker-compose.yml
      
         ```yaml
         version: '3.3'
         services:
           http-eureka-n1-7001:
             container_name: http-eureka-n1-7001
             build: .
             image: http-eureka-n1-7001
             restart: always
             hostname: http-eureka-n1-7001
             ports:
               - 7001:7001 # 对宿主机提供端口，这样其他Eureka服务器才能访问到这个服务器节点的docker容器的7001端口
             networks: 
               ash-http-bridge:
                 ipv4_address: 172.20.0.3 # 我自己创建的网桥bridge网络，下面会说
         networks: 
           ash-http-bridge:
             external:
               name: ash-http-bridge
         ```
      
   6. 把项目生成的Config和Eureka的jar包放到服务器上
   
      ```shell
      # linux上自己找文件夹放，这个文件夹内保证只有jar包、Dockerfile和docker-compose.yml
      # 我创建文件夹后，执行ls获取目录下文件如下，另一个eureka同理
      docker-compose.yml  Dockerfile  http-config-n0-3344-0.0.1-SNAPSHOT.jar
      ```
   
   7. 安装docker不讲了，这个讲安装docker-compose（我的环境是CentOS7，其他环境不确定是否相同）
   
      > [RHEL8-安装Docker Problem: package docker-ce-3:19.03.6-3.el7.x86_64 requires containerd.io](https://www.cnblogs.com/leoshi/p/12324827.html)
      >
      > [腾讯云服务器Centos7.6中docker-compose工具安装](https://blog.csdn.net/kellerxq/article/details/103326664)
      >
      > [centos 7.4 安装docker 19.03.6 版本。附带离线安装包](https://www.cnblogs.com/wangzy-tongq/p/12361880.html)
      >
      > [基于docker19.03.6的服务虚拟化的坑（系统内核版本）](https://blog.csdn.net/HK_poguan/article/details/104657849)
      >
      > [安装docker-compose的两种方式](https://blog.csdn.net/LUCKWXF/article/details/96131392)
   
      ```shell
      # 推荐根据上面的文章安装，因为我自己安装也出过问题，下面直接贴上面文章的安装方式
      # 首先我说下我的环境 CentOS7，自带python2.7.5,环境不同的情况不保证安装方式相同，自己解决
      yum -y install epel-release
      yum -y install python-pip
      pip install --upgrade pip
      pip install docker-compose 
      # 如果安装docker-compose报错了，安装不了，就执行以下几个步骤后再重新安装
      pip install cffi==1.6.0
      yum install python-devel
      pip install --ignore-installed requests
      #上面几个步骤我也没细追究，但是执行完后，我就能成功安装docker-compose了
      pip install docker-compose
      docker-compose --version #查看安装的版本，我是docker-compose version 1.25.4, build unknown
      ```
   
   8. 自定义网桥
   
      ```shell
      # 先通过ifconfig查看docker0网卡的ip，一般都是172.17.0.1或者172.18.0.1等(172.1X.0.1)左右，我们需要保证自定义的网桥和当前所有的其他网络网段不冲突，包括eth0->linux服务器自带的网卡
      # 这里我172.20.0.1不存在于ifconfig里面已有的网卡信息中，所以我使用这个来演示
      docker network ls # 查看当前docker配置的网络，一般会默认有bridge、host和none
      docker network inspect bridge #查看bridge的详细配置
      # 发现我们要配置的设置项有Subnet和Gateway
      docker network create ash-http-bridge --subnet=172.20.0.0/16 --gateway=172.20.0.1 # 创建名为ash-http-bridge的网络
      ifconfig # 发现出现了新的网络设备(br-XXXX),inet为172.20.0.1,netmask为255.255.0.0符合我们刚才的配置
      docker network ls # 看到我们新建的ash-http-bridge
      docker network inspect ash-http-bridge # 查看ash-http-bridge具体配置，确认Subnet和Gateway参数设置正确
      ```
   
   9. 因我项目还没完全确立下来，所以前面的docker-compose.yml分2个写，不然一般都是整合在一个文件里面写的。下面分别到两个docker-compose.yml所在的文件夹，启动项目(因为eureka的配置要从config那里读取，所以需要先启动config)
   
      ```shell
      # 先到config的jar包所在目录，确保只有jar、Dockerfile和docker-compose.yml
      docker-compose up -d --build # -d后台运行，--build临时通过Dockerfile构建image后使用该image来构建、运行容器
      
      # 再到eureka的jar包所在目录进行相同操作。之后只要开放服务器的7001端口。就可以通过ip:7001访问到eureka的界面了
      ```
   
   
   10. 不同网络环境（局域网）的几台公网IP服务器，必须开放eureka的端口给互联网访问才能互相注册。

## 5. Docker网络

> [跨宿主机Docker网络访问](https://blog.csdn.net/chenzhanhai/article/details/100694087)
>
> [docker容器间跨主机通信](https://www.cnblogs.com/lyhero11/p/9967328.html)
>
> [容器云技术选择之kubernetes和swarm对比](https://www.cnblogs.com/i6first/p/9399213.html)
>
> [Docker网络解决方案 - Weave部署记录](https://www.cnblogs.com/kevingrace/p/6859173.html)
>
> [Docker Swarm - Overlay 网络长连接问题](https://www.jianshu.com/p/f05294c0a456)
>
> [Docker Swarm - 网络管理](https://www.jianshu.com/p/60bccbdb6af9)
>
> [centos7 安装 openvswitch](https://blog.csdn.net/xinxing__8185/article/details/51900444)
>
> [Docker的4种网络模式](https://www.cnblogs.com/gispathfinder/p/5871043.html)
>
> [docker 网络模式 和 端口映射](https://www.cnblogs.com/chenpython123/p/10823879.html)

## 6. Podman

> [Docker 大势已去，Podman 万岁](https://blog.csdn.net/alex_yangchuansheng/article/details/102618128)

## 7. docker和虚拟机

> [docker容器与虚拟机有什么区别？](https://www.zhihu.com/question/48174633)	<=	以下图文出至该文章
>
> 说了这么多Docker的优势，大家也没有必要完全否定**虚拟机**技术，因为两者有不同的使用场景。
>
> + **虚拟机**更擅长于彻底<u>隔离整个运行环境</u>。例如，云服务提供商通常采用虚拟机技术隔离不同的用户。
> + **Docker**通常用于<u>隔离不同的应用</u>，例如**前端**，**后端**以及**数据库**。
>
> 因此，我们需要根据不同的应用场景和需求采用不同的方式使用Docker技术或使用服务器虚拟化技术。例如一个典型的Docker应用场景是当主机上的Docker实例属于单一用户的情况下，在保证安全的同时可以充分发挥Docker的技术优势。
>
> **虚拟机是为提供系统环境而生的，容器是为提供应用环境而生的，各有各的标地。**

![img](https://pic1.zhimg.com/80/v2-ee27d299f5e38ed460218ac087518bba_1440w.jpg?source=1940ef5c)

![img](https://pic2.zhimg.com/80/v2-0d67e01d75d19e227fb44104eca28f43_1440w.jpg?source=1940ef5c)

![img](https://pic1.zhimg.com/80/v2-0063a244295c22dc45ed92e412dbfc15_1440w.jpg?source=1940ef5c)

