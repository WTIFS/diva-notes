下载 `profile` 文件

```bash
wget localhost:21477/debug/pprof/heap
```



分析 `profile`

```bash
go tool pprof -http=:8080 heap
```

