 
  1 MaxScale 是干什么的？
 
配置好了MySQL的主从复制结构后，我们希望实现读写分离，把读操作分散到从服务器中，并且对多个从服务器能实现负载均衡。
 
读写分离和负载均衡是MySQL集群的基础需求，MaxScale 就可以帮着我们方便的实现这些功能。
 
 
 
  2 MaxScale 的基础构成
 
MaxScale 是MySQL的兄弟公司 MariaDB 开发的，现在已经发展得非常成熟。MaxScale 是插件式结构，允许用户开发适合自己的插件。
 
MaxScale 目前提供的插件功能分为5类：
 
•	认证插件
提供了登录认证功能，MaxScale 会读取并缓存数据库中 user 表中的信息，当有连接进来时，先从缓存信息中进行验证，如果没有此用户，会从后端数据库中更新信息，再次进行验证
•	协议插件
包括客户端连接协议，和连接数据库的协议
•	路由插件 
决定如何把客户端的请求转发给后端数据库服务器，读写分离和负载均衡的功能就是由这个模块实现的
•	监控插件
对各个数据库服务器进行监控，例如发现某个数据库服务器响应很慢，那么就不向其转发请求了
•	日志和过滤插件
提供简单的数据库防火墙功能，可以对SQL进行过滤和容错
 
  3 MaxScale 的安装使用
 
例如有 3 台数据库服务器，是一主二从的结构。
 
  过程概述
（1）配置好集群环境
（2）下载安装 MaxScale
（3）配置 MaxScale，添加各数据库信息
（4）启动 MaxScale，查看是否正确连接数据库
（5）客户端连接 MaxScale，进行测试
 
  详细过程
（1）配置一主二从的集群环境
准备3台服务器，安装MySQL，配置一主二从的复制结构。
 
（2）安装 MaxScale
最好在另一台服务器上安装，如果资源不足，可以和某个MySQL放在一起。
 
MaxScale 的下载地址：
https://downloads.mariadb.com/files/MaxScale
 
根据自己的服务器选择合适的安装包。
 
以 centos 7 为例 安装步骤如下：
yum install libaio.x86_64 libaio-devel.x86_64 novacom-server.x86_64 libedit -y
rpm -ivh maxscale-1.4.3-1.centos.7.x86_64.rpm
 
（3）配置 MaxScale
 
在开始配置之前，需要在 master 中为 MaxScale 创建两个用户，用于监控模块和路由模块。
 
创建监控用户
mysql> create user scalemon@'%' identified by "111111";
mysql> grant replication slave, replication client on *.* to scalemon@'%';
 
创建路由用户 
mysql> create user maxscale@'%' identified by "111111";
mysql> grant select on mysql.* to maxscale@'%';
 
用户创建完成后，开始配置
vi /etc/maxscale.cnf
 
找到 [server1] 部分，修改其中的 address 和 port，指向 master 的 IP 和端口。
 
复制2次 [server1] 的整块儿内容，改为 [server2] 与 [server3]，同样修改其中的 address 和 port，分别指向 slave1 和 slave2：
 
 
 
找到 [MySQL Monitor] 部分，修改 servers 为 server1,server2,server3，修改 user 和 passwd 为之前创建的监控用户的信息（scalemon,111111）。
 
 
 
找到 [Read-Write Service] 部分，修改 servers 为 server1,server2,server3，修改 user 和 passwd 为之前创建的路由用户的信息（maxscale,111111）。
 
 
 
由于我们使用了 [Read-Write Service]，需要删除另一个服务 [Read-Only Service]，删除其整块儿内容即可。
 
配置完成，保存并退出编辑器。
 
（4）启动 MaxScale
 
执行启动命令 
maxscale --config=/etc/maxscale.cnf
 
查看 MaxScale 的响应端口是否已经就绪
netstat -ntelp
 
 
 
•	4006 是连接 MaxScale 时使用的端口
•	6603 是 MaxScale 管理器的端口，需增加配置，本机无需端口号访问
 
登录 MaxScale 管理器，查看一下数据库连接状态，默认的用户名和密码是 admin/mariadb。
maxadmin --user=admin --password=mariadb
 
MaxScale> list servers
 
 
 
