## 简介

CentOS 8 Linux 将不再支持并在 2021 年底停止使用。取而代之的是滚动版本 CentOS Stream 作为 RHEL 的下游分支于 2019 年推出，将持续排查漏洞，让上游版本更加稳定和安全。但是，运行 CentOS 7 的服务器不会受到影响。他们将与 RHEL 7 生命周期并行更新。RHEL 7 将在 2024 年结束其最后一个维护周期。

CentOS7 X86_64下载[地址](http://isoredirect.centos.org/centos/7/isos/x86_64/), [点击](http://iso.mirrors.ustc.edu.cn/centos/7.9.2009/isos/x86_64/CentOS-7-x86_64-Minimal-2207-02.iso)直接下载 2207-02 版.

CentOS 7: https://www.centos.org/centos-linux/

CentOS Stream: https://www.centos.org/centos-stream/

## 安装

CentOS7的[仓库](http://isoredirect.centos.org/centos/7/isos/x86_64/)：

```
http://mirrors.163.com/centos/7.9.2009/isos/x86_64/
http://ftp.sjtu.edu.cn/centos/7.9.2009/isos/x86_64/
http://mirrors.bfsu.edu.cn/centos/7.9.2009/isos/x86_64/
http://mirrors.qlu.edu.cn/centos/7.9.2009/isos/x86_64/
http://mirrors.ustc.edu.cn/centos/7.9.2009/isos/x86_64/
http://mirrors.bupt.edu.cn/centos/7.9.2009/isos/x86_64/
http://mirrors.tuna.tsinghua.edu.cn/centos/7.9.2009/isos/x86_64/
http://mirrors.nju.edu.cn/centos/7.9.2009/isos/x86_64/
http://mirror.lzu.edu.cn/centos/7.9.2009/isos/x86_64/
http://mirrors.cqu.edu.cn/CentOS/7.9.2009/isos/x86_64/
http://mirrors.jlu.edu.cn/centos/7.9.2009/isos/x86_64/
http://mirrors.aliyun.com/centos/7.9.2009/isos/x86_64/
```

## 软件

### Samba 共享

```shell
sudo yum -y install samba nano
```

修改配置文件

```shell
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
sudo nano /etc/samba/smb.conf
```

添加配置，这里任何用户都可以访问且拥有文件是所有权限

```properties
[share]
       	comment = Home Directories
        path = /home/mdmbct
        public = yes
        browseable = yes
        writable = yes
        read only = No
        create mask = 0700
        directory mask = 0700
```

通过命令

```shell
testparm
```

检查`sam.conf`是否有语法错误。

将系统用户添加到samba用户组，并设置密码

```shell
sudo smbpasswd -a ${用户名}
```

重启smb，nmb服务

```shell
sudo systemctl restart smb nmb
sudo systemctl enable smb nmb
```

如果重启失败，则执行testparm或者通过`status`查看原因。

查看所有的samba用户

```shell
sudo pdbedit -L
```

此时需要关闭防火墙才能访问

```shell
sudo systemctl stop firewalld
```

禁止防火墙开机启动：

```shell
sudo systemctl disable firewalld
```

下面的操作貌似可以不用操作

最后要关闭selinux：

```shell
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
## 查看是否设置为disable
cat /etc/selinux/config | grep SELINUX
## 显示内容如下：
# SELINUX= can take one of these three values:
# SELINUX=disabled
SELINUX = disabled
# SELINUXTYPE= can take one of three values:
SELINUXTYPE=targeted 
```

### MySQL

