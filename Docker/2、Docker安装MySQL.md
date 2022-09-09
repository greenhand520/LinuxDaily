### **1、拉取镜像**

在[DockerHub](https://hub.docker.com/)搜索MySQL，然后使用`docker pull mysql:tag`拉取。

其中`tag`是mysql版本，默认是latest，这里选择5.7版。

```shell
docker pull mysql:5.7
```

### **2、启动容器实例**

简单启动

`docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:tag`

-d 表示后台运行

```shell
docker run --name mysql5.7-00 -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
```

端口映射

`docker run -p 3306:对外端口 --name some-mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:tag`

```shell
docker run -p 3306:3306 --name mysql5.7-01 -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
```

3、使用自己的配置文件启动

复制下面内容到自己的配置文件my.cnf中，这里路径设置成/conf/mysql5.7-00

```properties
# The MariaDB configuration file
#
# The MariaDB/MySQL tools read configuration files in the following order:
# 1. "/etc/mysql/mariadb.cnf" (this file) to set global defaults,
# 2. "/etc/mysql/conf.d/*.cnf" to set global options.
# 3. "/etc/mysql/mariadb.conf.d/*.cnf" to set MariaDB-only options.
# 4. "~/.my.cnf" to set user-specific options.
#
# If the same option is defined multiple times, the last one will apply.
#
# One can use all long options that the program supports.
# Run program with --help to get a list of available options and with
# --print-defaults to see which it would actually understand and use.

#
# This group is read both both by the client and the server
# use it for options that affect everything
#
[client-server]

[mysqld]
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
```

创建 MySQL 容器，绑定容器外文件并启动

实现创建好`/conf/mysql5.7-00/data`文件夹

```shell
docker run --name mysql5.7-00 \
-p 3306:3306 -e MYSQL_ROOT_PASSWORD=root \
--mount type=bind,src=/conf/mysql5.7-00/my.cnf,dst=/etc/mysql/my.cnf \
--mount type=bind,src=/conf/mysql5.7-00/data,dst=/var/lib/mysql \
--restart=on-failure:3 \
-d mysql:5.7
```

-   **--name**：为容器指定一个名字
-   **-p**：指定端口映射，格式为：**主机(宿主)端口:容器端口**
-   **-e**：username="xxx"，设置环境变量
-   **--restart=on-failure:3**：是指容器在未来出现异常退出（退出码非0）的情况下循环重启3次
-   **-mount**：绑定挂载
-   **-d**：后台运行容器，并返回容器 id

`--mount type=bind,src=/conf/mysql5.7-00/data,dst=/var/lib/mysql \`把镜像数据库路径映射到本地的`/conf/mysql5.7-00/data`中。

查看mysql字符集

```shell
# 进入docker容器内
docker exec -it mysql5.7-00 bash
# 登录mysql
mysql -uroot -p
# 查看mysql字符集命令
show variables like '%char%';
# 结果
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | utf8                       |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | utf8                       |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.01 sec)
```

### 3、连接MySQL

```sql
use mysql;
select host from user where user = 'root';
# 放开IP访问权限
update user set host='%' where user = 'root'
# 刷新权限
flush privileges;
```

exit;退出mysql

exit退出docker



参考

[官方文档](https://hub.docker.com/_/mysql)

[Docker安装MySQL并挂载数据及配置文件 - Weiles - 博客园 (cnblogs.com)](https://www.cnblogs.com/weile0769/p/11863779.html)

[连接mysql时报：message from server: "Host '192.168.76.89' is not allowed to connect to this MySQL server 处理方案 - asdyzh - 博客园 (cnblogs.com)](https://www.cnblogs.com/asdyzh/p/9708153.html)

