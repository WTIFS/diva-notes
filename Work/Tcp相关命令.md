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
# $5=打印第5列；-d:=用:分隔；-f1=取分隔后的第一列
netstat -nat | grep "ESTABLISHED" | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head -n 10

sudo tcpdump -i eth0 -tnn -c 10000 | awk -F "." '{print $1"."$2"."$3"."$4}' | sort | uniq -c | sort -nr | head -n 10
```



##### 检视 header

```bash
tcpdump -A -c 10 -s 10240 'tcp port 13066 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)' | egrep --line-buffered "^........(GET |HTTP\/|POST |HEAD )|^[A-Za-z0-9-]+: " | sed -r 's/^........(GET |HTTP\/|POST |HEAD )/\n\1/g'

tcpdump -i any -s 0 -A -c 1 'tcp port 13066' | grep -E '^(x-cpu.*):'
```



##### 查看端口号 

```
netstat -tunlp | grep 关键词
lsof -i:8080
```



#### 统计本机 TCP 各状态连接数

```
netstat -an | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'

netstat -nat | grep -i "established" | wc -l
ss -t state established | wc -l

```



#### 查询域名DNS

```bash
nslookup www.baidu.com
```

