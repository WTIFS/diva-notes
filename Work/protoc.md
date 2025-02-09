1. 在官方Github https://github.com/protocolbuffers/protobuf/releases 找发行版 tar.gz 下载后解压
1. 设置编译后二进制所在的目录。以2.5.0版本为例：

```bash
./configure --prefix=/usr/local/protobuf2.5.0
```

3. 编译并安装

```bash
make
sudo make install
```

4. 改PATH

```bash
export PROTOBUF=/usr/local/protobuf2.5.0
export PATH=$PATH:$PROTOBUF/bin
```

5. 检查

```bash
protoc --version
```



编译 pb 文件

```go
protoc --go_out=plugins=grpc:. hello.proto
```



#### protoc-gen-go

你可以修改 protoc-gen-go 的源码，并依此编译自己的插件

源码地址：https://github.com/golang/protobuf

```bash
cd protoc-gen-go
go build main.go
mv main /Users/xxx/Projects/go/bin/protoc-gen-go

然后 
protoc -I=. --go_out=. hello.proto
protoc -I=. --go-grpc_out=.  hello.proto
```

