##### 统计IP

```bash
sudo tcpdump -i em1 | grep "123.58.118" > em1.log

tcpdump -i em1  -tnn -c 1000 | awk -F "." '{print 1"."2"."3"."4}' > tcp.log
```



##### 按IP排序

```bash
sudo tcpdump -i em1  -tnn -c 10000 | awk -F "." '{print $1"."$2"."$3"."$4}' | sort | uniq -c |sort -nr | head -n 10
```



##### 参数说明

-i 指定网卡



##### 统计本机 TCP 各状态连接数

```
netstat -an | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

