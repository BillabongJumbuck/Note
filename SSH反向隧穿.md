##  反向 SSH 隧道 (Reverse SSH Tunnel) 

反向 SSH 隧道的原理是，让内网（笔记本）主动发起连接到公网（ECS），并在公网主机上打开一个端口进行监听。这样，当尝试从任何地方（包括自己或其他人）通过 SSH 连接到 ECS 的特定端口时，ECS 会将这个连接通过已建立的隧道**转发**回笔记本电脑。

#### 1. 准备 ECS (公网服务器)

1. **修改 SSH 配置：** 需要允许 ECS 上的 `sshd` 服务监听非本地的转发端口。

   - 修改ssh配置文件

     ```Bash
     sudo vim /etc/ssh/sshd_config
     ```

   - 确保以下行被取消注释或添加：

     ```shell
     GatewayPorts yes
     ```

   - **重启 SSH 服务**使配置生效：

     ```Bash
     sudo systemctl restart sshd
     # 或者
     sudo service sshd restart
     ```

2. **防火墙/安全组设置：** 确保你的 ECS **安全组**允许入站流量访问你将用于转发的端口。

#### 2. 在笔记本上设置反向隧道 (内网服务器)

在你的 Ubuntu 笔记本上，执行以下命令来建立反向 SSH 隧道。假设：

-  USER_{ECS} : 你的 ECS 登录用户名 (例如 `root` 或 `ubuntu`)
- IP_{ECS}: 你的 ECS 公网 IP 地址
- PORT_{ECS}: ECS 上用于接收转发连接的端口 (例如 `2222`)
- PORT_{Local}: 笔记本上要被访问的服务端口 (SSH 默认为 `22`)

使用以下命令：

```Bash
ssh -N -R $PORT_{ECS}:localhost:$PORT_{Local} $USER_{ECS}@$IP_{ECS}
# 示例：
# ssh -N -R 2222:localhost:22 ubuntu@123.45.67.89
```

- **`-N`**: 不执行远程命令（只进行端口转发）。
- **`-R`**: **反向**端口转发。格式是 `[bind_address:]port:host:hostport`。这里的意思是：将 ECS 上的 $PORT_{ECS}$ 转发到本地（笔记本）的 `localhost:22`。

> **提示：** 隧道建立后，命令行会保持阻塞。如果连接中断（例如笔记本断网），隧道会断开。为了让隧道持久运行，建议使用 **`autossh`** 工具，它能自动检测和重启断开的 SSH 会话。

> 公钥复制命令：
>
> ```Bash
> ssh-copy-id ubuntu@150.158.113.67
> ssh-copy-id -i ~/.ssh/my_ecs_key.pub ubuntu@50.158.113.67
> ```

