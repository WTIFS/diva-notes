统计 TCP 各状态连接数

```bash
netstat -an | awk '/^tcp/ {++state[$NF]} END {for(i in state) print i,"\t",state[i]}'
```

