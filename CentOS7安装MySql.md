---
title: CentOS7安装MySql5.7
tags: MySql
grammar_cjkRuby: true
---
***
## 环境

* 本文操作系统: CentOS 7.2.1511 x86_64
* MySQL 版本: 5.7.17
* 安装媒介： rpm-bundle.tar

## 卸载系统自带的 mariadb-lib

``` bash
[root@centos-linux ~]# rpm -qa|grep mariadb				
mariadb-libs-5.5.44-2.el7.centos.x86_64
[root@centos-linux ~]# rpm -e mariadb-libs-5.5.44-2.el7.centos.x86_64 --nodeps
```
## 下载 rpm 安装包

本次安装下载的是mysql-5.7.17-1.el7.x86_64.rpm-bundle.tar

解压tar文件
``` bash
[root@localhost extra]# ls
mysql-community-client-5.7.17-1.el7.x86_64.rpm  
mysql-community-embedded-5.7.17-1.el7.x86_64.rpm    
mysql-community-libs-5.7.17-1.el7.x86_64.rpm         
mysql-community-server-5.7.17-1.el7.x86_64.rpm
mysql-community-common-5.7.17-1.el7.x86_64.rpm 
mysql-community-embedded-compat-5.7.17-1.el7.x86_64.rpm
mysql-community-libs-compat-5.7.17-1.el7.x86_64.rpm   
mysql-community-server-minimal-5.7.17-1.el7.x86_64.rpm
mysql-community-devel-5.7.17-1.el7.x86_64.rpm 
mysql-community-embedded-devel-5.7.17-1.el7.x86_64.rpm 
mysql-community-minimal-debuginfo-5.7.17-1.el7.x86_64.rpm  
mysql-community-test-5.7.17-1.el7.x86_64.rpm
```
查看包里的内容
```bash
rpm -qpl mysql-community-*
```
然后可以借助于yum安装或者rpm安装

```yum
yum install mysql-community-{server,client,common,libs}-5*
```
或者
```rpm
rpm -ivh mysql-community-common-5.7.17-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.17-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.17-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.17-1.el7.x86_64.rpm
```
## 数据库初始化
启动mysql服务
```bash
service mysqld start
```
默认启动会自动创建root的密码
```bash
[root@localhost extra]# grep 'temporary password' /var/log/mysqld.log
2016-12-14T07:19:17.002312Z 1 [Note] A temporary password is generated for root@localhost: PlWTljaNU4<;
```
然后使用上面的密码登录mysql
```bash
mysql -uRoot -p
```
修改原始密码,密码检查默认启用,需要至少8个字符,包含一个大写,一个小写,一个数字和一个特殊字符
```bash
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!'
```
`MyNewPass4! `是你的密码
当然,如果想要被别的客户端访问需要把`localhost`改为对方IP,如果是全部机器可访问,可以修改为`%`
