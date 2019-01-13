*最近买了阿里云香港节点,于是想测试一下速度
并提前学习一下django项目的实际部署
(以后的项目会试着用docker部署)*

> 阅读这篇文章前,笔者默认您已对linux vim sql django命令行有一定的了解

## Let's do it!


**OS:** 
Distributor ID: Ubuntu
Description:    Ubuntu 16.04.3 LTS
Release:        16.04
Codename:       xenial

>输入 lsb_release -a 可以看发行版本

最终的依赖环境如下:
nginx
gunicorn
supervisor
mysql(mariaDB)

### 创建工作用户
在操作服务器的时候不推荐用root用户直接操作,于是我们先创建一个用户

创建用户的命令是 `adduser` 

于是创建一个叫`yuzk`的用户,密码要连续输入两次,其他的选项可以直接回车默认为空就行
```

root@iZj6ch8q22e14njvxsyzasZ:~# adduser yuzk
Adding user `yuzk' ...
Adding new group `yuzk' (1002) ...
Adding new user `yuzk' (1002) with group `yuzk' ...
Creating home directory `/home/yuzk' ...
Copying files from `/etc/skel' ...
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
Changing the user information for yuzk
Enter the new value, or press ENTER for the default
        Full Name []:
        Room Number []:
        Work Phone []:
        Home Phone []:
        Other []:
Is the information correct? [Y/n] y

```

#### 赋予用户sudo权限
在部署服务器环境的时候root权限是必须的,所以我们要赋予刚才创建的用户sudo权限,权限编辑的命令是`visudo`,或者直接编辑`/etc/sudoers`

这里我们直接用vim编辑
用vim打开sudoers,在最后一行追加`yuzk ALL=NOPASSWD: ALL`



```

# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"

# Host alias specification

# User alias specification

# Cmnd alias specification

# User privilege specification
root    ALL=(ALL:ALL) ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

# See sudoers(5) for more information on "#include" directives:

#includedir /etc/sudoers.d
admin ALL=(ALL)  NOPASSWD:ALL
yuzk NOPASSWD: ALL
```

>由于sudoer是只读的,这里要用`:w!`来写入保存

#### 测试一下sudo
用`su`命令切换用户,然后测试一下能不能用sudo命令

如果ubuntu修改文件的时候一直提示：
>sudo: unable to resolve host [主机名字]

查看文件：/etc/hostname ,查看一下主机名字

可以修改位自己喜欢的名字
也可以用
`hostname [主机名字]`来修改

下面就需要修改/etc/hosts文件： 
```
127.0.0.1       localhost
127.0.0.1       beaock
# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
在第一行127.0.0.1       localhost 下面加一行自己刚才添加的主机名字
重启ssh就可以看到新的服务器名字了

然后继续测试sudo命令
```

yuzk@beaock:/$ sudo touch a
yuzk@beaock:/$ sudo rm a
yuzk@beaock:/$
```

### ssh秘钥设置
创建了用户,赋予sudo权限之后,ssh秘钥也顺便制作一下了
(现在的用户和密码如果泄露了,谁都可以登录服务器,而且这个用户还有sudo权限,泄露的话就不得了了)

所以说,在创建用户之后,推荐立即制作秘钥
然后设置一下ssh秘钥认证,设置免密码更为安全

#### 准备秘钥
我在这里用的是`puttygen.exe`来制作秘钥

##### 运行Puttygen.exe生成密钥

1. 下载putty.exe   puttygen.exe  双击即可使用，无需安装 
>下载地址： http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html

2. 运行Puttygen.exe生成密钥
>运行Puttygen.exe---->Parameters选项选择----->SSH-2 RSA------>
点击Generate 按钮开始生成密钥(可以在程序Key下方的空白处移动鼠标，直到生成密钥结束)---->点下面的Save private key把私钥保存起来，扩展名是 .ppk 的文件。此时不要关闭程序

3. 连接远程服务器，上传密钥使用Putty登陆远程服务器在用户目录下，创建~/.ssh/authorized_keys
>例如：  user@freemongolia.cn%mkdir ~/.ssh   user@freemongolia.cn%cd ~/.ssh    
user@freemongolia.cn%vi authorized_keys  
复制Puttygen.exe程序Public key for pasting into Open SSH authorized_keys file:下面的内容到服务器上的authorized_keys文件中粘贴并保存退出。


