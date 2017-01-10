# 背景
规划

192.168.6.114  mysql master
192.168.6.115  mysql salve1
192.168.6.119  mysql slave2
192.168.6.121  maxscale

搭建114 115 119的一主两丛结构并开启半同步复制，和搭建mha一样
在搭建完主从后我们还需要在数据库建一个用于monitor的用户，同时建好后面测试的用户：
```mysql
mysql> grant all privileges on *.* to 'max'@'%' identified by 'ESBecs00';   ---为了省事就用这一个用户吧 
```


# 安装maxscale
下载链接：https://downloads.mariadb.com/files/MaxScale/1.3.0-beta/centos/5/x86_64
本次博客安装的是
maxscale-beta-1.3.0-1.centos5.x86_64.rpm 该版本

# 修改配置文件
安装完后会在/etc/目录下生成模板配置文件
[root@localhost etc]# ls maxscale.cnf.template     
maxscale.cnf.template

写一个配置文件（每个版本的配置文件不一样，最好是cp模板配置文件来修改）
[root@localhost etc]# more maxscale.cnf
## MaxScale documentation on GitHub:

> https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Documentation-Contents.md



## Global parameters

>  Number of threads is autodetected, uncomment for manual configuration
> Complete list of configuration options:
> 
>  https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Getting-Started/Configuration-Guide.md

``` ini
[maxscale]
threads=8
```


## Server definitions

>  Set the address of the server to the network  address of a MySQL
> server.




``` ini
[server1]
type=server
address=192.168.6.114
port=3306
protocol=MySQLBackend

[server2]
type=server
address=192.168.6.115
port=3306
protocol=MySQLBackend

[server3]
type=server
address=192.168.6.119
port=3306
protocol=MySQLBackend
```


## Monitor for the servers


>  This will keep MaxScale aware of the state of the servers.
>  MySQL Monitor documentation:
> 
> https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Monitors/MySQL-Monitor.md

``` ini
[MySQL Monitor]
type=monitor
module=mysqlmon
servers=server1,server2,server3
user=max
passwd=ESBecs00
monitor_interval=10000
```


# Service definitions

>  Service Definition for a read-only service and  a read/write
> splitting service.
> 
> 
> https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Routers/ReadConnRoute.md

``` nix
[Read-Only Service]                       ###只读服务
type=service
router=readconnroute
servers=server1,server2,server3
user=max
passwd=ESBecs00
router_options=slave
```


 

> https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Routers/ReadWriteSplit.md

[Read-Write Service]                       ####写服务
type=service
router=readwritesplit
servers=server1
user=max
passwd=ESBecs00
max_slave_connections=100%

> https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Reference/MaxAdmin.md

``` ini
[MaxAdmin Service]
type=service
router=cli


[Read-Only Listener]
type=listener
service=Read-Only Service
protocol=MySQLClient
port=4008                                  ##读服务启动监听 端口4008

[Read-Write Listener]
type=listener
service=Read-Write Service
protocol=MySQLClient
port=4006                                  ####写服务启动监听 端口

[MaxAdmin Listener]
type=listener
service=MaxAdmin Service
protocol=maxscaled
port=6603                                   ###管理端口
```




3 启动maxscale服务

``` scheme
  [root@localhost etc]# /etc/init.d/maxscale start
Starting MaxScale:  found maxscale (pid 
14126) 正在运行...                                         [确定]
```


查看message日志

