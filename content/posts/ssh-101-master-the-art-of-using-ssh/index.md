---
keywords:
- cloud native
- cloud native 101
- krew
- ksniff
- kubectx
title: "SSH 101: 一文掌握使用 SSH 的正确姿势"
subtitle: "一文告诉你什么是使用 SSH 的正确姿势！"
description: 本文详细介绍了 SSH 免密登录远程服务器的多种方法，包括 sshpass、RSA 密钥对、SSH Config 文件配置等，适合追求高效安全远程管理的用户阅读。
date: 2021-10-30T20:18:00+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- ssh
- 工程效率
---


![cover](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/ssh-101/c78494f4-56a4-4dc7-b0b1-10acfcc35550.png)

日常工作中你是否经常需要使用 `SSH` 登录远程服务器，平时你只用到一两台服务器还好，如果量上去了，你是否有为管理多台服务器苦恼过？

需要记住那么多的 `IP` 地址、用户名、以及不同非标准的默认端口，还要输入繁琐的口令密码，太难了。

我相信大家也不愿意去记这玩意儿，纯属浪费时间，而且这确实也不是一种非常有效率的方式，对于一个有追求的`新生代农民工`来说是不能忍的。

这个问题其实有很多种方法去解决，在我们团队内部，我发现大家使用的方法也各有不同。

## SSH

我们先来看下维基百科对 `SSH` 的介绍。

`Secure Shell`（安全外壳协议，简称 `SSH`）是一种加密的网络传输协议，可在不安全的网络中为网络服务提供安全的传输环境。`SSH` 通过在网络中创建安全隧道来实现 `SSH` 客户端与服务器之间的连接。`SSH` 最常见的用途是远程登录系统，人们通常利用 `SSH` 来传输命令行界面和远程执行命令。

