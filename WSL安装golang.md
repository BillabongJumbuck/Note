下载，使用中国科学技术大学的镜像

```bash
wget https://mirrors.ustc.edu.cn/golang/go1.24.0.linux-amd64.tar.gz    ## 注意按需替换版本号
```

解压

```bash
sudo tar -C /usr/local -xzf go1.24.0.linux-amd64.tar.gz
```

配置 `fish`环境变量。添加下面内容到`~/.config/fish/config.fish`

```bash
# Go lang configuration
set -gx PATH $PATH /usr/local/go/bin
set -gx GOPATH $HOME/go
set -gx PATH $PATH $GOPATH/bin
```

配置拉取 Go 模块（Module）的代理

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

验证安装

```bash
go version  # 出现版本号则为成功
```

