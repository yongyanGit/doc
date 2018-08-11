### mysql文档

**登陆mysql**

``mysql -h 120.0.0.1 -u root -p ``

**查看mysql使用了哪些配置文件**

进入到mysql的安装bin目录下面，执行下面的命令 

``` 
./mysql --help|grep my.cnf
```

**查看mysql的数据存放在哪个目录下面**

首先登陆mysql,然后执行下面的命令

```show variables like 'datadir'\G```

接着查看目录下存放着什么数据

```system ls -lh /usr/local/mysql/data```

**查看mysql缓存池的大小**

```
show variables like 'innodb_buffer_pool_size'\G
```

**查看重做日志缓冲池大小**

```
show variables like 'innodb_log_buffer_size'\G
```

**查看额外的内存池大小**

```
show variables like 'innodb_additional_mem_pool_size'\G
```

缓冲池是用来存放各种数据的缓存，InnoDB的存储引擎的工作方式总是将数据库的文件按照页（每页16k）读取到缓冲池，然后将LRU算法保留缓冲池中的数据，如果数据库文件需要修改，总是首先修改缓存池中的页，然后将缓冲池中的脏页刷新到文件。

**查看缓冲池的使用情况**

```
show engine innodb status\G
```

执行命令结果

```
Buffer pool size   8192
Free buffers       7817
Database pages     375
Old database pages 0
Modified db pages  0
```

buffer pool size代表有多少个缓冲帧（每个缓存帧16k），Free buffers表示当前空闲的缓冲帧，Database pages表示已经使用的缓冲帧，Modified db pages 表示脏页的数量。

**mysql内存结构图**

![内存结构图](../images/innodbbuff.png)



