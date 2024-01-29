```shell
# 带management是有管理界面的
docker pull rabbitmq:3.8.9-management
# 运行 5672通信端口、15672管理界面端口
docker run -d -p 5672:5672 -p 15672:15672 --name rabbitmq3.8.9 rabbitmq:3.8.9-management
```

管理页面访问用户及密码默认都是guest

