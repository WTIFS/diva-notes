下载 `profile` 文件

```bash
wget localhost:21477/debug/pprof/heap
```



分析 `profile`

```bash
go tool pprof -http=:8080 heap
```



在线分析 `profile`

```bash
go tool pprof -inuse_space localhost:21315/debug/pprof/heap
```

