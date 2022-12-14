## MySQL
#MySQL 

参考：[MySQL ArcWiki](https://wiki.archlinux.org/title/MySQL_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

### mariadb

安装

```shell
yay -S mariadb
```

初始化

```shell
sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
sudo systemctl enable mysql
sudo systemctl start mysql
# 修改密码
sudo mysql_secure_installation
```

### percona-server

安装

```shell
yay -S percona-server percona-server-clients
```

初始化

安装完直接启动是启动不了的，因为还没有初始化

```shell
# 清空数据数据目录的多余文件，如果不清空，会导致初始化失败，默认目录是/var/lib/mysql/
sudo rm -R /var/lib/mysql
sudo mysqld --initialize --user=root
# 初始化后，可以在日志文件里面找到初始化的密码
sudo grep -i password /var/log/mysqld.log 
# 给日志文件赋予权限
sudo chown mysql:mysql /var/log/mysqld.log
# 启动mysq,修改密码
sudo systemctl start mysqld
mysql -u root -p
alert user root@localhost identified by “rootpassword”;
```

## JDK

#JDK #JAVA

### **jdk8**

```shell
yay -S jdk8
# or
yay -S jdk8-openjdk
```

### **jdk11**

```shell
yay -S jdk11-openjdk
```

## Jetbrains

### fcitx 输入法光标不跟随

IDEA 的 JRE 有问题,下载重新编译的 [JRE](https://github.com/RikudouPatrickstar/JetBrainsRuntime-for-Linux-x64/releases/download/2022-04-15_00-02/jbr-linux-x64-2022-04-15_00-02.zip) Linux x64 版本

解压出 `jbr` 文件夹，备份 IDEA 目录下的 `jbr` 文件夹，然后把上面链接下载解压后的 `jbr` 文件夹替换。这里可使用 Linux 的软链接。

参考：https://github.com/RikudouPatrickstar/JetBrainsRuntime-for-Linux-x64

## 网络抓包

### charles

```shell
yay -S charles-bin
```

导入证书：

1、下载Charles的SSL根证书：`Help -> SSL Proxying -> Install Charles Root Certificate`，安装后，应该能在home目录的`.charles/ca/`中找到`charles-proxy-ssl-proxying-certificate.cer`和`charles-proxy-ssl-proxying-certificate.pem`两个文件。

2、转换格式：终端中执行

```shell
openssl x509 -outform der -in charles-proxy-ssl-proxying-certificate.pem -out charles.crt
```

3、在`/usr/share/ca-certificates`文件夹下新建一个目录`charles`，再将转换格式后得到的crt证书复制到`/usr/share/ca-certificates/charles` 中

```shell
sudo mkdir /usr/share/ca-certificates/charles
sudo cp charles.crt /etc/ca-certificates/trust-source/charles
sudo trust extract-compat
```

启动报错

> Error: A JNI error has occurred, please check your installation and try again
> Exception in thread "main" java.lang.UnsupportedClassVersionError: com/xk72/charles/gui/MainWithClassLoader has been compiled by a more recent version of the Java Runtime (class file version 55.0), this version of the Java Runtime only recognizes class file versions up to 52.0

java版本号对应关系如下

> 49 = Java 5
> 50 = Java 6
> 51 = Java 7
> 52 = Java 8
> 53 = Java 9
> 54 = Java 10
> 55 = Java 11
> 56 = Java 12
> 57 = Java 13
> 58 = Java 14

所以意思是需要 JDK11 但是默认 Java Runtime 是 JDK8，安装 JDK11 并修改 `/usr/bin/charles` 文件

```shell
yay -S jdk11-openjdk
# yay -S jdk11 下载不了
sudo nano /usr/bin/charles
```

修改如下

27 行添加 `export JAVA_HOME="/usr/lib/jvm/java-11-openjdk"`

```bash
# Check if we have the included JDK
if [ -d "$CHARLES_LIB/jdk" ]; then
    export JAVA_HOME="$CHARLES_LIB/jdk"
fi

# 添加内容
export JAVA_HOME="/usr/lib/jvm/java-11-openjdk"

# Find Java binary
if [ -z "$JAVA_HOME" ]; then
    hash java 2>/dev/null || { echo >&2 "Charles couldn't start: java not found. Please install java to use Charles."; exit 1; }
    JAVA=java
else
    JAVA="$JAVA_HOME/bin/java"
fi
```

## Python

### pip3

默认没有安装pip3

```shell
sudo pacman -S python-pip
```

### pip 换源

```shell
mkdir ～/.pip
nano ～/。pip/pip.conf
```

pip.conf 内容为

```properties
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/
[install]
trusted-host = mirrors.aliyun.com
```

国内的一些 pip 镜像 

- 阿里云 <http://mirrors.aliyun.com/pypi/simple/> 
- 中国科技大学 <https://pypi.mirrors.ustc.edu.cn/simple/> 
- 豆瓣(douban) <http://pypi.douban.com/simple/> 
- 清华大学 <https://pypi.tuna.tsinghua.edu.cn/simple/> 
- 中国科学技术大学 <http://pypi.mirrors.ustc.edu.cn/simple/

## Nodejs

```shell
yay -S nodejs npm
```

npm镜像设置成淘宝镜像，加快下载速度

```shell
npm config set registry http://registry.npm.taobao.org/
```

## Scala & sbt

安装 Scala 的包管理及构建工具 sbt 即可。

```shell
yay -S sbt
```

第一次启动 sbt 非常慢，需要设置 sbt 国内镜像

```shell
cd ～/.sbt & touch repositories
# 添加如下内容：
[repositories]
local
aliyun: https://maven.aliyun.com/repository/public
typesafe: https://repo.typesafe.com/typesafe/ivy-releases/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext], bootOnly
ivy-sbt-plugin:https://dl.bintray.com/sbt/sbt-plugin-releases/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]
sonatype-oss-releases
maven-central
sonatype-oss-snapshots
```

Linux 需要修改配置让其适用于全局

```shell
nano /etc/sbt/sbtopts
# 最后添加如下内容：
-Dsbt.override.build.repos=true
```

然后在控制台启动 sbt 就不会卡很久了。

参考：https://blog.51cto.com/zhangxueliang/4953538

## MongoDB

```shell
yay -s mongodb-bin
```

启动 MongoDB 进程

```shell
systemctl start mongodb.service
```

**注意：** 在MongoDB服务第一次启动期间，它将通过创建大文件(用于其日志和其他数据)来[预分配空间](https://docs.mongodb.com/manual/faq/storage/#preallocated-data-files)。这一步可能需要一段时间，在此期间MongoDB Shell不可用。

关闭进程

```shell
systemctl stop mongodb.service
```

参考：https://wiki.archlinuxcn.org/wiki/MongoDB
