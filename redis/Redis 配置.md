### Redis 配置

1. INCLUDES

不同redis server可以使用同一个模版配置作为主配置，并引用其他配置文件用于server的个性化配置

```properties
# include /path/to/local.conf
# include /path/to/other.conf
```

2. NETWORK

不指定bind，redis 会监听所有网络的连接。如果可能，监听一个或者多个指定的ip，如下：

```properties
bind 192.168.1.100 10.0.0.1
bind 127.0.0.1
```

redis 直接跑在公网上并且接收所有的网络请求连接是一件非常危险的事情，所以默认情况下，只能同一台机器上的客户端才能接入redis server。

3. daemonize

默认情况下，redis不会以一个后台的方式运行，如果需要，将它设置为yes。当它以后台的方式运行时，会将pid写入/var/run/redis.pid文件中。

```properties
daemonize no
```

4. protected-mode

是否开启保护模式，默认开启。要是配置里没有指定bind和密码，开启该参数后，redis只会本地进行访问，拒绝外部访问。因此开启了密码和bind，则可以开启，否则最好关闭。

```properties
protected-mode yes
```

5. port

在指定的端口接收连接，默认是6379。

6. tcp-backlog

这个参数确定了TCP连接已完成队列(完成三次握手之后)的长度，默认值是511。当然此值不能大于Linux系统定义的(/proc/sys/net/core/somaxconn)值，在Linux中，默认参数值为128。当系统并非量大并且客户端连接速度慢时，可以将这两个参数一起参考设定。

```properties
tcp-backlog 511
```

7. timeout

此参数为设置客户端空闲超过timeout，服务端会断开连接，为0则服务端不会主动断开连接，不能小于0.

```properties
timeout 0
```

8. tcp-keepalive

 如果设置不为0，表示将周期性的使用SO_KEEPALIVE检测客户端是否还处于健康状态，避免服务器一直阻塞，默认是300秒

```properties
cp-keepalive 300
```

9. pidfile

redis以后台进程方式运行redis，则需要指定pid文件

```
pidfile /var/run/redis_6379.pid
```

10. loglevel

服务端日志级别，级别包括debug(很多信息)、verbose(许多有用的信息，但是没有debug级别信息多)、notice(适当的日志级别，适合生产环境)、warn(只有非常重要的信息)

```properties
loglevel notice
```

11. logfile

指定了记录日志的文件。默认空字符串，日志会打印到标准输出设备。后台运行的redis标准输出是/dev/null。

```
logfile ""
```

12. syslog-enabled

是否打开记录syslog功能

```
syslog-enabled no
```

13. syslog-ident 

syslog的标识符。

```
syslog-ident redis
```

14. syslog-facility

日志的来源。

```
syslog-facility local0
```

15. databases

设置数据库的数量，默认使用的数据库是0。

```
databases 16
```

#### 快照配置

1. save

根据给定的时间间隔和写入次数将数据保存到磁盘

如下面的例子：

```properties
# 900 秒内如果至少一个key的值变化，则保存
save 900 1
# 300 秒内如果至少有10个key的值发生变化，则保存
save 300 10
# 60秒内如果至少有10000个key的值发生变化，则保存
save 60 10000
```

可以注释掉所有的save行来停用保存功能，也可以直接使用一个空字符串来实现停用。

```
save ""
```

2. stop-writes-on-bgsave-error

默认情况下，如果redis最后一次的后台保存失败，redis将停止接受写操作，这样以一种强硬的方式让用户知道不能正确的将数据持久化到磁盘。

如果安装了靠谱的监控，可能不希望redis这样做，改成no就好了。

```
stop-writes-on-bgsave-error yes
```

3. rdbcompression

是否在持久化数据的时候使用LZF压缩字符串，默认设置为yes。如果希望节省cpu，可以把它设置为no，不过这个数据集可能会比较大。

```
rdbcompression yes
```

4. rdbchecksum

是否校验rdb文件

```
rdbchecksum yes
```

5. dbfilename

设置dump的文件位置