4. 使用Putty密钥方式验证自动登陆 
> 打开Putty.exe------>Session------>Host name(or IP address)输入远程服务器IP地址----->Connection------>data------>Auto-login username输入用于登陆的用户名-------->SSH---->Auth------>Private key file for authentication:----->点击Browser选择到你保存私钥(.pkk)的文件。  （注意在此，返回含有IP地址的最初页面，点击保存 save，这样以后登陆的时候，直接load我们所保存的这个IP就可以直接登陆了）---->Open自动登陆到服务器上了。


5.可以为此时所设置的PUTTY建立以快捷键

(这里笔者用的秘钥格式是RSA,用SSH-2 RSA失败了,怀疑是windows的问题)

#### 实现免密码登录

因为现在有秘钥了,所以我们可以实现免密码登录
但是以防万一,下面的作业完成后,先不要关闭目前的终端,另开一个终端验证一下,
防止无法登录服务器

sshd的设置在`/etc/ssh/sshd_config`

>root@beaock:/etc/ssh# vim sshd_config
```
SyslogFacility AUTHPRIV
#PasswordAuthentication yes
PasswordAuthentication no
```

PasswordAuthentication no 这一项设置成**no** 然后重启
>service sshd restart

如果失败了就用没关闭的终端吧no改回yes,然后在重启,寻找一下问题所在

(这里笔者用的秘钥格式是RSA,用SSH-2 RSA失败了,换成RSA格式的秘钥就成功了,怀疑是windows的问题)

### 服务器的环境设置

#### 中文设置
先把ubuntu的apt-get升级一下
>sudo apt-get update

安装中文包

language-pack-zh-hans 简体中文
language-pack-zh-hans-base
language-pack-zh-hant 繁体中文
language-pack-zh-hant-base

>sudo apt-get -y install language-pack-zh-hans-base language-pack-zh-hans

**设置中文**

>sudo update-locale LANG=zh_CN.UTF-8 LANGUAGE="zh_CN:zh"

然后重启终端或者`source /etc/default/locale`就可以变成中文了

```

yuzk@beaock:~$ date
2019年 01月 12日 星期六 14:52:20 CST
yuzk@beaock:~$
```

**设置时区**
>sudo cp /etc/localtime /etc/localtime.org
>sudo ln -sf /usr/share/zoneinfo/Japan /etc/localtime


```

yuzk@beaock:~$ sudo ln -sf /usr/share/zoneinfo/Japan /etc/localtime
yuzk@beaock:~$ date
2019年 01月 12日 星期六 15:55:29 JST
yuzk@beaock:~$
```
ps 笔者人在东京

### 安装Nginx
Nginx是web服务器,比apache支持更高的并发量,效率更高

> sudo apt-get install nginx


### 安装supervisor
守护进程
可以让服务器重启的时候让应用自动启动
>sudo apt-get install supervisor

设置守护进程在服务器重启的时候自动启动
>sudo update-rc.d supervisor defaults

### 安装mysql(mariaDB)
这俩数据库一样用法
>sudo apt-get install mariadb-server


>sudo apt-get install libmariadbd-dev

### 安装gunicorn
这是python WSGI HTTP的server
在安装完python-pip下载
#### 安装python的虚拟环境
`(为了避免各个项目冲突或者污染,我们一般把各个项目放到不通的python虚拟环境中)`

**安装virtualenv和virtualenvwrapper**
首先安装
>sudo apt-get install python-pip

然后安装
>sudo pip install virtualenv
>sudo pip install virtualenvwrapper

##### 详细设置
virtualenvwrapper能帮助我们输入更少的命令来操作virtualenv
但是首先要修改一下环境变量
修改WORKON_HOME的环境变量要修改~/.bashrc文件

>vim ~/.bashrc

在文件最后追加这两行:
```

source /usr/local/bin/virtualenvwrapper.shexport WORKON_HOME=~/.virtualenvs

```
然后输入命令`source .bashrc`来使之生效