``` less
[root@localhost ~]# tail -f /var/log/messages
Jul 29 17:04:33 localhost maxscale[14035]: Configuration file: /etc/maxscale.cnf
Jul 29 17:04:33 localhost maxscale[14035]: Log directory: /var/log/maxscale               ------日志目录
Jul 29 17:04:33 localhost maxscale[14035]: Data directory: /var/lib/maxscale/data
Jul 29 17:04:33 localhost maxscale[14035]: Module directory: /usr/lib64/maxscale
Jul 29 17:04:33 localhost maxscale[14035]: Service cache: /var/cache/maxscale
Jul 29 17:04:33 localhost maxscale[14035]: Initialise CLI router module V1.0.0.
Jul 29 17:04:33 localhost maxscale[14035]: Loaded module cli: V1.0.0 from /usr/lib64/maxscale/libcli.so
Jul 29 17:04:33 localhost maxscale[14035]: Initializing statemend-based read/write split router module.
Jul 29 17:04:33 localhost maxscale[14035]: Loaded module readwritesplit: V1.0.2 from /usr/lib64/maxscale/libreadwritesplit.so
Jul 29 17:04:33 localhost maxscale[14035]: Initialise readconnroute router module V1.1.0.
Jul 29 17:04:33 localhost maxscale[14035]: Loaded module readconnroute: V1.1.0 from /usr/lib64/maxscale/libreadconnroute.so
```


查看日志：

``` groovy
[root@localhost maxscale]# tail -f maxscale1.log 
MariaDB Corporation MaxScale    /var/log/maxscale/maxscale1.log Fri Jul 29 17:09:46 2016
-----------------------------------------------------------------------
2016-07-29 17:09:46   notice : Configuration file: /etc/maxscale.cnf
2016-07-29 17:09:46   notice : Log directory: /var/log/maxscale
2016-07-29 17:09:46   notice : Data directory: /var/lib/maxscale/data
2016-07-29 17:09:46   notice : Module directory: /usr/lib64/maxscale
2016-07-29 17:09:46   notice : Service cache: /var/cache/maxscale
2016-07-29 17:09:46   notice : Initialise CLI router module V1.0.0.
2016-07-29 17:09:46   notice : Loaded module cli: V1.0.0 from /usr/lib64/maxscale/libcli.so
2016-07-29 17:09:46   notice : Initializing statemend-based read/write split router module.
2016-07-29 17:09:46   notice : Loaded module readwritesplit: V1.0.2 from /usr/lib64/maxscale/libreadwritesplit.so
2016-07-29 17:09:46   notice : Initialise readconnroute router module V1.1.0.
2016-07-29 17:09:46   notice : Loaded module readconnroute: V1.1.0 from /usr/lib64/maxscale/libreadconnroute.so
2016-07-29 17:09:46   notice : Initialise the MySQL Monitor module V1.4.0.
2016-07-29 17:09:46   notice : Loaded module mysqlmon: V1.4.0 from /usr/lib64/maxscale/libmysqlmon.so
2016-07-29 17:09:46   notice : MariaDB Corporation MaxScale beta-1.3.0 (C) MariaDB Corporation Ab 2013-2015
2016-07-29 17:09:46   notice : MaxScale is running in process 14140
2016-07-29 17:09:46   notice : Loaded 2 MySQL Users for service [Read-Only Service].
2016-07-29 17:09:46   notice : Loaded module MySQLClient: V1.0.0 from /usr/lib64/maxscale/libMySQLClient.so
2016-07-29 17:09:46   notice : Listening MySQL connections at 0.0.0.0:4008
2016-07-29 17:09:46   notice : Loaded 2 MySQL Users for service [Read-Write Service].
2016-07-29 17:09:46   notice : Listening MySQL connections at 0.0.0.0:4006
2016-07-29 17:09:46   notice : Loaded module maxscaled: V1.0.0 from /usr/lib64/maxscale/libmaxscaled.so
2016-07-29 17:09:46   notice : Listening maxscale connections at 0.0.0.0:6603
2016-07-29 17:09:46   notice : Started MaxScale log flusher.
2016-07-29 17:09:46   notice : MaxScale started with 8 server threads.
2016-07-29 17:09:47   notice : A Master Server is now available: 192.168.6.114:3306
```





4测试
读负载均衡测试

