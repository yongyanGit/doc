#### Logstash

1. 启动

```
bin/logstash -e 'input { stdin { } } output { stdout {} }'
```

* e：直接在控制台测试日志输入输出，不用去编辑配置文件。

2. 加载配置文件启动

```
 bin/logstash -f config/logstash.conf
```

