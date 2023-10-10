gomonkey 不支持 Mac M2 芯片：使用如下方法运行测试：

```bash
 GOARCH=amd64 go test -exec='arch -x86_64' -gcflags='-N -l' -run XXX
```



##### gomod replace

[【Go mod 学习之 replace 篇】解决 go 本地依赖、无法拉取依赖、禁止依赖等问题](https://blog.csdn.net/qq_24433609/article/details/127323097)



##### gomod graph

绘制依赖图