``` asciidoc
[root@localhost etc]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
Warning: Using a password on the command line interface can be insecure.
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test115 |
+---------------------+
[root@localhost etc]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
Warning: Using a password on the command line interface can be insecure.
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test119 |
+---------------------+
[root@localhost etc]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
Warning: Using a password on the command line interface can be insecure.
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test115 |
+---------------------+
[root@localhost etc]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
Warning: Using a password on the command line interface can be insecure.
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test119 |
+---------------------+
```


写测试

``` asciidoc
[root@localhost etc]# mysql -umax -pESBecs00 -h192.168.6.121 -P4006 -e "create table tt as select * from mysql.user" chenliang;    --创建一个表
Warning: Using a password on the command line interface can be insecure.
[root@localhost etc]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;                                     ---同步到slave1
Warning: Using a password on the command line interface can be insecure.
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test115 |
| tt |
+---------------------+
[root@localhost etc]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;                                     ---同步到slave2了，很明显写在了主库上，可以开generlog去验证
Warning: Using a password on the command line interface can be insecure.
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test119 |
| tt |
+---------------------+
```



5 后台管理

``` asciidoc
[root@localhost ~]# maxadmin -pmariadb
MaxScale> 
MaxScale> 
MaxScale> list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server | Address | Port | Connections | Status 
-------------------+-----------------+-------+-------------+--------------------
server1 | 192.168.6.114 | 3306 | 0 | Master, Running
server2 | 192.168.6.115 | 3306 | 0 | Slave, Running
server3 | 192.168.6.119 | 3306 | 0 | Slave, Running
-------------------+-----------------+-------+-------------+--------------------
MaxScale> list services
Services.
--------------------------+----------------------+--------+---------------
Service Name | Router Module | #Users | Total Sessions
--------------------------+----------------------+--------+---------------
Read-Only Service | readconnroute | 1 | 3
Read-Write Service | readwritesplit | 1 | 1
MaxAdmin Service | cli | 5 | 5
--------------------------+----------------------+--------+---------------
```


故障测试之：一台slave，stop
原始状态

``` asciidoc
MaxScale> list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server | Address | Port | Connections | Status 
-------------------+-----------------+-------+-------------+--------------------
server1 | 192.168.6.114 | 3306 | 0 | Master, Running
server2 | 192.168.6.115 | 3306 | 0 | Slave, Running
server3 | 192.168.6.119 | 3306 | 0 | Slave, Running
-------------------+-----------------+-------+-------------+--------------------

停掉119的slave，stop slave；查看状态
MaxScale> list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server             | Address         | Port  | Connections | Status              
-------------------+-----------------+-------+-------------+--------------------
server1            | 192.168.6.114   |  3306 |           0 | Master, Running
server2            | 192.168.6.115   |  3306 |           0 | Slave, Running
server3            | 192.168.6.119   |  3306 |           0 | Running
-------------------+-----------------+-------+-------------+--------------------
```


此时读负载测试：全部落到正常的那台slave

``` asciidoc
[root@node4 ~]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test115             | 
| tt                  | 
| tt2                 | 
+---------------------+
[root@node4 ~]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test115             | 
| tt                  | 
| tt2                 | 
+---------------------+
[root@node4 ~]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test115             | 
| tt                  | 
| tt2                 | 
+---------------------+
```



恢复119 在测试：负载均衡正常了

``` asciidoc
MaxScale> list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server | Address | Port | Connections | Status 
-------------------+-----------------+-------+-------------+--------------------
server1 | 192.168.6.114 | 3306 | 0 | Master, Running
server2 | 192.168.6.115 | 3306 | 0 | Slave, Running
server3 | 192.168.6.119 | 3306 | 0 | Slave, Running
-------------------+-----------------+-------+-------------+--------------------
[root@node4 ~]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test119             | 
| tt                  | 
| tt2                 | 
+---------------------+
[root@node4 ~]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test115             | 
| tt                  | 
| tt2                 | 
+---------------------+
```






