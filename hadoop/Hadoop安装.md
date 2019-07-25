### Hadoop安装

#### 安装JDK（centos7）

1. 卸载自带jdk

```
#查看系统原生的jdk
rpm -qa|grep java
#卸载
yum -y remove java*
```

2. 安装JDK，将下载好的JDK安装包解压到指定目录下

```
#解压
tar -zxvf jdk-8ul44-linux-x64.tar.gz
#将jdk移动到指定目录下,并命名为jdk
mv jdk-8ul44-linux-x64 /usr/local/jdk
```

3. 配置环境变量

```
#打开全局环境变量配置文件
vim /etc/profile
# 添加具体内容
export JAVA_HOME=/usr/local/jdk
export PATH=$PATH:$JAVA_HOME/bin/
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
```

3. 使全局环境生效

```
# 退出vim 然后使配置文件生效
source /etc/profile

#查看配置是否生效
javac
```





