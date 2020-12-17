# Shell学习

> 结合网课、课内教材、网文学习
>
> [【IT】Shell编程从入门到放弃（2016）【全20集】]( https://www.bilibili.com/video/av8104450?p=12 )

## 常用指令

| 指令/参数 | 使用/作用                                                    |
| ---- | ------------------------------------------------------------ |
| $?        | 保存前一个命令的返回码                                       |
| $*        | 传给脚本或函数的参数组成的单个字符串，即除脚本名称后从第一个参数开始的字符串，每个参数以$IFS分割(一般内部域分隔符$IFS为一个空格) |
| $#        | 传给脚本/函数的参数的个数 |



颜色字体

-e "\033[3*m内容\033[0m" 也可以是 4*m;后面如果是1m就后面的颜色都变化



| 指令/参数 | 使用/作用                            |
| --------- | ------------------------------------ |
| -d file   | 返回真的条件，file存在并且是一个目录 |
| -e file   | File 存在                            |
| -f file   | file存在并且是一个普通文件           |



select语句，配合case语句使用

 

sed ‘/^/&123’在每行行首插入123

 

grep -E “正则表达式”文件 （不加-E不会有输出）

 

awk '{print $1}' 打印第一列

awk -F: 'print$1' 以“：”冒号为分隔符，打印第一列

 

find 目录 -maxdepth 最深遍历层数 -name "文件名" 或者 -type 类型（比如f，就是普通文件）-mtime +30（三十天以前的，要是要今天的，可以是 -1）

 

-exec 指令 {} \ ;   大括号就是等于前面执行的指令获取的结果

|xargs rm -f {} \;     看情况2个都可以不用 \;  



下面是运维的增量备份、全备份使用

tar -g 文件，全备份

echo \`date +%d\` 日； \`date +%u\` 周几；



运维根据恶意访问（22端口多次输入密码错误，强封IP）



运维根据脚本同步不同服务器的文件rsync



Shell批量监控服务发送邮件报警

---

# web开发自己用到的常用Linux指令

## 1. Linux远程服务器文件的上传、下载

> [linux系统下的rz、sz详解](https://blog.csdn.net/mynamepg/article/details/81118580)

1. sz:将选定的文件发送（send）到本地机器
2. rz:运行该命令会弹出一个文件选择窗口，从本地选择文件上传到服务器(receive)

## 2. linux下的文本去重方法

> [linux下的几种文本去重方法](https://blog.csdn.net/qq_40809549/article/details/82591302)

1. 基础使用

   awk命令去重输出：awk '!x[$0]++' filename

   应用扩展1：cat Afile Bfile|awk '!x[$0]++' >Cfile

   依次输出A、B两个文件内容，去掉B中与A重复的行，输出到C；多应用于日志拼接。

   灵活扩展2：cat Afile|awk '!x[$0]++'

   也可以写作：awk '!x[$0]++' Afile

   去掉重复的行，输出A文件

2. 复制.bash_history到本地(配合sz指令)

   先到当前用户根目录，使用ls -a确保有.bash_history文件

   + 使用去重指令把去重后的文件复制出来。

     cat .bash_history|awk '!x[$0]++' > historyXXXX.txt

   + 使用sz指令把文件传到本地

     sz historyXXXX.txt

## 3. MongoDB的启动和停止

> [Centos环境下安装mongoDB](https://www.cnblogs.com/layezi/p/7290082.html)

1. MongoDB启动

   sudo systemctl start mongod.service

2. MongoDB停止

   sudo systemctl stop mongod.service

3. MongoDB重启

   sudo systemctl restart mongod.service

## 4. 查看进程指令

> [Linux ps命令](https://www.runoob.com/linux/linux-comm-ps.html)

1. 查看指定的程序是否运行

   ps -ef|grep '程序名'

   + 查看mongod是否启动

     ps -ef|grep 'mongod'

   + 查看是否有java程序运行中（比如springboot项目）

     ps -ef|grep java

## 5. 查看系统信息

> [Linux top命令](https://www.runoob.com/linux/linux-comm-top.html)
>
> [linux 下 取进程占用内存(MEM)最高的前10个进程](https://www.cnblogs.com/liuzhengliang/p/5343988.html)
>
> [linux系统查看系统内存和硬盘大小](https://www.cnblogs.com/yangzailu/p/10939126.html)

1. top指令(类似windows的任务管理器)

2. 查看总的内存使用情况

   ```shell
   free -m | sed -n '2p' | awk '{print""($3/$2)*100"%"}'
   ```

3. linux 下 取进程占用 cpu 最高的前10个进程

   ```shell
   ps aux|head -1;ps aux|grep -v PID|sort -rn -k +3|head
   ```

4. linux 下 取进程占用内存(MEM)最高的前10个进程

   ```shell
   ps aux|head -1;ps aux|grep -v PID|sort -rn -k +4|head
   ```

5. 查看系统运行内存

   ```shell
   free -m
   # （Gb查看） 
   free -g
   ```

6. 查看硬盘大小

   ```shell
   df -hl 
   ```

## 6. ICMP检测服务器是否可连通

> [Linux ping命令](https://www.runoob.com/linux/linux-comm-ping.html)

1. ping指令

## 7. 简单文件操作

1. 简单写一句话到文件中

   echo '我是一句废话' > test.txt

2. 删除文件(-r 递归，通常用于删除文件夹及子目录;-f强制删除，不询问)

   rm -rf xxxx 

3. 查看文件

   less 文件名 （空格下一页，B上一页）

4. 监听文件(比如查看实时变化的日志文件)

   tail -f 文件名

5. 修改文件权限（4r读+2w写+1x执行=7） 

   + rwx 读写执行，777三个分别表示文件拥有者用户权限、用户组权限、其他人权限

   + 数字设定法

     chmod 777 文件名or目录名

   + 字符设定法

     chmod [who] [+|-|=] [mode] 文件名

     > + who表示操作对象
     >
     >   u表示用户；g表示用户组；o表示其他用户；a表示所有用户
     >
     > + 操作符号含义
     >
     >   +表示添加权限 ；-表示取消权限；=表示删除其他所有权限后重新赋予权限
     >
     > + mode表示执行的权限
     >
     >   r只读；w只写；x可执行

6. 查看当前目录文件

   + 不包括隐藏文件

     ls -l

   + 包括隐藏文件

     ls -al

7. 创建目录

   mkdir 目录名

8. 打印当前目录

   pwd

9. 查找文件

   1. find <指定目录> <指定条件> <指定动作>

      + 从根目录起最大深度查找7层，查找名字带有‘.txt’的文件

        find / -maxdepth 7 -name '*.txt'

      + 从根目录起最大 深度查找2层，查找类型为‘f’(普通文件类型)的文件

        find / -maxdepth 2 -type f

      + 在logs目录中查找更改时间在5日以前的文件并删除它们

        find logs -type f -mtime +5 -exec rm { } \ 

      + 在/etc目录中查找文件名以host开头的文件，并将查找到的文件输出到标准输出

        find / etc -name "host*" -print

## 8. 查看端口占用

> [Linux中netstat和ps命令的使用](https://blog.csdn.net/u014303647/article/details/82530495)
>
> [Linux 查看服务器开放的端口号](https://www.cnblogs.com/wanghuaijun/p/8971152.html)

1. Linux查看端口号占用命令

   netstat -pan | grep 12345

   `lsof -i :8080`

2. 通过进程ID查找程序

   ps -aux | grep 12345
   
3. 通过关键字查找进程

   ps -ef | grep java

   netstat -unltp|grep fdfs

4. 使用nmap工具

   + 查看本机开放的端口

     ```shell
     nmap 127.0.0.1
     ```

     

## 9. 强制结束某进程

1. kill -9 进程号

## 10. 后台运行程序

> [nohup和&后台运行，进程查看及终止](https://blog.csdn.net/themanofcoding/article/details/81948094)

1. nohup

   + 这里举例后台运行springboot程序(后台运行，并指定日志输出位置)

     nohup java -jar SpringBoot的jar包文件.jar > /xxxx/yyy/logs.txt &
     
     nohup java -jar 自己的springboot项目.jar >日志文件名.log 2>&1 &
     
     nohup java -jar 自己的springboot项目.jar >/dev/null 2>&1 &

## 11. 查看、修改网络接口配置

> [ifconfig 命令详解](https://blog.csdn.net/u011857683/article/details/83758503)
>
> [linux服务器查看公网IP信息的方法](https://www.cnblogs.com/ksguai/p/6090115.html)
>
> [linux命令之ifconfig详细解释](https://www.cnblogs.com/jxhd1/p/6281427.html)
>
> [ifconfig 命令详解](https://blog.csdn.net/u011857683/article/details/83758503)
>
> [ifconfig 中的 eth0 eth0:1 eth0.1 与 lo](https://www.cnblogs.com/jokerjason/p/10695189.html)]
>
> [Linux 查看网卡全双工 还是半双工 以及设置网卡为半双工](https://www.osgeo.cn/post/30ggg)

1. ifconfig

   + 查看自己的公网IP即信息

     ```shell
     curl ifconfig.me
     #不起效可以用下面这条
     curl cip.cc
     ```

     

## 12. 网络通讯TCP/UDP等

> [Linux nc命令](https://www.runoob.com/linux/linux-comm-nc.html)
>
> [linux系统下tcpdump和nc工具的使用](https://blog.51cto.com/jachy/1753951)
>
> [Linux基础：用tcpdump抓包](https://www.cnblogs.com/chyingp/p/linux-command-tcpdump.html)

1. nc

   + 检测UDP端口是否连通(发UDP包)

     nc -vuz IPv4地址 端口号

     例如 nc -vuz 123.123.123.123 12345

2. tcpdump

   + Linux抓包(比如监听UDP包接收发送等)

     tcpdump -vvv -X -n udp port 端口号

     例如tcpdump -vvv -X -n udp port 12345

## 13. Docker操作RabbitMQ

> [docker安装与使用](https://www.cnblogs.com/glh-ty/articles/9968252.html)
>
> [docker快速安装rabbitmq](https://www.cnblogs.com/angelyan/p/11218260.html)
>
> [docker 安装rabbitMQ](https://www.cnblogs.com/yufeng218/p/9452621.html)
>
> [阿里云-docker安装rabbitmq及无法访问主页](https://www.cnblogs.com/hellohero55/p/11953882.html)
>
> [docker运行jar文件](https://www.cnblogs.com/zhangwufei/p/9034997.html)

1. docker安装RabbitMQ后，启动

   ```shell
   docker run -d --name rabbitmq3.8.2 -p 5672:5672 -p 15672:15672 -v `pwd`/data:/var/lib/rabbitmq --hostname XX-xxx-Docker-RabbitMQ-主机名 -e RABBITMQ_DEFAULT_VHOST=XX-xxx-Docker-RabbitMQ-vhost -e RABBITMQ_DEFAULT_USER=用户名 -e RABBITMQ_DEFAULT_PASS=密码 docker中RabbitMQ的imageID
   ```


2. docker使用国内的加速源

   ```shell
   vim /etc/docker/daemon.json
   ```

   修改上述json文件为以下内容

   ```json
   {"registry-mirrors": ["http://95822026.m.daocloud.io","https://registry.docker-cn.com","http://hub-mirror.c.163.com","https://almtd3fa.mirror.aliyuncs.com"]}
   ```

## 14. 用户操作

1. 切换用户

   ```she
   su 用户名
   ```

## 15. 网络检测

1. 查看是否被木马开后门(查看建立连接的程序)

   ```shell
   netstat -nb
   ```

   