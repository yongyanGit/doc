### Bash命令

检查linux支持的shell：

```
cat /etc/shells 

/bin/sh
/bin/bash
/usr/bin/sh
/usr/bin/bash
/bin/tcsh
/bin/csh
```



#### 指令换行

当我们在控制台输入的指令过长时，需要进行换行，可以通过 ‘\’+空格来让指令在下一行输入。

当输入了一串错误的指令时，可以通过以下命令在进行删除：

```
[ctrl]+u 从光标处向前删除
[ctrl]+k 从光标处向后删除
[ctrl]+a 将光标移动到整个字符串的最前面
[ctrl]+e 将光标移动到最后面
```

#### 设置变量

```
myname=VBird
echo ${myname}
```

**变量设置规则**

1. 变量与变量内容以一个等号(=)来连接。

```
[yongyan@localhost ~]$ name=bird
```

2. 等号两边不能直接接空格。

```
[yongyan@localhost ~]$ name = vbird
bash: name: 未找到命令
```

3. 变量名称只能是英文字母和字母，但是开头不能是数字。

```
[yongyan@localhost ~]$ 12name=vbird
bash: 12name=vbird: 未找到命令...
```

4. 变量内容若有空格，可使用双引号或者单引号将变量内容结合起来。双引号内的特殊字符如：$等，可以保有原本的特性。单引号内的特殊字符则仅为一般字符（纯文本）。

```
[yongyan@localhost ~]$ name="bird's name"
##单引号与双引号的区别
[yongyan@localhost ~]$ myname='${name}its me'
[yongyan@localhost ~]$ echo ${myname} 
${name}its me

[yongyan@localhost ~]$ myname="${name}its me"
[yongyan@localhost ~]$ echo ${myname} 
bird's nameits me
```

5. 反斜杠\可以将特殊字符转译成一般字符。

```
[yongyan@localhost ~]$ name=bird\'s\ name
```

6. 若变量是扩增变量内容时，可用'$变量名'或者${变量}累加内容

```
PATH=${PATH}:/home/dmtsai/bin
```

7. 若该变量需要在其它子程序执行，则需要以export来使变量变成环境变量。

```
##先设定一个值
[yongyan@localhost ~]$ name=bird
[yongyan@localhost ~]$ echo ${name} 
bird
## 新开shell
[yongyan@localhost ~]$ bash
##下面的命令没有值
[yongyan@localhost ~]$ echo ${name}
## 退出
[yongyan@localhost ~]$ exit
## 将变量变成环境变量
[yongyan@localhost ~]$ export name
[yongyan@localhost ~]$ bash
[yongyan@localhost ~]$ echo ${name} 
bird

```

8. 在一串指令的执行中，还需要藉由其它额外的指令所提供的信息时，可以使用反单引号(`)或者($())。

```
[yongyan@localhost ~]$ cd /lib/modules/$(uname -r)/kernel
```

9. 取消变量的方法为使用unset

#### 环境变量

```
ngyan@localhost ~]$ env

## 主机名
HOSTNAME=localhost.localdomain
## 当前环境下，使用的Shell是哪一种
SHELL=/bin/bash
## 记录指令的笔数
HISTSIZE=1000
## 上一个工作目录的所在
OLDPWD=/lib/modules/3.10.0-957.el7.x86_64/kernel
## 使用者
USER=yongyan
##当前用户所使用的mail位置，当我们使用mail指令收信时，系统会去读取邮件信箱文件(mailbox)
MAIL=/var/spool/mail/yongyan
PATH=/usr/local/bin
## 目前用户所在的工作目录
PWD=/home/yongyan
## 语系
LANG=zh_CN.UTF-8
## 当前登录用户的家目录
HOME=/home/yongyan
##登录者用来登录的账号名称
LOGNAME=yongyan
## 上一次使用的指令的最后一个参数
_=/usr/bin/env
```



