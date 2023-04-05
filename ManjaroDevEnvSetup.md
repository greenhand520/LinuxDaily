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
alter user 'root'@'localhost' identified by 'rootpassword';
flush privileges;
quit;
```

配置其他IP可访问

```sql
# 更新域属性，’%’表示允许外部访问：
update user set host='%' where user ='root';
# 再执行授权语句
grant all privileges on *.* to 'root'@'%'with grant option;
# 执行以上语句之后再执行
flush privileges;
```

开机自启

```shell
sudo systemctl enable mysqld
sudo systemctl restart mysqld
sudo systemctl status mysqld
# 输出
● mysqld.service - MySQL Server
     Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; preset: disabled)
     Active: active (running) since Wed 2023-04-05 18:19:09 CST; 27min ago
       Docs: man:mysqld(8)
             http://dev.mysql.com/doc/refman/en/using-systemd.html
   Main PID: 1685 (mysqld)
     Status: "Server is operational"
      Tasks: 42 (limit: 18901)
     Memory: 618.2M
        CPU: 32.208s
     CGroup: /system.slice/mysqld.service
             └─1685 /usr/bin/mysqld

Apr 05 18:19:08 host systemd[1]: Starting MySQL Server...
Apr 05 18:19:09 host systemd[1]: Started MySQL Server.
```

参考：

https://wiki.archlinuxcn.org/wiki/MySQL

```shell
sudo chown -R mysql:mysql /usr/local/mysql
```

## Redis

```shell
yay -S redis
```

配置开机自启动及其他IP可访问

```shell
sudo nano /etc/redis/redis.conf
#注释掉下面的IP绑定，开启守护进程启动，添加密码
## bind 127.0.0.1 -::1
requirepass password
daemonize yes

# 保存退出后设置开机自启
sudo systemctl enable redis.service
sudo systemctl start redis,service

# 查看运行状态
sudo systemctl status redis.service
● redis.service - Advanced key-value store
     Loaded: loaded (/usr/lib/systemd/system/redis.service; enabled; preset: disabled)
     Active: active (running) since Wed 2023-04-05 18:00:54 CST; 10s ago
   Main PID: 1257 (redis-server)
     Status: "Ready to accept connections"
      Tasks: 5 (limit: 18901)
     Memory: 7.6M
        CPU: 58ms
     CGroup: /system.slice/redis.service
             └─1257 "/usr/bin/redis-server *:6379"
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

## Nacos

官网：https://nacos.io/zh-cn/docs/quick-start.html

release下载地址：https://github.com/alibaba/nacos/releases

下载后解压到目录，单机启动

```shell
sh startup.sh -m standalone
```

单机启动配置成开启自启

```shell
sudo nano /etc/systemd/system/nacoss.service
```

添加如下内容

```properties
[Unit]
Description=nacos-standalone
After=network.target

[Service]
Type=forking
ExecStart=/home/${用户名}/Apps/nacos/bin/startup.sh -m standalone
ExecReload=/home/${用户名}/Apps/nacos/bin/shutdown.sh
ExecStop=/home/${用户名}/Apps/nacos/bin/shutdown.sh
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

修改nacos配置

```shell
nano /home/${用户名}/Apps/nacos/conf/aplication.properties
```

取消注释这几行

```properties
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect>
db.user.0=root
db.password.0=root
# 设置密钥，这里使用文档上的默认值
nacos.core.auth.plugin.nacos.token.secret.key=VGhpc0lzTXlDdXN0b21TZWNyZXRLZXkwMTIzNDU2Nzg=
```

关于2.x的鉴权密钥，参考:

 [2.x文档](https://nacos.io/zh-cn/docs/v2/quickstart/quick-start.html)

[关于Nacos默认token.secret.key及server.identity风险说明及解决方案公告](https://nacos.io/zh-cn/blog/announcement-token-secret-key.html)

启动nacos


```shell
sudo systemctl daemon-reload  
sudo systemctl start nacoss.service
sudo systemctl status nacoss.service

# 输出
● nacoss.service - nacos-standalone
     Loaded: loaded (/etc/systemd/system/nacoss.service; disabled; preset: disabled)
     Active: active (running) since Wed 2023-04-05 19:14:43 CST; 5s ago
    Process: 2457 ExecStart=/home/${用户名}/Apps/nacos/bin/startup.sh -m standalone (code=exited, status=0/SUCCESS)
   Main PID: 2484 (java)
      Tasks: 31 (limit: 18901)
     Memory: 377.3M
        CPU: 15.823s
     CGroup: /system.slice/nacoss.service
             └─2484 /usr/lib/jvm/java-8-openjdk/bin/java -Djava.ext.dirs=/usr/lib/jvm/java-8-openjdk/jre/lib/ext:/usr/lib/jvm/>

