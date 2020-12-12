# Shell编程复习

> 最近一段时间，需要频繁使用Linux，所以就回过头来复习一下shell编程。
>
> [Shell教程 ｜ 菜鸟教程 ](https://www.runoob.com/linux/linux-shell.html)	<=	回顾简单Shell编程的话， 把基础语法过一遍就差不多了。不需要再额外看视频、看书了。
>
> 当然如果需要内核编程的话，还是需要注意对C语言、操作系统、计算机网络知识复习、巩固。

# 0. 零散知识

1. **#!** 告诉系统其后路径所指定的程序即是解释此脚本文件的 Shell 程序。

   ```shell
    #!/bin/bash
    #!/bin/sh
    
    ....
   ```

2. 编写shell程序后，两种执行方式：

   1. 给shell程序可执行权限

      ```shell
      [root@VM_0_5_centos shell_test]# ls -al
      total 12
      drwxr-xr-x  2 root root 4096 Dec 12 14:48 .
      drwxr-xr-x. 8 root root 4096 Dec 12 14:47 ..
      -rw-r--r--  1 root root   33 Dec 12 14:48 echo_test
      
      [root@VM_0_5_centos shell_test]# ./echo_test
      -bash: ./echo_test: Permission denied
      
      [root@VM_0_5_centos shell_test]# chmod +x echo_test 
      
      [root@VM_0_5_centos shell_test]# ./echo_test 
      hello world!
      ```

   2. 作为shell脚本解释器的参数（这时候shell程序第一行指定的解释器信息不生效；此时shell程序本身不需要有可执行权限）

      ```shell
      [root@VM_0_5_centos shell_test]# ls -al
      total 12
      drwxr-xr-x  2 root root 4096 Dec 12 14:48 .
      drwxr-xr-x. 8 root root 4096 Dec 12 14:47 ..
      -rw-r--r--  1 root root   33 Dec 12 14:48 echo_test
      
      [root@VM_0_5_centos shell_test]# bash echo_test 
      hello world!
      ```

# 1. Shell变量

## 1. 定义变量

​	**定义变量时，变量名不加美元符号**（$，PHP语言中变量需要），如：

```shell
param=123456
```

​	注意，**变量名和等号之间不能有空格**。同时，变量名的命名须遵循如下规则：

+ 命名只能使用英文字母，数字和下划线，首个字符不能以数字开头。
+ 中间不能有空格，可以使用下划线（_）。
+ 不能使用标点符号。
+ **不能使用bash里的关键字（可用help命令查看保留关键字）**。

除了显式地直接赋值，还可以用语句给变量赋值，如：

```shell
for file in `ls /etc`
# 或
for file in $(ls /etc)
```

## 2. 使用变量

​	使用一个定义过的变量，只要**在变量名前面加美元符号**即可

```shell
[root@VM_0_5_centos shell_test]# param_01=hello
[root@VM_0_5_centos shell_test]# echo $param_01
hello
[root@VM_0_5_centos shell_test]# echo ${param_01}
hello
```

变量名外面的花括号是可选的，加不加都行，**加花括号是为了帮助解释器识别变量的边界**，比如下面这种情况：

```shell
for skill in Ada Coffe Action Java; do
    echo "I am good at ${skill}Script"
done

### test
[root@VM_0_5_centos shell_test]# for skill in Ada Coffe Action Java; do
> echo "I am good at ${skill}Script"
> done
I am good at AdaScript
I am good at CoffeScript
I am good at ActionScript
I am good at JavaScript
```

​	如果不给skill变量加花括号，写成echo "I am good at $skillScript"，解释器就会把$skillScript当成一个变量（其值为空），代码执行结果就不是我们期望的样子了。

**推荐给所有变量加上花括号，这是个好的编程习惯**。

## 3. 只读变量

​	**使用 readonly 命令可以将变量定义为只读变量，只读变量的值不能被改变**。

下面的例子尝试更改只读变量，结果报错：

```shell
[root@VM_0_5_centos shell_test]# myURL="baidu.com"
[root@VM_0_5_centos shell_test]# readonly myURL 
[root@VM_0_5_centos shell_test]# myURL="google.com"
-bash: myURL: readonly variable
```

## 4. 删除变量

​	**使用 unset 命令可以删除变量**	(变量被删除后不能再次使用；unset 命令不能删除只读变量)。语法：

```shell
unset variable_name
```

用例：

```shell
[root@VM_0_5_centos shell_test]# myparam=1
[root@VM_0_5_centos shell_test]# yourparam=2
[root@VM_0_5_centos shell_test]# readonly yourparam
[root@VM_0_5_centos shell_test]# echo $myparam 
1
[root@VM_0_5_centos shell_test]# echo $yourparam 
2
[root@VM_0_5_centos shell_test]# unset myparam
[root@VM_0_5_centos shell_test]# echo $myparam

[root@VM_0_5_centos shell_test]# unset yourparam
-bash: unset: yourparam: cannot unset: readonly variable
```

## 5. 变量类型

运行shell时，会同时存在三种变量：

- **1) 局部变量** 局部变量在脚本或命令中定义，仅在当前shell实例中有效，其他shell启动的程序不能访问局部变量。
- **2) 环境变量** 所有的程序，包括shell启动的程序，都能访问环境变量，有些程序需要环境变量来保证其正常运行。必要的时候shell脚本也可以定义环境变量。
- **3) shell变量** shell变量是由shell程序设置的特殊变量。shell变量中有一部分是环境变量，有一部分是局部变量，这些变量保证了shell的正常运行

## 6. Shell字符串

​	字符串是shell编程中最常用最有用的数据类型（除了数字和字符串，也没啥其它类型好用了），**字符串可以用单引号，也可以用双引号，也可以不用引号**。