可以看到，MaxScale 已经连接到了 master 和 slave。
 
（5）测试
 
先在 master 上创建一个测试用户
mysql> grant ALL PRIVILEGES on *.* to rtest@"%" Identified by "111111";
 
使用 Mysql 客户端到连接 MaxScale
mysql -h MaxScale所在的IP -P 4006 -u rtest -p111111
 
执行查看数据库服务器名的操作来知道当前实际所在的数据库：
 下面的结果在新的maxscale上无法重新
 
 
开启事务后，就自动路由到了 master，普通的查询操作，是在 slave上
MaxScale 的配置完成了。
 
  4 MaxScale 在 slave 有故障后的处理
 
前面已经介绍了 MaxScale可以实现MySQL的读写分离和读负载均衡，那么当 slave 出现故障后，MaxScale 会如何处理呢？
 
例如有 3 台数据库服务器，一主二从的结构，数据库名称分别为 master, slave1, slave2。
 
现在我们实验以下两种情况：
（1）当一台从服务器（ slave1 或者 slave2 ）出现故障后，查看 MaxScale 如何应对，及故障服务器重新上线后的情况
（2）当两台从服务器（ slave1 和 slave2 ）都出现故障后，查看 MaxScale 如何应对，及故障服务器重新上线后的情况
 
准备
 
为了更深入的查看 MaxScale 的状态，需要把 MaxScale 的日志打开：
 
修改配置文件
vi /etc/maxscale.cnf
 
 找到 [maxscale] 部分，这里用来进行全局设置，在其中添加日志。
 
配置 
log_info=1
logdir=/tmp/
通过开启 log_info 级别，可以看到 MaxScale 的路由日志。
 
修改配置后，重启 MaxScale 。
 
实验过程
 
1 单个 slave 故障的情况
 
 初始状态是一切正常。
 
 
 
停掉 slave2 的复制，登录 slave2 的 mysql 执行。
mysql> stop slave;
 
查看 MaxScale 服务器状态
 
 
 
slave2 已经失效了。
 
查看日志信息
cat /tmp/maxscale1.log
 
尾部显示：
2016-08-15 12:26:02   notice : Server changed state: slave2[172.17.0.4:3306]: lost_slave
 
提示 slave2 已经丢失。
 
查看客户端查询结果：
 
 
 
查询操作全都转到了 slave1。
 
可以看到， 在有 slave 故障后，MaxScale 会自动进行排除，不再向其转发请求。
 
下面看下 slave2 再次上线后的情况。
 
登录 slave2 的 MySQL 执行
mysql> start slave;
 
查看 MaxScale 服务器状态
 
 
 
恢复了正常状态，重新识别到了 slave2。
 
查看日志信息，显示：
2016-08-15 12:32:36   notice : Server changed state: slave2[172.17.0.4:3306]: new_slave
 
查看客户端查询结果：
 
 
 
slave2 又可以正常接受查询请求。
 
通过实验可以看到，在部分 slave 发生故障时，MaxScale 可以自动识别出来，并移除路由列表，当故障恢复重新上线后，MaxScale 也能自动将其加入路由，过程透明。
 
2 全部 slave 故障的情况
 
分别登陆 slave1 和 slave2 的MySQL，执行停止复制的命令
mysql> stop slave;
 
查看 MaxScale 服务器状态
 
 
 
发现各个服务器的角色都识别不出来了。
 
查看日志：
 
 
从日志中看到，MaxScale 发现2个slave 和 master 都丢了，然后报错：没有 master 了。
 
客户端连接 MaxScale 时也失败了。
 
 
 
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
 
 
 
可以看到，虽然 slave 都无法识别了，但 master 还在，并提示处于稳定状态
客户端执行请求：
 
 
 
客户端可以连接 MaxScale，而且请求都转到了 master 上，说明 slave 全部失效时，由 master 支撑了全部请求。
 
当恢复两个 slave 后，整体状态自动恢复正常，从客户端执行请求时，又可以转到 slave 上。
 
 
 
  小结
 
通过测试发现，在部分 slave 故障情况下，对于客户端是完全透明的，当全部 slave 故障时，经过简单的配置，MaxScale 也可以很好地处理。

