gomonkey 不支持 Mac M2 芯片：使用如下方法运行测试：

```bash
 GOARCH=amd64 go test -exec='arch -x86_64' -gcflags='-N -l' -run XXX
```



##### gomod replace

[【Go mod 学习之 replace 篇】解决 go 本地依赖、无法拉取依赖、禁止依赖等问题](https://blog.csdn.net/qq_24433609/article/details/127323097)



##### gomod graph

绘制依赖图




#### 单测

查看增量单测覆盖率详情

```bash
pip install diff_cover

go install github.com/axw/gocov/gocov@latest
go install github.com/AlekSi/gocov-xml@latest

go test -gcflags "all=-N -l" -v ./... -coverprofile=cover.out
gocov convert cover.out | gocov-xml > coverage.xml
diff-cover coverage.xml --compare-branch=master --html-report report.html
# 打开 report.xml
```