然后就可以使用`mkvirtual`来制作虚拟环境了

### 创建虚拟环境

我也是第一次部署django的生产环境,就创建一个`deploytest`的环境,指定用python3的环境

>mkvirtualenv deploytest -p python3

```

yuzk@beaock:~$ mkvirtualenv deploytest -p python3
NOTE: Virtual environments directory /home/yuzk/.virtualenv does not exist. Crea                       ting...
Running virtualenv with interpreter /usr/bin/python3
Using base prefix '/usr'
New python executable in /home/yuzk/.virtualenv/deploytest/bin/python3
Also creating executable in /home/yuzk/.virtualenv/deploytest/bin/python
Installing setuptools, pip, wheel...
done.
virtualenvwrapper.user_scripts creating /home/yuzk/.virtualenv/deploytest/bin/pr                       edeactivate
virtualenvwrapper.user_scripts creating /home/yuzk/.virtualenv/deploytest/bin/po                       stdeactivate
virtualenvwrapper.user_scripts creating /home/yuzk/.virtualenv/deploytest/bin/pr                       eactivate
virtualenvwrapper.user_scripts creating /home/yuzk/.virtualenv/deploytest/bin/po                       stactivate
virtualenvwrapper.user_scripts creating /home/yuzk/.virtualenv/deploytest/bin/ge                       t_env_details
(deploytest) yuzk@beaock:~$
```

可以看到我们目前的工作环境变成`(deploytest) yuzk@beaock:~$`了


## 开始部署项目
>终于到了部署项目的阶段了,我在网上找了个用django写的开源博客,这次来部署一下这个项目做实验