故障测试之：两台slave全部stop
停掉1156，119的slave;stop slave
查看状态

``` asciidoc
MaxScale> list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server | Address | Port | Connections | Status 
-------------------+-----------------+-------+-------------+--------------------
server1 | 192.168.6.114 | 3306 | 0 | Running
server2 | 192.168.6.115 | 3306 | 0 | Running
server3 | 192.168.6.119 | 3306 | 0 | Running
-------------------+-----------------+-------+-------------+--------------------
```


负载测试：此时负载就出问题，会报错，起码得保持一台slave，存活才行
[root@node4 ~]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
ERROR 1045 (28000): failed to create new session
说明从服务器全部失效后，会导致 master 也无法识别，使整个数据库服务都失效了。

对于 slave 全部失效的情况，能否让 master 还可用？这样至少可以正常提供数据库服务。

这需要修改 MaxScale 的配置，告诉 MaxScale 我们需要一个稳定的 master。
处理过程

先恢复两个 slave，让集群回到正常状态，登陆两个 slave 的MySQL。
mysql> start slave;

修改 MaxScale 配置文件，添加新的配置。
vi /etc/maxscale.cnf

找到 [MySQL Monitor] 部分，添加：
detect_stale_master=true

保存退出，然后重启 MaxScale。

验证

停掉两台 slave ，查看 MaxScale 服务器状态。

``` asciidoc
MaxScale> list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server | Address | Port | Connections | Status 
-------------------+-----------------+-------+-------------+--------------------
server1 | 192.168.6.114 | 3306 | 0 | Master, Stale Status, Running
server2 | 192.168.6.115 | 3306 | 0 | Running
server3 | 192.168.6.119 | 3306 | 0 | Running
-------------------+-----------------+-------+-------------+--------------------
```



负载测试：

``` asciidoc
[root@node4 ~]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test114 | 
| tt | 
| tt2 | 
+---------------------+
[root@node4 ~]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test114 | 
| tt | 
| tt2 | 
+---------------------+
[root@node4 ~]# mysql -umax -pESBecs00 -h192.168.6.121 -P4008 -e "show tables" chenliang;
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test114 | 
| tt | 
| tt2 | 
+---------------------+
```





补充：可将读写分离配置在一个port上
配置文件

``` nix
[maxscale]
threads=80

[server1]
type=server
address=192.168.6.114
port=3306
protocol=MySQLBackend

[server2]
type=server
address=192.168.6.115
port=3306
protocol=MySQLBackend

[server3]
type=server
address=192.168.6.119
port=3306
protocol=MySQLBackend


[MySQL Monitor]
type=monitor
module=mysqlmon
servers=server1,server2,server3
user=max
passwd=ESBecs00
monitor_interval=10000
detect_stale_master=true


#[Read-Only Service]
#type=service
#router=readconnroute
#servers=server1,server2,server3
#user=max
#passwd=ESBecs00
#router_options=slave


[Read-Write Service]
type=service
router=readwritesplit
servers=server1,server3,server2      -----都配置在这即可
user=max
passwd=ESBecs00
max_slave_connections=100%


[MaxAdmin Service]
type=service
router=cli




#[Read-Only Listener]
#type=listener
#service=Read-Only Service
#protocol=MySQLClient
#port=4008


[Read-Write Listener]
type=listener
service=Read-Write Service
protocol=MySQLClient
port=4006


[MaxAdmin Listener]
type=listener
service=MaxAdmin Service
protocol=maxscaled
port=6603
#####只需要将 read-only 读负载均衡的部分注释掉，只保留read-write部分即可
```


重启测试
写操作

``` sql
[root@node4 ~]# mysql -uchenliang -pESBecs00 -h192.168.6.121 -P4006  -e "create table tt3 as select * from tt" chenliang;
```


检查114，115，119都存在

