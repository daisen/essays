MySQL复制就是一台MySQL服务器（slave）从另一台MySQL服务器（master）进行日志的复制然后再解析日志并应用到自身，类似Oracle中的Data Guard。
MySQL复制有那些好处：
•	第一是解决宕机带来的数据不一致，因为MySQL复制可以实时备份数据；
•	第二点是减轻数据库服务器的压力，多台服务器的性能一般比单台要好。但是MySQL复制不适合大数据量，大数据量推荐使用集群。
MySQL复制过程分成三步：
•	master将改变记录到二进制日志（binary log）。这些记录过程叫做二进制日志事件，binary log events；
•	slave将master的binary log events拷贝到它的中继日志（relay log）；
•	slave重做中继日志中的事件，将改变应用到自己的数据库中。 MySQL复制是异步的且串行化的
 MySQL复制原理图
MySQL复制搭建
________________________________________
环境准备
master 192.168.0.2
slave 192.168.0.3~6
OS：CentOs7
MySQL：5.7.17
Master配置
1）分配复制权限
主库和从库均需要执行
mysql> grant replication client,replication slave on *.* to root@'192.168.0.%' identified by 'Root@123';
Query OK, 0 rows affected (0.00 sec)
2）清空日志文件
主从库必须开启二进制日志文件
vim /etc/my.cnf
加入
[mysqld]
log-bin=mysql-bin
server-id=2
server-id设置为唯一号，该处未192.168.0.2的配置，使用了IP作为id号
重启MysSql
service mysqld restart
mysql> show binary logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |     26636 |
| mysql-bin.000002 |   1069399 |
| mysql-bin.000003 |     26636 |
| mysql-bin.000004 |   1069399 |
| mysql-bin.000005 |       536 |
+------------------+-----------+
5 rows in set (0.00 sec)

mysql> reset master;
Query OK, 0 rows affected (0.01 sec)

mysql> show binary logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       107 |
+------------------+-----------+
1 row in set (0.00 sec)
需要注意的是，如果不想清空日志文件的话，需要记录当前master的log_file和log_pos，并在下面启用复制操作时指定这两个参数或者在slave的配置文件指定。
Slave设置
1） 修改从服务器server-id
# vim /etc/my.cnf
server-id = 2
修改完以后需要重启数据库
2）清空日志文件，同Master
3）启用复制
让slave连接master并开始重做master二进制日志中的事件
mysql> change master to 
master_host='192.168.0.2',master_user='root',master_password='Root@123',master_port=3306,master_log_file='mysql-bin.000001',master_log_pos=0;
master_log_pos的值为0，因为它是日志的开始位置；master_log_file是初始日志文件。如果master日志没有被清空，这里就是当前master的日志信息
需要注意的是，默认情况下，会同步该用户下所有的DB，如果想限定哪些DB，有3种思路
•	在master上的/etc/my.inf中通过参数binlog-do-db、binlog-ignore-db设置需要同步的数据库。
•	在执行grant分配权限操作的时候，限定数据库
•	在slave上限定数据库使用replicate-do-db=dbname
4）开启slave
mysql> start slave;
Query OK, 0 rows affected (0.00 sec)
5）确认Slave是否和Mater成功通信。如果 Slave_IO_Running和Slave_SQL_Running都是yes，则证明配置成功
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.1.5
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 107
               Relay_Log_File: rac-node2-relay-bin.000002
                Relay_Log_Pos: 253
        Relay_Master_Log_File: mysql-bin.000001
            **Slave_IO_Running: Yes**
            **Slave_SQL_Running: Yes**
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 107
              Relay_Log_Space: 413
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
1 row in set (0.00 sec)
测试
Master创建数据库
mysql> create database d;
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| d                  |
| mysql              |
| performance_schema |
| test               |
+--------------------+
Slave查看数据库已同步
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| d                  |
| mysql              |
| performance_schema |
| test               |
+--------------------+
Master创建表插入数据
mysql> use d
mysql> create table t(id int);
mysql> insert into t values(1);
mysql> commit;
Slave查看
mysql> use d;
mysql> show tables;
mysql> select * from t;
+------+
| id   |
+------+
|    1 |
+------+
通过以上验证，可以看到主服务器上的修改能够正常同步到从服务器。
补充说明
1）做了MySQL主从复制以后，使用mysqldump对数据备份时，一定要注意按照如下方式：
mysqldump –master-data –single-transaction –user=username –password=password dbname> dumpfilename
这样就可以保留file和position的信息，在新搭建一个slave的时候，还原完数据库，file和position的信息也随之更新，接着再start slave 就可以很迅速的完成增量同步。
其他复制方式
________________________________________
主主复制
master-slave只能进行单向操作，像网络中的半双工。master-master可以实现服务器之间互相同步，且主主复制可以避免出现单点故障后整个系统宕机。主主复制最大的问题就是数据插入或更新冲突。配置方法同主从复制，反过来让slave同步master。注意master_log_file和master_log_pos参数要与master上对应。具体可以参考这里
单一master和多slave
由一个master和多个slave组成的复制系统比较简单。slave之间并不互相通信，只能与master通信。如果写操作较少，读操作很多，可以采用。可以将读操作分布到其他slave，从而减轻master的压力。但slave增加到一定数量时，slave对master的负载以及网络带宽都会成为问题。
主从多级复制
读操作很多可以采用单一maste和多slave，但增大到一定slave后连到master的slaveIO线程太多会造成master压力增大，从而造成数据复制延时。多级复制就是为了解决这个问题。如果想实现主-从（主）-从多级复制，需要设置log-slave-updates参数。同时二进制日志也必须启用。
 mysql多级复制
当然，增加复制的级联层次，同一个变更传到最底层的Slave所需要经过的MySQL也会更多，同样可能造成延时较长的风险。如果条件允许，倾向于通过拆分成多个Replication集群来解决。

