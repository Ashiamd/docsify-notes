# Jenkins杂记

# 1. 安装和使用

> 零基础搭建
>
> [手把手教你利用Docker+jenkins部署你的网站](https://zhuanlan.zhihu.com/p/145604644)
>
> [Jenkins+Maven+Github+Springboot实现可持续自动部署(非常详细)](https://www.cnblogs.com/rmxd/p/11609983.html)
>
> 
>
> 配置轮询git服务器，拉取更新并重新部署
>
> [Jenkins定时构建和轮询SCM设置说明](https://blog.csdn.net/MenofGod/article/details/81288987)
>
> [Jenkins定时构建与轮询SCM](https://www.cnblogs.com/panda-sweets/p/12554200.html)

上述文章贼详细。

需要注意的就是参考[Jenkins+Maven+Github+Springboot实现可持续自动部署(非常详细)](https://www.cnblogs.com/rmxd/p/11609983.html)时，需要修改最后文章下面的`.sh`可执行脚本文件，把里面的jdk、项目配置修改成自己实际的。

我自己gitee出现403问题，没法发送push的信号给jenkins，暂时没处理好，就用了折中的轮训SCM方案，每过一段时间到git服务器检查是否有新更新，有则部署jenkins项目的服务器自动拉取git项目并重新部署。

# 2. 常见异常/报错/问题

## 2.1 SSH key添加问题

> [jenkins:配置密钥时报错的解决：Failed to add SSH key. Message invalid privatekey(Jenkins 2.257)](https://www.cnblogs.com/architectforest/p/13707244.html)

1. 错误信息

   ```none
   jenkins.plugins.publish_over.BapPublisherException: Failed to add SSH key. Message [invalid privatekey: [B@60373f7]
   ```

2. 产生问题的原因

   生成密钥的openssh的版本过高，jenkins不支持

   ```shell
   [root@localhost ~]# ssh-keygen -t rsa
   ```

   查看所生成私钥的格式:

   ```shell
   [root@localhost ~]$ more .ssh/id_rsa
   -----BEGIN OPENSSH PRIVATE KEY-----
   b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
   …
   ```

   密钥文件首行（jenkins 2.2.57 版本在检验密钥时还不支持这种格式）

   ```shell
   -----BEGIN OPENSSH PRIVATE KEY——
   ```

3. 解决方案

   指定格式生成密钥文件

   ```shell
   [root@localhost ~]# ssh-keygen -m PEM -t rsa -b 4096
   
   # -m 参数指定密钥的格式，PEM是rsa之前使用的旧格式
   # -b 指定密钥长度。对于RSA密钥，最小要求768位，默认是2048位。
   ```

   重新生成密钥文件

   ```shell
   [root@localhost ~]# more /root/.ssh/id_rsa
   -----BEGIN RSA PRIVATE KEY-----
   MIIJKAIBAAKCAgEA44rzAenw3N7Tpjy5KXJpVia5oSTV/HrRg7d8PdCeJ3N1AiZU
   ...
   ```

   密钥首行（这样改动后可以通过jenkins对密钥格式的验证）

   ```
   -----BEGIN RSA PRIVATE KEY-----
   ```

   

