### Redis单机部署多实例

#### 安装Redis

略。

#### 修改配置

1. 创建数据存放目录。

我们根据Redis的端口分别创建6380、6381、6382三个目录，然后在实例目录下分别创建conf(配置文件)、db(持久化)、log(日志)目录。

2. 修改配置。

分别修改不同实例的配置信息，修改内容大致如下：

```properties
#以后台运行
daemonize yes
#pid目录，以后台模式运行，需要创建该目录
pidfile /Users/user/Documents/ProgramFile/redis-data/6380/redis_6380.pid
#端口
port 6380
# 日志存放目录
logfile /Users/user/Documents/ProgramFile/redis-data/6380/log/redis.log
#持久化目录
dir /Users/user/Documents/ProgramFile/redis-data/6380/db/
```

3. 以配置文件的方式启动Redis





