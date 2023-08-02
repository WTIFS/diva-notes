gomonkey 不支持 Mac M2 芯片：使用如下方法运行测试：

```bash
 GOARCH=amd64 go test -exec='arch -x86_64' -gcflags='-N -l' -run XXX
```