``` asciidoc
mysql> show tables;
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test119             |
| tt                  |
| tt2                 |
| tt3                 |
+---------------------+
4 rows in set (0.00 sec)
```


 读测试：落在了slave上

``` asciidoc
[root@node4 ~]# mysql -uchenliang -pESBecs00 -h192.168.6.121 -P4006 -e "show tables" chenliang;
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test115 | 
| tt | 
| tt2 | 
| tt3 | 
+---------------------+
[root@node4 ~]# mysql -uchenliang -pESBecs00 -h192.168.6.121 -P4006 -e "show tables" chenliang;
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test115 | 
| tt | 
| tt2 | 
| tt3 | 
+---------------------+
```









对比：
实验一：实现了读写分离，和读的负载均衡，但是需要连接两个不同的端口，不方便
实验二：实现了读写分离，但读负载均衡不能实现，只需要链接一个端口即可，比较方便



思考：
1.假设程序用户是chenliang  对 chengliang业务库具有所有权，通过chenliang 这个角色能否做到读写分离呢？
直接使用

``` asciidoc
[root@localhost etc]# mysql -uchenliang -pESBecs00 -h192.168.6.121 -P4006  -e "create table tt2 as select * from tt" chenliang;
[root@localhost etc]# mysql -uchenliang -pESBecs00 -h192.168.6.121 -P4008  -e "show tables" chenliang;
Warning: Using a password on the command line interface can be insecure.
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test119             |
| tt                  |
| tt2                 |
+---------------------+
[root@localhost etc]# mysql -uchenliang -pESBecs00 -h192.168.6.121 -P4008  -e "show tables" chenliang;                                         
Warning: Using a password on the command line interface can be insecure.
+---------------------+
| Tables_in_chenliang |
+---------------------+
| test115             |
| tt                  |
| tt2                 |
+---------------------+
```


完美

2 若直接将程序用户写到配置文件的读写分离部分可以吗？

``` ini
[Read-Only Service]
type=service
router=readconnroute
servers=server1,server2,server3
user=chenliang
passwd=ESBecs00
router_options=slave
```


>  https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Routers/ReadWriteSplit.md

``` ini
[Read-Write Service]
type=service
router=readwritesplit
servers=server1
user=chenliang
passwd=ESBecs00
max_slave_connections=100%
```


重新启动观察日志会报错：

``` sql
or table 'user'
2016-07-29 17:40:44 warning: Read-Only Service: User 'chenliang' is missing SELECT privileges on mysql.db table. Database name will be ignored in authentication. MySQL error message: SELECT command denied to user 'chenliang'@'192.168.6.121' for table 'db'
2016-07-29 17:40:44 error : Read-Only Service: Inadequate user permissions for service. Service not started.
2016-07-29 17:40:44 error : Failed to start service 'Read-Only Service'.
2016-07-29 17:40:44 error : Read-Write Service: User 'chenliang' is missing SELECT privileges on mysql.user table. MySQL error message: SELECT command denied to user 'chenliang'@'192.168.6.121' for table 'user'
2016-07-29 17:40:44 warning: Read-Write Service: User 'chenliang' is missing SELECT privileges on mysql.db table. Database name will be ignored in authentication. MySQL error message: SELECT command denied to user 'chenliang'@'192.168.6.121' for table 'db'
2016-07-29 17:40:44 error : Read-Write Service: Inadequate user permissions for service. Service not started.
2016-07-29 17:40:44 error : Failed to start service 'Read-Write Service'.
2016-07-29 17:40:44 notice : Loaded module maxscaled: V1.0.0 from /usr/lib64/maxscale/libmaxscaled.so
```


很明显：读写分离的配置用户需要对mysql.user表具有查询权限
建议：为了省事就将[Read-Only Service][Read-Write Service][MySQL Monitor]部分用同一个账户算了（非应用程序user），可以加密处理，让别人看不到明文密码

