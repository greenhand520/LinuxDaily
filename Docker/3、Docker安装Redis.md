### 拉取镜像

```shell
docker pull redis
```

或者使用registry.docker-cn.com加速

```shell
docker pull registry.docker-cn.com/library/redis:5.0.5
```

### 查看镜像

```shell
docker images
# 结果
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mysql               5.7                 2836a03e922f        42 hours ago        448MB
redis               latest              bd571e6529f3        8 days ago          104MB
```

### 运行

```shell
docker run -d -p 6379:6379 redis
```