GitHub的地址: [https://github.com/Rousong/django-blog-tutorial](https://github.com/Rousong/django-blog-tutorial)

在用户目录创建一个project目录,然后用git克隆一下这个项目(如果没有安装git,先`sudo apt-get install git
`安装一下git)
```

(deploytest) yuzk@beaock:~$ mkdir project
(deploytest) yuzk@beaock:~$ cd project/
(deploytest) yuzk@beaock:~/project$ git clone https://github.com/Rousong/django-blog-tutorial

正克隆到 'django-blog-tutorial'...
remote: Enumerating objects: 565, done.
remote: Total 565 (delta 0), reused 0 (delta 0), pack-reused 565
接收对象中: 100% (565/565), 189.73 KiB | 0 bytes/s, 完成.
处理 delta 中: 100% (279/279), 完成.
检查连接... 完成。
(deploytest) yuzk@beaock:~/project$ ls -l
总用量 4
drwxrwxr-x 7 yuzk yuzk 4096 1月  12 16:29 django-blog-tutorial
(deploytest) yuzk@beaock:~/project$

```




####  安装项目的依赖库

##### 由于这个博客项目用的是django默认的sqllite,但是我们这次要测试一下链接mysql
为了链接数据库的依赖包
~~> pip install mysqlclient~~
这里安装mysqlclient失败(应该是编译问题),于是改用用纯python写的pysql 效率低一些 但是测试来说足够了
> pip install pymysql

##### requirements.txt
这个是项目的依赖库文件,cd到工程目录执行
> pip install -r requirements.txt

##### gunicorn

>pip install gunicorn


### 设置数据库

初始化数据库
>sudo mysql_secure_installation
```

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] n
 ... skipping.

By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] Y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!

```

#### 设置数据库用户

创建一个用户来连接数据库 密码先设置为123456
>sudo mysql

>grant all on *.* to yuzk@localhost identified by '123456';

```

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> grant all on *.* to yuzk@localhost identified by '123456';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]>
```


#### 创建项目的数据库database

> mysql -u yuzk -p

```
(deploytest) yuzk@beaock:~/project/django-blog-tutorial$ mysql -u yuzk -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 52
Server version: 10.0.36-MariaDB-0ubuntu0.16.04.1 Ubuntu 16.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> create database blog_test character set utf8;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| blog_test          |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.00 sec)

```


### 设置部署用的各种设定

#### 设置django项目里的settings.py文件


```
# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = False

ALLOWED_HOSTS = ['*']

# Application definition


DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'blog_test',
        'USER': 'yuzk',
        'PASSWORD': '123456',
        'HOST': 'localhost',
    }
}

```


生产模式要先把debug模式关闭,这个关闭就要把ALLOWED_HOSTS = ['*']同时设置

db虽然用的是mariaDB,但是设置方法和mysql是一样的

#### 设置__init__.py文件
在项目文件夹下的__init__.py 添加两行内容
```
import pymysql
pymysql.install_as_MySQLdb()
```

>在 python3 中，改变了连接库，改为了 pymysql 库，使用pip 
install pymysql 进行安装，直接导入import pymysql使用本来在上面的基础上把 python3 的 pymysql 库安装上去就行了，但是问题依旧经过查阅得知， Django 依旧是使用 py2 的 MySQLdb 库的，得到进行适当的转换才行







#### 生产数据库迁移文件

>python manage.py migrate

```

(deploytest) yuzk@beaock:~/project/django-blog-tutorial$ python manage.py migrate
System check identified some issues:

WARNINGS:
?: (mysql.W002) MySQL Strict Mode is not set for database connection 'default'
        HINT: MySQL's Strict Mode fixes many data integrity problems in MySQL, such as data truncation upon insertion, by escalating warnings into errors. It is strongly recommended you activate it. See: https://docs.djangoproject.com/en/1.10/ref/databases/#mysql-sql-mode
Operations to perform:
  Apply all migrations: admin, auth, blog, comments, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying blog.0001_initial... OK
  Applying blog.0002_auto_20170517_1929... OK
  Applying comments.0001_initial... OK
  Applying sessions.0001_initial... OK
(deploytest) yuzk@beaock:~/project/django-blog-tutorial$

```

#### 确认一下是否生成了表


```

MariaDB [blog_test]> show tables;
+----------------------------+
| Tables_in_blog_test        |
+----------------------------+
| auth_group                 |
| auth_group_permissions     |
| auth_permission            |
| auth_user                  |
| auth_user_groups           |
| auth_user_user_permissions |
| blog_category              |
| blog_post                  |
| blog_post_tags             |
| blog_tag                   |
| comments_comment           |
| django_admin_log           |
| django_content_type        |
| django_migrations          |
| django_session             |
+----------------------------+
15 rows in set (0.00 sec)

MariaDB [blog_test]>
```


### 创建超级用户


虚拟环境下继续运行 python manage.py createsuperuser 命令创建一个超级用户，方便我们进入 Django 管理后台。

> python manage.py createsuperuser

```
(deploytest) yuzk@beaock:~/project/django-blog-tutorial$ python manage.py createsuperuser
Username (leave blank to use 'yuzk'):
Email address: beaock@gmail.com
Password:
Password (again):
Superuser created successfully.
```

填好账户密码就创建好了


### 收集静态文件

虚拟环境下继续运行 python manage.py collectstatic 命令收集静态文件到 static 目录下：

```

(deploytest) yuzk@beaock:~/project$ cd django-blog-tutorial/
(deploytest) yuzk@beaock:~/project/django-blog-tutorial$ python manage.py collectstatic
```


### 设置supervisor

接下来设置一下gunicorn的启动命令


`/etc/supervisor/conf.d/`下面新建一个` deploytest.conf`文件
复制以下的内容:
```

[program:deploytest]
user=yuzk
command=/home/yuzk/.virtualenv/deploytest/bin/gunicorn -w 2 blogproject.wsgi:application --bind=unix:///tmp/blogproject.sock
directory=/home/yuzk/project/django-blog-tutorial
stdout_logfile=/var/log/deploytest/info.log
stderr_logfile=/var/log/deploytest/error.log
numprocs = 1
stdout_logfile_maxbytes = 10MB
stderr_logfile_maxbytes = 10MB
stdout_logfile_backups = 5
stderr_logfile_backups = 5
autostart = true
autorestart = true
environment = LANG=zh_CN.UTF-8,LC_ALL=zh_CN.UTF-8,LC_LANG=zh_CN.UTF-8,
DJANGO_SETTINGS_MODULE="blogproject.settings"

```

>user这个地方根据自己的用户名设置就可以

>command就是实际开始运行时 supervisor执行的命令
>这里要制定自己的虚拟环境里面的路径
>`blogproject.wsgi:application` wsgi前面的名字要写app的文件夹文字

>directory要指定自己项目文件夹,wsgi所在的地方,也就是指定`manage.py`所在的路径

>最后的environment的环境设定 要指定自己的项目的settings的文件



> log 日志的输出路径 自己设置一下具体的路径

*因为输出日志的user这里我设置的是yuzk, 所以最好还是改一下日志文件夹的所有者*

```

yuzk@beaock:/var/log$ sudo mkdir deploytest
yuzk@beaock:/var/log$ sudo chown yuzk deploytest
yuzk@beaock:/var/log$

```

### 设置nginx

在`/etc/nginx/site-available/`路径下创建以下的文件`deploytest.conf`

```


upstream blog-app {
    server unix:///tmp/blogproject.sock;
}

server {
    listen 80;
    server_name deploytest.2zzy.com;

    access_log /var/log/nginx/deploytest.log;
    error_log /var/log/nginx/deploytest_error.log;

    proxy_pass_header Server;
    proxy_set_header Host $host;
    proxy_set_header REMOTE_ADDR $remote_addr;

    location /static {
        
        alias /home/yuzk/project/django-blog-tutorial/static;
    }

    location / {
        proxy_pass http://blog-app;
    }
}

```


然后复制方才的文件到site-enabled目录
```
$ cd /etc/nginx/sites-enabled
$ sudo ln -s ../sites-available/deploytest.conf deploytest.conf
```




## 接近尾声

>sudo supervisorctl 进入supervisor的命令行

>输入reload重启以下

>输入status查看状态 如果现实RUNNING就是成功开启了(开启需要几秒钟 有时候要刷新一下)

如果显示`unix:///var/run/supervisor.sock no such file`
name就再执行一次`sudo supervisord`

成功启动的话 就会生成`/tmp/blogproject.sock`这个文件

查看一下进程
```

yuzk@beaock:~$ ps aux| grep blog
yuzk     30844  0.0  2.2  63140 22812 ?        S    18:53   0:00 /home/yuzk/.virtualenv/deploytest/bin/python3 /home/yuzk/.virtualenv/deploytest/bin/gunicorn -w 2 blogproject.wsgi:application --bind=unix:///tmp/blogproject.sock
yuzk     30847  0.0  3.5  90524 35576 ?        S    18:53   0:00 /home/yuzk/.virtualenv/deploytest/bin/python3 /home/yuzk/.virtualenv/deploytest/bin/gunicorn -w 2 blogproject.wsgi:application --bind=unix:///tmp/blogproject.sock
yuzk     30848  0.0  3.5  90528 35576 ?        S    18:53   0:00 /home/yuzk/.virtualenv/deploytest/bin/python3 /home/yuzk/.virtualenv/deploytest/bin/gunicorn -w 2 blogproject.wsgi:application --bind=unix:///tmp/blogproject.sock
yuzk     30907  0.0  0.0  15984  1016 pts/3    S+   18:59   0:00 grep --color=auto blog
yuzk@beaock:~$

```

看,果然开启了,我们已经接近成功


#### 重启nginx 

最后我们重启一下nginx

>sudo nginx -s reload

然后输入网址试验一下看看是否可以正常访问吧!!!

>地址一定要解析到服务器上才行

项目部署地址: [deploytest.2zzy.com][1]

笔者博客地址 [www.do1024.com](https://www.do1024.com/)
## 总结

花了5,6个小时写这篇文章,一边做一边写
其中遇到几个坑,但是总体还算顺利
大家也可以看出,这样一点点安装环境和输入命令简直是流程又臭又长
这次之所以这样做是因为自己没有真正从头开始部署一个生产环境,
虽然麻烦但是也学到了很多东西

今后想要部署更多的应用的时候,如果每次都这样部署简直是太麻烦了
所以说以后会写一个docker的compose脚本来做这些事情



  [1]: http://deploytest.2zzy.com