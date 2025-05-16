---
title: docker开发环境
date: 2025-03-11T09:46:48Z
lastmod: 2025-04-10T15:03:30Z
tags: ["Docker"]
categories: ["Docker"]
---

## Docker开发环境
为了隔离开发环境与日常工作环境, 可以使用Docker来当作轻型的虚拟机.  

docker代理设置
在`/etc/systemd/system/docker.service.d/proxy.conf`中设置

```
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:3128" Environment="HTTPS_PROXY=http://proxy.example.com:3128"
```

创建环境

```
docker run -it --name container_name --network host -v ~/:/host images
```
## 问题解决
```
cabbaken is not in the sudoers file.
```

通过`usermod`将用户添加入sudoer组

```
usermod -aG sudo user_name
```

其通过修改`/etc/sudoers`将用户添加入sudoer组
同样的, 有时使用`sudo`执行一些程序会忽视一些环境变量, 可以修改`/etc/sudoers`来指定保留的环境变量, 如代理:

```
# This preserves proxy settings from user environments of root
# equivalent users (group sudo)
Defaults:%sudo env_keep += "http_proxy https_proxy ftp_proxy all_proxy no_proxy"
```

有时普通用户(非root)使用`ping`时会出现以下错误:

```
ping: socktype: SOCK_RAW
ping: socket: Operation not permitted
ping: => missing cap_net_raw+p capability or setuid?
```

参考 https://www.reddit.com/r/openSUSE/comments/14neo41/pinging_from_a_base_user_wsl_tumbleweed/

```
sudo setcap cap_net_raw+ep /bin/ping
```

字符显示问题:

```
# 查看当前系统的locale
locale
# 在/etc/locale.gen中取消注释需要的字符集
locale-gen
# 切换字符集(在/etc/locale.conf中写入)
LANG=en_US.UTF-8
```

sshd启动问题:

```
sshd: no hostkeys available -- exiting.
```

这是因为没有生成key

```
ssh-keygen -A
```

利用跳板简洁的ssh连接到docker, 修改`~/.ssh/config`:

```
Host docker
  HostName 172.25.84.146
# host ssh
  ProxyJump cabbaken@outlook.com@host_ip/domain
  User cabbaken
# docker sshd port
  Port 2222
```

‍
