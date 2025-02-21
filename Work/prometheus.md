##### 安装

```go
mkdir -p /data/lib && cd /data/lib

wget https://github.com/prometheus/prometheus/releases/download/v2.41.0/prometheus-2.41.0.linux-amd64.tar.gz

tar -xvf prometheus-2.41.0.linux-amd64.tar.gz
rm -f prometheus-2.41.0.linux-amd64.tar.gz
cd prometheus-2.41.0.linux-amd64/

vim start.sh

pkill -f prometheus
sleep 1
./prometheus --web.enable-admin-api --storage.tsdb.retention.time=24h &

chmod +x start.sh
sh start.sh

# grafana 账密: admin/admin
yum install docker
systemctl start docker
docker run -d -p 3000:3000 --net=host --name=grafana grafana/grafana
# grafana 设置数据源 http://localhost:9090
```





##### 设置数据最大生命周期

默认为 15天，最小为 2h

```--storage.tsdb.retention.time=1y
# 启动时增加以下参数：
--storage.tsdb.retention.time=1y
```