```
dbfilename dump.rdb
```

如上，dbfilename 只定义了文件名称，所以我们必须还要指定一个目录

```
dir ./
```

#### 主从复制

1. slaveof

主从复制。使用slaveof来让一个redis实例成为另一个redis实例的副本。这个只需要在slave上配置。

```properties
# slaveof <masterip> <masterport>
```

2. masterauth

如果master 需要密码认证，就在这里设置。

```properties
# masterauth <master-password>
```

3. slave-serve-stale-data

当一个slave与master失去联系，或者复制正在进行时，slave可能会有两种表现：

1）如果为yes，slave仍然会应答客户端请求，但返回的数据可能是过时的，或者数据可能为空。

2）如果是no，在伱执行除了“INFO and SLAVEOF”之外的其他命令时，slave都将返回一个"SYNC with master in progress"的错误。

```
slave-serve-stale-data yes
```

4. slave-read-only

你可以配置一个slave实例是否接受写入操作，通过写入操作来存储一些短暂的数据对于一个slave实例来说可能是有用的，因为相对从master重新同步数据而言，直接写入到slave的数据会更容易被删除。

从redis 2.6起，默认slaves都是只读的。

```
slave-read-only yes
```

5. repl-ping-slave-period

Slaves在一个预定义的时间间隔内发送ping命令到server，你可以改变这个时间间隔。默认是10秒。

```
repl-ping-slave-period 10
```

6. repl-timeout

设置主从复制的过期时间，这个值一定要比repl-ping-slave-period大。

```
repl-timeout 60
```

7. repl-diskless-sync

是否使用socket方式复制数据。

新的slave或者重连的 slave无法继续同步master新产生的数据，就会执行全量同步。

将数据从master转移到slave需要借助rdb文件，并且Redis复制提供了两种方式来进行，disk和socket:

1）disk-backed：master创建一个新的线程用来将数据写入到RDB文件并保存在磁盘上。然后这个文件被主线程同步到从节点。所以当一个rdb保存的过程中，多个slave都能共享这个rdb文件。

2）diskless：master创建一个子线程，然后直接将rdb文件以socket的方式发给从节点。

```
repl-diskless-sync no
```



8. repl-diskless-sync-delay

 diskless复制的延迟时间。

一但复制开始，节点不会再接收新的slave的复制请求直到下一个rdb传输。所以最好等待一段时间，等待更多的slave连上来。



9. epl-disable-tcp-nodelay no

如果master设置为yes，那么在把数据复制给slave的时候，会合并较小的TCP数据包从而节省网络宽带的消耗，因此这可能会带来数据的延迟。相反如果设置为no，数据的传输延迟会更小，并且占用的带宽会更大。

```
epl-disable-tcp-nodelay no
```

10. repl-backlog-size

复制缓冲区大小，这是一个环形复制缓冲区，用来保存最新复制命令。这样在slave离线时，不需要全量复制master的数据，如果可以执行部分同步，只要把缓冲区的部分数据复制给slave，就能恢复复制状态。缓冲区的大小越大，slave离线的时间可以更长，复制缓冲区只有在有slave连接的时候才分配内存。没有slave的一段时间，内存会被释放出去，默认为1M。

```
repl-backlog-size 1mb
```

11. repl-backlog-ttl

master没有slave一段时间会释放复制缓冲区的内存，repl-backlog-ttl用来设置改时间长度。单位秒。

```
repl-backlog-ttl 3600
```

12. slave-priority

当master不可用时，Sentinel会根据slave的优先级选举一个master。优先级最低的slave当选为master。而配置为0则表示永远不会被选举为master。

 ```
slave-priority 100
 ```

13. min-slaves-to-write

redis 提供了可以让master停止写入的方式。如果配置了min-slaves-to-write，健康的slave的个数小于N，master就禁止写入。master最少得有多少个健康的slave才能执行写命令。设置为0时关闭该功能。

```
min-slaves-to-write 3
```

14. min-slaves-max-lag

延迟小于min-slaves-max-lag秒的slave才认为是健康的slave。