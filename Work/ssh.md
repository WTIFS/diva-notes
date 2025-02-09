**ssh-keygen** 产生公钥与私钥对.

**ssh-copy-id** 将本机的公钥复制到远程机器的authorized_keys文件中

```
ssh-copy-id -p 端口 -i ~/.ssh/id_rsa.pub 用户名字@192.168.x.xxx
```



或者手动 

```bash
vim ~/.ssh/authorized_keys
# 将本机的 id_rsa.pub 贴入
```



ssh连接无操作一段时间会断开怎么办？

```bash
# 修改 /etc/ssh/sshd_config 文件
# 每60秒连接一下服务器，最大重试次数为5次
ClientAliveInterval 60
ClientAliveCountMax 5
```

