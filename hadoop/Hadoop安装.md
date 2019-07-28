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

#### 创建Hadoop 账号

```
#查看系统已经创建的用户
cat /etc/passwd
##创建账号
useradd hadoop
## 设置密码
passwd hadoop
## 给sudoers 文件赋予写权限
chmod +w /etc/sudoers
## 对sudoers文件进行编辑,免登录
vim /etc/sudoers
hadoop ALL=(root)NOPASSWD :ALL
## 取消sudoers 文件的写权限
chmod -w /etc/sudoers
```

#### 配置host系统文件

```
## 配置ip
## 修改配置文件
vim /etc/sysconfig/network-scripts/ifcfg-ens33 
#开启启动
ONBOOT=YES
#子网掩码
NETMASK=255.255.255.0
#ip地址
IPADDR=192.168.252.130	
# 默认网关(没有配置)
GATEWAY=192.168.122.1

## 修改hosts 文件
vim /etc/hosts
# 增加内容
192.168.252.130 nna
192.168.252.131	nns
192.168.252.132 dn1
192.168.252.133 dn2
192.168.252.134 dn3

# 将修改变动同步到其它服务器
scp /etc/hosts root@nns:/etc/
scp /etc/hosts root@dn1:/etc/
scp /etc/hosts root@dn2:/etc/
scp /etc/hosts root@dn3:/etc/

```

#### 安装SSH

Hadoop 集群需要保证各个节点互相通信，需要用到SSH。

```
# 在每个节点下生成公钥和私钥
ssh-keygen -t rsa
# 将每个节点下的公钥都写入authorized_keys,如nns 节点生成的公钥汇集到nna节点
scp id.rsa.pub hadoop@nna:~/.ssh/id.rsa.pub.nns
#将各个节点汇集过来的公钥追加到authorized_keys文件中,如nns 的公钥追加到authorized_keys
cat id.rsa.pub.nns >> ~/.ssh/authorized_keys
#将完整的authorized_keys分发到各个节点下，如：nna 到nns
scp authorized_keys hadoop@nns:~/.ssh/
# 在hadoop 账号下，需要给authorized_keys 文件赋予600权限
chmod 600 ~/.ssh/authorized_keys

##验证是否可以免密码登录
ssh nns 
```

#### 关闭防火墙

```
# 关闭
[hadoop@localhost ~]$ chkconfig iptables off
# 检查是否关闭
chkconfig list
```

#### 修改时区

略。



