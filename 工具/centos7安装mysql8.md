### 使用yum方式安装mysql8

1. 由于centos7中存在默认的mariadb, 所以需要下载
[mysql的yum源]('https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm')

2. 安装源
```shell
yum -y install mysql80-community-release-el7-3.noarch.rpm
```

3. 安装mysql-community-server
```shell
yum -y install mysql-community-server
```

4. 启动mysql
```shell
systemctl start  mysqld.service
```

5. 获取默认的root密码
```shell
grep "password" /var/log/mysqld.log
```

6. 登录mysql之后修改密码设置规范**(如果不设置会导致新密码太简单而报错)**
```shell
set global validate_password.policy=0;
```

7. 设置密码
```shell
ALTER USER 'root'@'localhost' IDENTIFIED BY 'new password';
```

###### 完成
