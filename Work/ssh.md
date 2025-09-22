**ssh-keygen** 产生公钥与私钥对.

**ssh-copy-id** 将本机的公钥复制到远程机器的authorized_keys文件中

```shell
ssh-copy-id -p 端口 -i ~/.ssh/id_rsa.pub 用户名字@192.168.x.xxx
```



或者手动 

```shell
vim ~/.ssh/authorized_keys
# 将本机的 id_rsa.pub 贴入
```



ssh连接无操作一段时间会断开怎么办？

```shell
# 修改 /etc/ssh/sshd_config 文件
# 每60秒连接一下服务器，最大重试次数为5次
ClientAliveInterval 60
ClientAliveCountMax 5
```



# 完整流程

##### 1. 创建用户

```bash
sudo su -
useradd 用户名
```



##### 2. 清除用户密码

```bash
passwd -d 用户名
```



##### 3. 授予用户root权限

```shell
vim /etc/sudoers

## Allow root to run any commands anywhere
用户名   ALL=(ALL)       ALL
```



##### 4. 添加 ssh 公钥
```shell
su 用户名
mkdir ~/.ssh
chmod 700 ~/.ssh
vim ~/.ssh/authorized_keys
# 将本机的 id_rsa.pub 贴入
chmod 600 ~/.ssh/authorized_keys
```

   

