##### 查看端口号 

```
netstat -tunlp | grep 关键词
lsof -i:8080
```



##### 服务列表 

```
systemctl list-units --type=service
```



**日期**

```
date +"%Y-%m-%d %H:00" -d "1 hour ago"
```



##### 清理log

```
chattr -a /var/log/messages
echo /dev/null > /var/log/messages
```



##### 清理磁盘

**du -h --max-depth=1** 查看当前目录，哪个文件占用最大

du -h -d 1

**du -sh \*** 查看当前目录下各文件及文件夹占用大小

du -s /usr/* | sort -rn 按大小倒序排序

**mac:**

du -d 1 -h | sort -rn

找到没有被删除的较大的文件

lsof | grep deleted

ssh copy

ssh-copy-id -i ~/.ssh/id_rsa.pub 用户名字@192.168.x.xxx



**压缩**

```
tar -cvf filename.tar dir --exclude=xxx
```



**解压**

1、*.tar 用 tar –xvf 解压

2、*.gz 用 gzip -d或者gunzip 解压

3、*.tar.gz和*.tgz 用 tar –xzf 解压

4、*.bz2 用 bzip2 -d或者用bunzip2 解压

5、*.tar.bz2用tar –xjf 解压

6、*.Z 用 uncompress 解压

7、*.tar.Z 用tar –xZf 解压

8、*.rar 用 unrar e解压

9、*.zip 用 unzip 解压



##### 软连接

```
sudo mount -uw /

ln -s 实际文件夹 /app/xxx/conf
```



**curl**

```
curl -X POST -D “name=@1.jpg” url
```



**查看当前ssh的用户**

```
who
```



**查看ssh日志**

```
last | less
```



**关机**

```bash
poweroff
```





#### bash脚本

$0 这个程式的执行名字

$n 这个程式的第n个参数值，n=1..9

$* 这个程式的所有参数,此选项参数可超过9个。

$@ 同上

$# 这个程式的参数个数

$$ 这个程式的PID(脚本运行的当前进程ID号)

$! 执行上一个背景指令的PID(后台运行的最后一个进程的进程ID号)

$? 执行上一个指令的返回值 (显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误)

$- 显示shell使用的当前选项，与set命令功能相同

-eq      //等于

-ne      //不等于

-gt       //大于 （greater than）

-lt       //小于  （less than）

-ge       //大于等于

-le       //小于等于


