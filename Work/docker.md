##### 标准命令

```
docker run -it --name 容器名称 -v 容器外目录:容器内目录 镜像名 /bin/bash
```



##### 参考命令

```
docker run -it --name golang -v /Users/chris/Projects/go/src:/workspace -v /Users/chris:/root golang:latest /bin/bash

docker run -it --name golang1.17-amd64 -v /Users/chris/Projects/go/src:/workspace -v /Users/chris:/root --platform linux/amd64 golang:1.17 /bin/bash

docker run -it --name golang1.21-amd64 -v /Users/chris/Projects/go/src:/workspace -v /Users/chris:/root --platform linux/amd64 golang:1.21 /bin/bash

docker run -it --name gcc -v /Users/chris/Projects/go/src:/workspace -v /Users/chris:/root gcc /bin/bash

export GOPRIVATE="gitlab.xxx.com"
```



-d: 后台运行

-p: 端口映射

-v: 挂载目录

-w: 默认工作目录



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



##### 重启

```
docker restart {containerID}
```



##### 启动北极星
```bash
docker run -d --name polaris-server -p 8190:8090 -p 8191:8091 polarismesh/polaris-server
docker run -d --name polaris-console -p 8180:8080 --link polaris-server

docker run -d --name polaris-server -p 8090:8090 -p 8091:8091 polarismesh/polaris-server
```

##### 启动 jaeger
```bash
# 16686: jaeger UI 端口
# 14268: jaeger API 端口
# 6831 jaeger agent 端口
docker run -d -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 -p 16686:16686 -p 14268:14268  -p 14269:14269   -p 9411:9411 -p 6831:6831/udp jaegertracing/all-in-one:latest
```
