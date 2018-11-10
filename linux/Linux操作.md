### Linux操作

#### 文件上传

```
scp eid-service.zip root@172.16.0.111:/opt/eiddir
```

#### 解压文件

```
unzip eid-service.zip
```

查看端口占用

```
netstat -apn | grep 4040
kill -9 26105
```

mvn打包

```
mvn clean package -Dmaven.test.skip=true
nohub java -jar xxxx.jar&
```


1、mvn clean install -DskipTests

1、 sh stopMicrosoftService_back20180718.sh

2、 sh startMicrosoftService_back20180718.sh

docker-compose up -d
docker-compose down



docker pull reg.evercenter.cn/eid-service/eid-service:latest

docker tag reg.evercenter.cn/eid-service:v1.1 reg.evercenter.cn/eid-service/eid-service:latest

docker push reg.evercenter.cn/eid-service/eid-service:latest






——————————————————————————————
解决方法1

使用sudo获取管理员权限，运行docker命令
解决方法2

docker守护进程启动的时候，会默认赋予名字为docker的用户组读写Unix socket的权限，因此只要创建docker用户组，并将当前用户加入到docker用户组中，那么当前用户就有权限访问Unix socket了，进而也就可以执行docker相关命令

sudo groupadd docker     #添加docker用户组
sudo gpasswd -a $USER docker     #将登陆用户加入到docker用户组中
newgrp docker     #更新用户组
docker ps    #测试docker命令是否可以使用sudo正常使用


登录harbor
docker login  -u yanyong@seaeverit.com -p A1991100598yan https://reg.evercenter.cn/harbor


