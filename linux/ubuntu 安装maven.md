## ubuntu 安装maven

### 下载maven

apache maven官网地址：http://maven.apache.org/download.cgi

 找到Link列下的“apache-maven-3.5.2-bin.tar.gz“进行下载。

### 二、安装配置maven

解压maven：

```
$ sudo tar zxvf apache-maven-3.5.2-bin.tar.gz -C /opt/
```

 

配置maven环境变量：

```
$ sudo vim ~/.bashrc
```

在文本最后加入以下几句：

```
export M2_HOME=/opt/apache-maven-3.5.2
export CLASSPATH=$CLASSPATH:$M2_HOME/lib
export PATH=$PATH:$M2_HOME/bin
```

使文件生效：

```
$ source ~/.bashrc
```

### 三、查看maven版本信息

```
$ mvn -v
```

显示出以下信息即安装成功：

```
Apache Maven 3.5.2 (138edd61fd100ec658bfa2d307c43b76940a5d7d; 2017-10-18T15:58:13+08:00)
Maven home: /opt/maven/apache-maven-3.5.2
Java version: 1.8.0_151, vendor: Oracle Corporation
Java home: /opt/jdk1.8.0_151/jre
Default locale: zh_CN, platform encoding: UTF-8
OS name: "linux", version: "4.4.0-98-generic", arch: "amd64", family: "unix"
```