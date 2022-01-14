**ssh-keygen** 产生公钥与私钥对.

**ssh-copy-id** 将本机的公钥复制到远程机器的authorized_keys文件中

```
ssh-copy-id -p 端口 -i ~/.ssh/id_rsa.pub 用户名字@192.168.x.xxx
```