Apr 05 19:14:43 host systemd[1]: Starting nacos-standalone...
Apr 05 19:14:43 host startup.sh[2457]: /usr/lib/jvm/java-8-openjdk/bin/java -Djava.ext.dirs=/usr/lib/jvm/java-8-openjdk>
Apr 05 19:14:43 host startup.sh[2457]: nacos is starting with standalone
Apr 05 19:14:43 host startup.sh[2457]: nacos is starting，you can check the /home/${用户名}/Apps/nacos/logs/start.out
Apr 05 19:14:43 host systemd[1]: Started nacos-standalone.
lines 1-16/16 (END)

# 输出启动成功，启用开机自启
sudo systemctl enable nacoss.service
```

[通过网页](http://127.0.0.1:8848/nacos)访问，如果可以打开则运行成功，否则查看日志文件，查找报错原因。

## Sentinel-dashboard

Sentinel文档：https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D

github：https://github.com/alibaba/Sentinel

Dashboard文档：https://sentinelguard.io/zh-cn/docs/dashboard.html

下载jar包复制到安装路径，新建日志文件`sentinel_temp.log`，日志文件夹`logs`，在jar包同路径新建文件`dashboard-startup.sh`，添加以下内容

```shell
#!/bin/bash
 
RUN_NAME="sentinel-dashboard.jar"
JAVA_OPTS=/home/${用户名}/Apps/Sentinel/sentinel-dashboard.jar
 
LOG_DIR=/home/${用户名}/Apps/Sentinel/logs
LOG_FILE=$LOG_DIR/sentinel-dashboard.log
LOG_OPTS=/home/${用户名}/Apps/Sentinel/sentinel_temp.log
 
start() {
        nohup java -Xms256M -Xmx512M -XX:PermSize=128M -XX:MaxPermSize=256M -Dcsp.sentinel.log.dir=$LOG_DIR -Dlogging.file=$LOG_FILE -Dserver.port=8718  -Dcsp.sentinel.dashboard.server=127.0.0.1:8718 -Dproject.name=sentinel-dashboard -jar $JAVA_OPTS >$LOG_OPTS 2>&1 &
        echo "$RUN_NAME started success."
}
 
stop() {
        echo "stopping $RUN_NAME ..."
        kill -9 `ps -ef|grep $JAVA_OPTS|grep -v grep|grep -v stop|awk '{print $2}'`
}
 
case "$1" in
        start)
            start
            ;;
        stop)
            stop
            ;;
        restart)
            stop
            start
            ;;
        *)
                echo "Userage: $0 {start|stop|restart}"
                exit 1
esac
```

添加服务设置开机自启

```shell
sudo nano /etc/systemd/system/sentinel-dashboard.service
```

添如下内容

```properties
[Unit]
Description=sentinel-dashboard
After=network.target

[Service]
Type=forking
ExecStart=/home/${用户名}/Apps/Sentinel/dashboard-startup.sh start
ExecReload=/home/${用户名}/Apps/Sentinel/dashboard-startup.sh restart
ExecStop=/home/${用户名}/Apps/Sentinel/dashboard-startup.sh stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

启动dashboard

```shell
sudo systemctl daemon-reload  
sudo systemctl start sentinel-dashboard.service 
sudo systemctl status sentinel-dashboard.service 
# 输出
● sentinel-dashboard.service - sentinel-dashboard
     Loaded: loaded (/etc/systemd/system/sentinel-dashboard.service; disabled; preset: disabled)
     Active: active (running) since Wed 2023-04-05 20:38:48 CST; 2s ago
    Process: 4770 ExecStart=/home/${用户名}/Apps/Sentinel/dashboard-startup.sh start (code=exited, status=0/SUCCESS)
   Main PID: 4771 (java)
      Tasks: 17 (limit: 18901)
     Memory: 212.7M
        CPU: 7.058s
     CGroup: /system.slice/sentinel-dashboard.service
             └─4771 java -Xms256M -Xmx512M -XX:PermSize=128M -XX:MaxPermSize=256M -Dcsp.sentinel.log.dir=/home/${用户名}/Apps/Sentinel/logs -Dlogging.file=/home/${用户名}/Apps/Sentinel/logs/sentinel-dashboard.log -Dserver.port=8718 -Dcsp.sentinel.dashboard.se>

Apr 05 20:38:48 host systemd[1]: Starting sentinel-dashboard...
Apr 05 20:38:48 host dashboard-startup.sh[4770]: sentinel-dashboard.jar started success.
Apr 05 20:38:48 host systemd[1]: Started sentinel-dashboard.

# 输出SUCCESS，可以通过IP:PORT访问的话，添加自启
sudo systemctl enable sentinel-dashboard.service 
```

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
nano ～/.pip/pip.conf
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

参考：https://wiki.archlinuxcn.org/wiki/MongoDB

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

## MinIO

参考：http://docs.minio.org.cn/minio/baremetal/index.html

**安装**

