##### 标准命令

```
docker run -it --name 容器名称 -v 容器外目录:容器内目录 镜像名 /bin/bash
```



##### 参考命令

```
docker run -it --name golang -v /Users/chris/Projects/go/src:/workspace -v /Users/chris:/root golang:latest /bin/bash

docker run -it --name golang1.17-amd64 -v /Users/chris/Projects/go/src:/workspace -v /Users/chris:/root -v /usr/local/include:/usr/local/include --platform linux/amd64 golang:1.17 /bin/bash

docker run -it --name golang1.24-amd64 -v /Users/chris/Projects:/workspace -v /Users/chris:/root -v /usr/local/include:/usr/local/include --platform linux/amd64 golang:1.24.1 /bin/bash

docker run -it --name golang1.24 golang:1.24.1 /bin/bash

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
docker run -d --name polaris-console -p 8080:8080 polarismesh/polaris-console

docker run -d --name polaris-standalone -p 8090:8090 -p 8091:8091 -p 8080:8080 polarismesh/polaris-standalone:latest
```

##### 启动 jaeger
```bash
# 16686: jaeger UI 端口
# 14268: jaeger API 端口
# 6831 jaeger agent 端口
docker run -d -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 --net=host jaegertracing/all-in-one:latest

docker run -d -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 -p 16686:16686 -p 14268:14268  -p 14269:14269 -p 9411:9411 -p 6831:6831/udp jaegertracing/all-in-one:latest
```

**启动 apollo**

```bash
git clone https://github.com/apolloconfig/apollo-quick-start.git
docker-compose up
```

**启动mongo**

```bash
docker run -d --name mongo -p 27017:27017 mongo
```

**rabbitmq**

```go
docker run -d --hostname my-rabbit --name rabbitmq -p 8480:8080 -p 5672:5672 -p 15672:15672 rabbitmq
```

**kafka**

```bash
docker run -d --name kafka4.0 -p 9092:9092 apache/kafka:4.0.0

docker run --name kafka4.0 -p 9092:9092 apache/kafka:4.0.0
```





##### 编译

```
docker build -t harbor.xxx.com/dsp/xxx:v0.0.7 -f docker/Dockerfile .
```





## 分析Docker使用了多少空间

```bash
docker system df

TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          61        16        21.1GB    15.25GB (72%)
Containers      69        0         12.26MB   12.26MB (100%)
Local Volumes   3         2         539.1MB   50.04MB (9%)
Build Cache     76        0         1.242GB   1.242GB
```



要尽可能地清理，不包括正在使用的组件，请运行这个命令：

```sh
# 全部清理（磁盘、镜像、容器等）
docker system prune -a -f

# 清理不用的磁盘
docker volume prune -a
```



**查询僵尸文件**

在 Docker 1.9 以上的版本中，官方提供用于查询僵尸文件的命令：

```
docker volume ls -qf dangling=true

# 删除所有dangling数据卷（即无用的Volume，僵尸文件）
docker volume rm $(docker volume ls -qf dangling=true)
```



##### 镜像列表

```
{
    "registry-mirrors": [
        "https://dockerhub.icu",
        "https://docker.chenby.cn",
        "https://docker.1panel.live",
        "https://docker.awsl9527.cn",
        "https://docker.anyhub.us.kg",
        "https://dhub.kubesre.xyz",
        "https://do.nark.eu.org",
        "https://dc.j8.work",
        "https://docker.m.daocloud.io",
        "https://dockerproxy.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://docker.nju.edu.cn"
    ]
}
```

