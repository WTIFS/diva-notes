#### tcpdump

##### 抓包选项

```bash
-c 指定数量
-i 指定网卡
-n 显示IP代替host
-nn 显式port代替host
-q 快速输出，仅列出少数的传输协议信息
-t 在输出的每一行不打印时间戳；
-tt 在输出的每一行显示未经格式化的时间戳记
-v 输出稍微详细的信息
-vv 输入更详细的信息
-w 文件名 输出到文件

host 10.76.114.33 指定host，可用 and, or 等条件表达式
port 80 指定端口号
```

将dump文件下载到本地后使用 wireshark 可视化观察更方便



##### 统计IP

```bash
sudo tcpdump -i em1 host 123.58.118.1 -w em1.log

tcpdump -i em1 -tnn -c 1000 | awk -F "." '{print 1"."2"."3"."4}' > tcp.log
```



##### 按IP排序

```bash
sudo tcpdump -i em1 -tnn -c 10000 | awk -F "." '{print $1"."$2"."$3"."$4}' | sort | uniq -c |sort -nr | head -n 10
```



#### 统计本机 TCP 各状态连接数

```
netstat -an | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```



#### 查询域名DNS

```bash
nslookup www.baidu.com
```

