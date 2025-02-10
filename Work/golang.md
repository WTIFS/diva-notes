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

go test -gcflags "all=-N -l" ./... -coverprofile=cover.out
gocov convert cover.out | gocov-xml > coverage.xml
diff-cover coverage.xml --compare-branch=master --html-report report.html
# 打开 report.xml
```



查看全量具体哪些行

```go
go tool cover -html=coverage.out  
```



#### gitlab-runner 集成

gitlab-ci.yml 中增加以下配置：

```
make test = go test -gcflags "all=-N -l" -cover -v -test.short `go list ./... | grep -vE 'example|test|mock'`

unit-test:
  stage: test
  tags:
    - xxx
  only:
    - merge_requests
  script:
    - make test > test.out
    - go-junit-report -in test.out -iocopy -out coverage.xml | grep coverage
  artifacts:
    paths:
      - coverage.xml
    reports:
      junit: coverage.xml
```



`gitlab - Settings - CI/CD - General pipelines` 中的 `Test coverage parsing` 增加以下配置：

```
\d+.\d+% of statements
```
