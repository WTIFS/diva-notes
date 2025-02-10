在 Linux 系统中，systemd 是一个非常强大的系统管理工具，它可以自动启动，停止和重新启动服务，还可以在需要时重启服务。

下面是将一个 node.js 服务写成 systemd 服务的步骤：

1. 创建一个 systemd 配置文件，比如 `/etc/systemd/system/my-node-app.service`，并编辑该文件：

```
sudo touch /etc/systemd/system/my-node-app.service
```

1. 在该文件中添加以下内容：

```
[Unit]
Description=My Node.js App
After=network.target

[Service]
User=<username>
ExecStart=/usr/bin/node server.js # 这里要用绝对路径
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

其中：

- `Description`：服务的名称或描述信息。
- `After`：定义服务启动时需要等待的依赖项。这里是网络，表示在网络服务启动后才启动该服务。
- `User`：该服务运行的用户。
- `WorkingDirectory`：服务的工作目录。
- `ExecStart`：启动服务的命令，可以是绝对路径或者命令名，这里以 node.js 服务为例。
- `Restart`：服务退出后是否自动重启。
- `RestartSec`：重启服务的时间间隔。
- `WantedBy`：定义该服务所属的系统运行级别，这里是多用户运行级别。

1. 保存并关闭该文件，然后重新加载 systemd 配置文件：

```
sudo systemctl daemon-reload
```

1. 启动服务并设置开机自启：

```
sudo systemctl start my-node-app
sudo systemctl enable my-node-app
```

这样，就将一个 node.js 服务写成 systemd 服务了。你可以使用以下命令控制服务：

- `sudo systemctl start my-node-app`：启动服务
- `sudo systemctl stop my-node-app`：停止服务
- `sudo systemctl restart my-node-app`：重启服务
- `sudo systemctl status my-node-app`：查看服务状态
- `sudo systemctl enable my-node-app`：设置开机自启
- `sudo systemctl disable my-node-app`：禁止开机自启