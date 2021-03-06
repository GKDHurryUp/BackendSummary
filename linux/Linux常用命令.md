
## 进程相关 ##


### ps ###
ps命令就是最基本进程查看命令。使用该命令可以确定有哪些进程正在运行和运行的状态、进程是否结束、进程有没有僵尸、哪些进程占用了过多的资源等等，显示**瞬间**进程的状态，并**不动态连续**

参数：
> -A ：所有的进程均显示出来，与 -e 具有同样的效用
  -f ：做一个更为完整的输出

常用：ps -ef | grep <pid>

### top ###
top命令经常用来监控linux的系统状况，是常用的性能分析工具，能够**实时显示**系统中各个进程的资源占用情况。

参数：
-p <pid>: 通过指定监控进程ID来仅仅监控某个进程的状态

### netstat

显示网络连接、路由表和端口信息，可以让用户得知有哪些网络连接正在运作

- netstat -a （列出所有端口）

- netstat -at （列出所有 tcp 端口 ）
- netstat -au （列出所有 udp 端口 ）

### lsof

列出当前系统打开的文件（lists open files ），查看端口占用

- lsof -i （用以显示符合条件的进程情况）

```shell
lsof -i tcp:22
```

列出所有端口

```shell
netstat -ntlp
```

### kill ###


用来杀死系统中的进程

kill -Signal pid
> 1 终端断线
	**2 中断（等同 Ctrl + C）**
	3 退出（同 Ctrl + \）
	**9 强制终止**
	**15 终止（可以使得进程在退出之前清理并释放资源）**
	18 继续（与19相反）
	19 暂停（等同 Ctrl + Z）


### grep ###
grep [-acinv] [--color=auto] '搜寻字符串' filename
选项与参数：
-a ：将 binary 文件以 text 文件的方式搜寻数据
-c ：计算找到 '搜寻字符串' 的次数
-i ：忽略大小写的不同，所以大小写视为相同
-n ：顺便输出行号
-v ：反向选择，亦即显示出没有 '搜寻字符串' 内容的那一行！
--color=auto ：可以将找到的关键词部分加上颜色的显示喔！

### strace ###
追踪系统调用

```shell
strace -ff -o out java Xxxx
```

### man ###
man 2 查看对应的函数说明

### nc ###
net connet

## 抓包

tcpdump命令来抓取数据包，dump the traffic on a network，根据使用者的定义对网络上的数据包进行截获的包分析工具。 tcpdump可以将网络中传送的数据包的“头”完全截获下来提供分析。它支持针对网络层、协议、主机、网络或端口的过滤，并提供and、or、not等逻辑语句来帮助你去掉无用的信息

### **监视指定网络接口的数据包**

```
tcpdump -i eth1
```

### **监视指定主机和端口的数据包**

如果想要获取主机210.27.48.1接收或发出的telnet包，使用如下命令

```
tcpdump tcp port 23 and host 210.27.48.1
```

## 查看信息

### ls -lh

drwxrwxr-x 6 root  root  4.0K Jun 11 09:25 redis-5.0.8

- 第一位，d为目录文件，l连接文件，-普通文件，p管道
- 后面三位三位来看，r可读，w可写，x可执行
  - 2-4位，文件所有者
  - 5-7位，文件所属组
  - 8-10位，其他用户

- 数字代表目录/链接个数
  - 对于目录文件，表示它的第一级子目录的个数
  - 对于其他文件，表示指向它的链接文件的个数
- root表示该文件的所有者/创建者（owner）及其所在的组（group）

###  查看文件磁盘使用情况

```shell
df -h
```

### uname

uname可显示电脑以及操作系统的相关信息。

- -r或--release 　显示操作系统的发行编号。
- -v 　显示操作系统的版本。

### crontab

用来固定时间或固定间隔执行程序的命令

```shell
crontab [ -u user ] file
```

### Sed

sed 全名为 stream editor，流编辑器，用程序的方式来编辑文本，支持正则表达式，可以进行大量的复杂的文本编辑操作

sed 是一种非交互式编辑器(即用户不必参与编辑过程)，它使用预先设定好的编辑指令对输入的文本进行编辑，完成之后再输出编辑结构

工作原理：

 sed会一次处理一行内容。处理时，把当前处理的行存储在临时缓冲区中，成为"模式空间"，接着用sed命令处理缓冲区中的内容，处理完成后，把缓冲区的内容送往屏幕。接着处理下一行，这样不断重复，直到文件末尾。文件内容并没有改变，除非你使用重定向存储输出。



```shell
sed -i  //直接修改读取的文件内容，而不是输出到终端
```

### tail

如何实时查看Linux下日志？

使用tail命令的-f选项可以方便的查阅正在改变的日志文件,tail -f filename会把filename里最尾部的内容显示在屏幕上,并且不断刷新，使你看到最新的文件内容

```shell
tail -f  *.log
```



## 解压

### tar.gz

```shell
tar -zxvf *.tar.gz  -C /usr/java
```

