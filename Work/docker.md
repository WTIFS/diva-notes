##### 标准命令

```
docker run -it --name 容器名称 -v 容器外目录:容器内目录 镜像名 /bin/bash
```



##### 参考命令

```
docker run -it --name golang -v /Users/chris/Projects/go/src:/workspace -v /Users/chris:/root golang:latest /bin/bash

docker run -it --name golang1.17-amd64 -v /Users/chris/Projects/go/src:/workspace -v /Users/chris:/root --platform linux/amd64 golang:1.17 /bin/bash

docker run -it --name gcc -v /Users/chris/Projects/go/src:/workspace -v /Users/chris:/root gcc /bin/bash

export GOPRIVATE="gitlab.xxx.com"
```



-d: 后台运行

-p: 端口映射

-v: 挂载目录

-w: 默认工作目录

```go
docker run -d --name polaris-server -p 8190:8090 -p 8191:8091 polarismesh/polaris-server
docker run -d --name polaris-console -p 8180:8080 --link polaris-server --link prometheus polaris-console3
docker run -d --user root --name prometheus --link polaris-server -p 9090:9090 prometheus2
```



##### 进入 docker

```go
docker exec -it 容器ID bash
```



##### 修改镜像

进入容器并修改后退出

```
docker commit 容器id 要保存的镜像名称
然后用 docker run 这个新镜像
```

