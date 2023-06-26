# mysql安装

## 1.获取软件包

https://downloads.mysql.com/archives/community/

![1645492220241](assets\1645492220241.png)

## 2.创建用户

```bash
$ sudo useradd mysql -s /usr/sbin/nologin
```

## 3.创建mysql数据路径并授权

```bash
$ mkdir -p mysql/data_3306
$ sudo chown -R mysql.mysql data_3306/
```

## 4.准备配置文件

`sudo vim /etc/my.cnf `

```ini
[mysqld]
user=mysql
basedir=/usr/local/mysql
datadir=/data/mysql/data_3306
socket=/tmp/mysql.sock
[mysql]
socket=/tmp/mysql.sock
```

## 5.解压文件做软连接

```bash
$ tar xf mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz
$ sudo ln -s /data/install_mysql/mysql-8.0.20-linux-glibc2.12-x86_64 /usr/local/mysql
```

## 6.添加环境变量

```bash
$ sudo vim /etc/profile
export PATH=/usr/local/mysql/bin:$PATH
$ source /etc/profile
$ mysql -V  #验证mysql命令
```

## 7.初始化mysql系统数据

```bash
$ sudo /usr/local/mysql/bin/mysqld --initialize-insecure --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysql/data_3306
```

在初始化时还可以选择`--initialize`参数来进行初始化，使用这个参数初始化会生成一个临时密码，登录进去需要重新修改密码，否则将无法管理数据库。而使用`--initialize-insecure`这个参数将可以不用密码登录root用户。


- 初始化会遇到的报错
  1. 在数据路径下不能有文件存在
  2. 使用非root用户时，sudo加绝对路径
  3. 缺少libaio.so.1的包，centos安装libaio-devel包，ubuntu安装libaio1包

## 8启动管理

```bash
$ sudo cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
$ sudo /etc/init.d/mysqld start
```