> 摘自 [Secure Shell](https://zh.wikipedia.org/wiki/Secure_Shell "Secure Shell")

```bash
    ssh 客户端(ssh)                    ssh 服务端(sshd)

     +----------+                       +---------+
     |          |                       |         |
     |          |                       |         |
     |          |                       |         |
     |          |                       |         |
     +---+---+--+                       +-+---+---+
         |   |                            |   |
         |   |                            |   |
         |   +----------------------------+   |
         |                                    |
         |        ssh 加密了的 TCP 通信         |
         +------------------------------------+
```

## Non-Interactive SSH Login

初次使用 `SSH` 登录远程服务器的同学一般很老实的采用交互式的方式，通过输入密码登录，操作久了难免也会乏味。那么有没有什么工具可以让你摆脱这种重复的流程呢？有的，它就是 `sshpass`。

`sshpass` 它是一个非常优秀的非交互式 `SSH` 登录工具，使用起来非常的简单，我先来看几个常用命令。

```bash
# 使用默认端口
sshpass -p '<PASSWORD>' ssh <USER_NAME>@<HOST_NAME>

# 使用非标准端口
sshpass -p '<PASSWORD>' ssh -p 8000 <USER_NAME>@<HOST_NAME>

# 密码作为环境变量 `SSHPASS` 传递
SSHPASS='<PASSWORD>' sshpass -e ssh <USER_NAME>@<HOST_NAME>

# 其他使用方法，可以通过 help 自行摸索
sshpass -h
```

聪明如你，通过这个工具我们其实就有很多发挥的空间了，比如扩展 `Bash Aliases` 或者通过 `Shell Script` 做一些自动化的东西。

### Bash Aliases

你可以在你的 `~/.aliases` 配置文件里追加以下配置

```bash
# Login devbox
alias login_devbox="sshpass -p '<PASSWORD>' ssh <USER_NAME>@<HOST_NAME>"
```

然后你就可以通过更简短的命令来登录了，我们来验证下

```bash
➜ login_devbox

Last login: Sat Oct 30 12:51:16 2021 from 10.0.3.xxx
[root@vm-ds2-srv26-42 ~]#
```

### Shell Script

我们来看个简单例子，用 `sshpass` 在 `Shell script` 里能做什么事情。

```bash
SSHPASS='<PASSWORD>' sshpass -e ssh <USER_NAME>@<HOST_NAME> df -h
```

效果如下

```bash
➜ SSHPASS='tx#M58bFKQ' sshpass -e ssh root@172.18.0.xx df -h

Filesystem      Size  Used Avail Use% Mounted on
devtmpfs         15G     0   15G   0% /dev
tmpfs            15G     0   15G   0% /dev/shm
tmpfs            15G  1.5G   14G  10% /run
tmpfs            15G     0   15G   0% /sys/fs/cgroup
/dev/vda5       126G  7.6G  119G   6% /
/dev/vda2        15G   33M   15G   1% /var/lib/mysql
/dev/vda1      1014M  195M  820M  20% /boot
/dev/vdc        1.9T  144G  1.7T   8% /minio02
/dev/vdb        1.9T  144G  1.7T   8% /minio01
tmpfs           3.0G     0  3.0G   0% /run/user/0
```

> 但是需要注意的是 **永远不要在生产服务器上使用**，建议只在本地开发机器上使用，解脱重复乏味的操作即可，原因相信大家都懂的。

> 如何安装 `sshpass`，可以参考以下链接
>
> [Installing SSHPASS](https://gist.github.com/arunoda/7790979 "Installing SSHPASS")

## SSH Passwordless Login

`SSH` 通过密码方式认证，难免会产生一些不必要的麻烦，那么还有没其他更简单的登录方式，而且又能防止信息泄露呢？当然有，它就是 `Public-key Authentication`，一种相对安全的模式，且能做到免密码登录远程服务器。

### Step 1. 在客户端生成 RSA 密钥对

我们用 `ssh-keygen` 命令来生成 `rsa key pair`，以下是生成过程。

```bash
➜ ssh-keygen -t rsa -C "lqshow@gmail.com" -f ~/.ssh/example-rsa -b 4096

Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
```

对私钥进行保护的口令密码我们可以留空，直接按 `Enter`。

```bash
➜ ssh-keygen -t rsa -C "lqshow@gmail.com" -f ~/.ssh/example-rsa -b 4096
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /Users/linqiong/.ssh/example-rsa
Your public key has been saved in /Users/linqiong/.ssh/example-rsa.pub
The key fingerprint is:
SHA256:Z2/4RbsSZeYlYYpgkiG42w4SJZZI9decFouFMyPrB7k lqshow@gmail.com
The key's randomart image is:
+---[RSA 4096]----+
|o.oo. .o.o       |
|ooo .oo=B +   o  |
|.o . .+=+B . o . |
|. .  +. . . . = .|
| . o. o S o  =.o |
|. o .E . o o.... |
| . o  .   . o.o  |
|    .      o.. . |
|            ...  |
+----[SHA256]-----+
```

成功生成后，我们来查看下刚刚生成的一对 `RSA` 密钥

> 需要特别注意的是 `私钥` 是不能对外泄露的

```bash
➜ ls -lha ~/.ssh |grep example
-rw-------    1 linqiong  staff   3.3K Oct 30 14:45 example-rsa      #私钥
-rw-r--r--    1 linqiong  staff   742B Oct 30 14:45 example-rsa.pub  #公钥
```

当然，你也可以给不同的场景配置不同的 `rsa key pair`，如下所示

```bash
# company
ssh-keygen -t rsa -C "qiong.lin@basebit.ai"

# github
ssh-keygen -t rsa -C "lqshow@gmail.com" -f ~/.ssh/github-rsa -b 4096
```

### Step 2. 拷贝公钥到远程服务器

通过 `ssh-copy-id` 将 `Step 1` 生成的客户端公钥追加到远程机器的 `authorized_keys` 文件中（需要输入密码）

> 这里说明下，密钥只针对本地生成的用户有效，其他用户需要自己生成，并做同样的操作

```bash
# 拷贝公钥文件
ssh-copy-id -i ~/.ssh/example-rsa.pub root@172.18.0.xxx
```

### Step 3. 验证免密登录

现在通过使用客户端私钥来登录，我们来验证下是否生效了

```bash
➜ ssh root@172.18.0.xxx -i ~/.ssh/example-rsa

Last login: Sat Oct 30 15:35:38 2021 from 10.0.3.xxx
[root@vm-ds2-srv26-42 ~]#
```

同时使用 `scp` 命令，验证下上传动作

```bash
➜ scp -i ~/.ssh/example-rsa ~/Downloads/ssh.png root@172.18.0.xxx:/home/qiong.lin
ssh.png                                         100%  976KB 234.3KB/s   00:04
```

## SSH Config File

大家肯定也发现了，免密登录的方式虽然比 `sshpass xxx` 简化了很多，不过还是比较麻烦的，要记住用户名的同时又要记住具体服务器的 `ip` 地址，我们还有没有更方便的方式呢？

我们可以通过对 `~/.ssh/config` 文件的配置，简化 `SSH` 相关的操作。

### SSH Config File Example

需在客户端的 `~/.ssh` 目录下新建名称为 `config` 的文件，配置多个不同的 `host` 可以使用不同的 `ssh key`。

我们先来看一个配置文件，如下所示

```bash
Host login_devbox                         # 目标服务器别名
    HostName 172.18.0.xxx                 # 主机地址
    User root                             # 用户名
    Port 22                               # 指定端口
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/example-rsa       # 私钥文件路径
```

现在登录方式变的更加简洁，完全不需要再去记 `IP` 地址、用户名、以及不同非标准的端口，也不需要在指定 `-i` 参数了，只需用自定义的服务器别名登录即可。

```bash
➜ ssh login_devbox

Last login: Sat Oct 30 16:59:25 2021 from 10.0.3.xxx
[root@vm-ds2-srv26-42 ~]#
```

我们在来看下文件上传命令是否也更简单了

```bash
➜ scp ~/Downloads/ssh.png login_devbox:/home/qiong.lin
ssh.png                   100%  976KB 234.3KB/s   00:04
```

同时我们也可以利用 `ssh` 直接来验证上传的文件

```bash
➜ ssh login_devbox "ls -lh /home/qiong.lin"

total 980K
-rw-r--r-- 1 root root 977K Oct 30 17:05 ssh.png
```

### 小技巧

对于一些有规律的主机配置，我们其实不需要为每台机器设置一个配置，比如说一个 `Kubernetes` 集群的节点，如下所示:

```bash
➜ kubectl get node -owide
NAME         STATUS   ROLES    AGE    VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION              CONTAINER-RUNTIME
kind-dev1     Ready    master   219d   v1.19.2   172.18.0.135   <none>        CentOS Linux 7 (Core)   5.7.7-1.el7.elrepo.x86_64   docker://20.10.5
kind-dev2     Ready    master   219d   v1.19.2   172.18.0.136   <none>        CentOS Linux 7 (Core)   5.7.7-1.el7.elrepo.x86_64   docker://20.10.5
kind-dev3     Ready    master   219d   v1.19.2   172.18.0.137   <none>        CentOS Linux 7 (Core)   5.7.7-1.el7.elrepo.x86_64   docker://20.10.5
kind-dev4     Ready    <none>   219d   v1.19.2   172.18.0.138   <none>        CentOS Linux 7 (Core)   5.7.7-1.el7.elrepo.x86_64   docker://20.10.5
```

按照惯性思维，如果这`四台机器`你都要维护，你是不是准备要设置`四份配置`，反正很简单嘛，复制粘贴改一下就行嘛，又不是不能用？

其实有更简单的办法，看一下以下配置，通过一些参数来完成。

```bash
Host 1*
  HostName 172.18.%h
  User qiong.lin
  IdentityFile ~/.ssh/id_rsa
  Port 22
```

配置完后，我们来看下效果吧。

```bash
# 登录 `172.18.0.135` 节点
➜ ssh 135

Last login: Sat Oct 30 18:14:14 2021 from 10.0.3.42
[qiong.lin@kind-dev1 ~]$ ifconfig|grep 135
        inet 172.18.0.135  netmask 255.255.255.0  broadcast 172.18.0.255
```

```bash
# 登录 `172.18.0.138` 节点
➜ ssh 138

Last login: Sat Oct 30 18:14:41 2021 from 10.0.3.42
[qiong.lin@kind-dev4 ~]$ ifconfig|grep 138
        inet 172.18.0.138  netmask 255.255.255.0  broadcast 172.18.0.255
```

### 连接 github

> 将本地公钥添加到 github 的 SSH keys 中

```bash
# 验证登录 github
➜ ssh -T git@github.com
Hi lqshow! You've successfully authenticated, but GitHub does not provide shell access.
```

## Conclusion

通过这篇文章的介绍，相信大家都知道 `SSH` 支持两种形式的登录认证。

1. `Password Authentication`
2. `Public-key Authentication`

文中提到过的两种 `SSH` 免密登录，对我们日常工作中是非常有用的，特别是在做一些自动化任务(比如使用脚本自动备份、使用 `SCP` 命令同步文件和远程命令执行等)，且该模式安全性是非常高的。

其实 `SSH Config` 不仅能帮助我们快速的使用 `SSH` 登录不同的主机，同时还可以管理多个 `SSH key pairs`。

希望这篇文章对大家有所帮助。