```shell
yay -S minio minio-client
# 设置minio文件存储路径 注意不要在NTFS文件系统上设置,我设置后运行会报错
mkdir /home/${用户名}/minio
chmod -R 775 /home/${用户名}/minio
# 设置环境变量
nano ~/.zshrc
# 添加以下内容
export MINIO_ROOT_USER=minioroot
export MINIO_ROOT_PASSWORD=miniorootpwd
export MINIO_KMS_SECRET_KEY=my-minio-encryption-key:bXltaW5pb2VuY3J5cHRpb25rZXljaGFuZ2VtZTEyMwo=
# 运行inio
minio server /home/${用户名}/minio
```

运行输出类似于以下内容：

> ```
> API: http://127.0.0.1:9000
> RootUser: minioadmin
> RootPass: minioadmin
> Region:   us-east-1
> Console: http://127.0.0.1:64518
> RootUser: minioadmin
> RootPass: minioadmin
> Command-line: http://docs.minio.org.cn/minio/baremetal/minio-client-quickstart-guide
>    $ mc alias set myminio http://127.0.0.1:9000 minioadmin minioadmin
> Documentation: http://docs.minio.org.cn
> ```

| [`MINIO_ROOT_USER`](http://docs.minio.org.cn/minio/baremetal/reference/minio-server/minio-server.html#envvar.MINIO_ROOT_USER) | The [root user](http://docs.minio.org.cn/minio/baremetal/security/minio-identity-management/user-management.html#minio-users-root) 访问钥匙.更换具有长、随机和唯一字符串的样本值。 |
| :----------------------------------------------------------- | ------------------------------------------------------------ |
| [`MINIO_ROOT_PASSWORD`](http://docs.minio.org.cn/minio/baremetal/reference/minio-server/minio-server.html#envvar.MINIO_ROOT_PASSWORD) | The [root user](http://docs.minio.org.cn/minio/baremetal/security/minio-identity-management/user-management.html#minio-users-root) 密钥。 更换具有长、随机和唯一字符串的样本值。 |
| [`MINIO_KMS_SECRET_KEY`](http://docs.minio.org.cn/minio/baremetal/reference/minio-server/minio-server.html#envvar.MINIO_KMS_SECRET_KEY) | MinIO IAM 后端的加密密钥。 更换具有 32 位 base-64 编码值的样本值。 例如， 使用以下命令生成随机密钥:`cat /dev/urandom | head -c 32 | base64 -` |

打开 MinIO 控制台: 打开浏览器和 [http://127.0.0.1:9000](http://127.0.0.1:9000/) 打开 MinIO 控制台 登录页面。使用[`MINIO_ROOT_USER`](http://docs.minio.org.cn/minio/baremetal/reference/minio-server/minio-server.html#envvar.MINIO_ROOT_USER) 和 [`MINIO_ROOT_PASSWORD`](http://docs.minio.org.cn/minio/baremetal/reference/minio-server/minio-server.html#envvar.MINIO_ROOT_PASSWORD) 登录。

**添加开机自启**

修改配置

```shell
sudo nano /etc/minio/minio.conf
# 修改存储卷
MINIO_VOLUMES="/mnt/ServerLocal/minio/data/"
```

设置自启

```shell
sudo systemctl start minio.service
sudo systemctl status minio.service
# 输出
● minio.service - Minio
     Loaded: loaded (/usr/lib/systemd/system/minio.service; disabled; preset: disabled)
     Active: active (running) since Wed 2023-04-05 22:52:50 CST; 6s ago
       Docs: https://docs.minio.io
    Process: 5408 ExecStartPre=/bin/bash -c { [ -z "${MINIO_VOLUMES}" ] && echo "Variable MINIO_VOLUMES not set in /etc>
   Main PID: 5409 (minio)
      Tasks: 10 (limit: 18899)
     Memory: 104.9M
        CPU: 591ms
     CGroup: /system.slice/minio.service
             └─5409 /usr/bin/minio server /srv/minio/data/

Apr 05 22:52:50 host minio[5409]: Copyright: 2015-0000 MinIO, Inc.
Apr 05 22:52:50 host minio[5409]: License: GNU AGPLv3 <https://www.gnu.org/licenses/agpl-3.0.html>
Apr 05 22:52:50 host minio[5409]: Version: RELEASE.2023-02-10T18-48-39Z (go1.20 linux/amd64)
Apr 05 22:52:50 host minio[5409]: Status:         1 Online, 0 Offline.
Apr 05 22:52:50 host minio[5409]: API: http://192.168.31.239:9000  http://127.0.0.1:9000
Apr 05 22:52:50 host minio[5409]: Console: http://192.168.31.239:37953 http://127.0.0.1:37953
Apr 05 22:52:50 host minio[5409]: Documentation: https://min.io/docs/minio/linux/index.html
Apr 05 22:52:50 host minio[5409]: Warning: The standard parity is set to 0. This can lead to data loss.
Apr 05 22:52:52 host minio[5409]:  You are running an older version of MinIO released 1 month ago
Apr 05 22:52:52 host minio[5409]:  Update: Run `mc admin update`

# 通过连接192.168.31.239:9000访问，同户名和密码都是 minioadmin，访问正常后，添加自启
sudo systemctl enable minio.service
```