### 1. 单引号

```shell
[root@VM_0_5_centos shell_test]# param='hello'
[root@VM_0_5_centos shell_test]# echo 'say ${param}'
say ${param}
[root@VM_0_5_centos shell_test]# echo $param
hello
[root@VM_0_5_centos shell_test]# echo 'say''''${param}'
say${param}
```

单引号字符串的限制：

- **单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的**；
- **单引号字串中不能出现单独一个的单引号（对单引号使用转义符后也不行），但可成对出现，作为字符串拼接使用**。

### 2. 双引号

```shell
[root@ash ~]# str='hello'
[root@ash ~]# echo "say $str \\\' ' '$str' world"
say hello \\' ' 'hello' world
```

双引号的优点：

- **双引号里可以有变量**
- **双引号里可以出现转义字符**

### 3. 拼接字符串

编写shell程序

```shell
your_name="runoob"
# 使用双引号拼接
greeting="hello, "$your_name" !"
greeting_1="hello, ${your_name} !"
echo $greeting  $greeting_1
# 使用单引号拼接
greeting_2='hello, '$your_name' !'
greeting_3='hello, ${your_name} !'
echo $greeting_2  $greeting_3
```

输出结果为：

```shell
hello, runoob ! hello, runoob !
hello, runoob ! hello, ${your_name} !
```

### 4. 获取字符串长度

指令即`${#字符串变量}`

```shell
[root@ash shell_test]# str='123456\n\n'
[root@ash shell_test]# echo ${#str}
10
[root@ash shell_test]# str="123456\n\n"
[root@ash shell_test]# echo ${#str}
10
```

### 5. 提取子字符串

指令即`${字符串变量:包含的起始下标:总截取长度}`。（**注意**：第一个字符的索引值为 **0**。）

以下实例从字符串第 **3**个字符开始截取 **5** 个字符：

```shell
[root@ash shell_test]# str='0123456789'
[root@ash shell_test]# echo ${str:2:5}
23456
```

### 6. 查找子字符串

查找字符 **i** 或 **o** 的位置(哪个字母先出现就计算哪个)：

+ 这里的位置，是从1开始计数，而不是从0开始。
+ **只有不存在指定的子字符串时，才会返回0**

```shell
[root@ash shell_test]# cat echo_test 
string="runoob is a great site"
echo `expr index "$string" io`

[root@ash shell_test]# bash echo_test 
4

[root@ash shell_test]# cat echo_test 
string="ash is a great site"
echo `expr index "$string" zo`

[root@ash shell_test]# bash echo_test 
0
```

**注意：** 以上脚本中 **`** 是反引号，而不是单引号 **'**，不要看错了哦。

（这里如果去掉$string的双引号，报错`expr: syntax error`）

## 7. Shell 数组

+ **bash支持一维数组（不支持多维数组），并且没有限定数组的大小**。

+ 类似于 C 语言，**数组元素的下标由 0 开始编号**。
+ 获取数组中的元素要利用下标，下标可以是整数或算术表达式，其值应大于或等于 0。

### 1. 定义数组

在 Shell 中，**用括号来表示数组，数组元素用"空格"符号分割开**。定义数组的一般形式为：

```shell
数组名=(值1 值2 ... 值n)
```

例如：

```shell
array_name=(value0 value1 value2 value3)
```

或者

```shell
array_name=(
value0
value1
value2
value3
)
```

还可以单独定义数组的各个分量：

```shell
array_name[0]=value0
array_name[1]=value1
array_name[n]=valuen
```

可以不使用连续的下标，而且下标的范围没有限制。

### 2. 读取数组

读取数组元素值的一般格式是：`${数组名[下标]}`

例如：

```shell
valuen=${array_name[n]}
```

使用 **@**或者\* 符号可以获取数组中的所有元素，例如：

```shell
echo ${array_name[@]}
echo ${array_name[*]}
```

### 3. 获取数组的长度

**获取数组长度的方法与获取字符串长度的方法相同**，例如：

```shell
# 取得数组元素的个数
length=${#array_name[@]}
# 或者
length=${#array_name[*]}
# 取得数组单个元素的长度
lengthn=${#array_name[n]}
```

```shell
[root@ash shell_test]# arr=(123 456 78910)
[root@ash shell_test]# echo ${#arr[2]}
5
[root@ash shell_test]# echo ${#arr[@]}
3
```

# 2. Shell注释

## 1. 单行注释

以 **#** 开头的行就是注释，会被解释器忽略。

通过每一行加一个 **#** 号设置多行注释，像这样：

```shell
#--------------------------------------------
# 这是一个注释
# author：菜鸟教程
# site：www.runoob.com
# slogan：学的不仅是技术，更是梦想！
#--------------------------------------------
##### 用户配置区 开始 #####
#
#
# 这里可以添加脚本描述信息
# 
#
##### 用户配置区 结束  #####
```

​	*如果在开发过程中，遇到大段的代码需要临时注释起来，过一会儿又取消注释，怎么办呢？每一行加个#符号太费力了，可以把这一段要注释的代码用一对花括号括起来，定义成一个函数，没有地方调用这个函数，这块代码就不会执行，达到了和注释一样的效果。*

## 2. 多行注释

**个人还是比较建议直接用单行注释来替代多行注释，省得出问题。**

多行注释还可以使用以下格式：

```shell
:<<EOF
注释内容...
注释内容...
注释内容...
EOF
```

EOF 也可以使用其他符号:

（然而我试了下，有时候会出问题，比如单引号的那个，就报错了）

```shell
:<<'
注释内容...
注释内容...
注释内容...
'

:<<!
注释内容...
注释内容...
注释内容...
!
```

# 3. Shell传递参数

