MySQL5.7 修改 root 密码


    5.7.6 以后的版本

MySQL5.7 开始，增加了很多安全性的更新。老版本的用户可能会有一些不习惯，这里介绍关于5.7版本的数据库密码问题。
5.7.6 以后的版本

5.7.6 以后的版本在启动数据库的时候，会生成密码放到日志文件里，像这样：
	
[root@centos-linux ~]# cat /var/log/mysqld.log | grep 'password'
2016-07-16T03:07:53.587995Z 1 [Note] A temporary password is generated for root@localhost: 2=s6NZk.t:fz

然后使用该密码登陆数据库，但是不能进行任何操作，提示需要先修改密码。
	
mysql> show databases;
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.

这里修改密码就会遇到验证，简单的密码会提示不符合规则
	
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '123';
ERROR 1819 (HY000): Your password does not satisfy the current policy requirements

因为5.7里引入了一个validate_password插件来检验密码强度。

默认值分别如下：
mysql> show variables like 'vali%';
+--------------------------------------+--------+
| Variable_name                        | Value  |
+--------------------------------------+--------+
| validate_password_dictionary_file    |        | 
| validate_password_length             | 8      | 
| validate_password_mixed_case_count   | 1      |
| validate_password_number_count       | 1      |
| validate_password_policy             | MEDIUM |
| validate_password_special_char_count | 1      |
+--------------------------------------+--------+
6 rows in set (0.01 sec)

意义如下：
	
validate_password_length
# 密码的最小长度，默认为8。
validate_password_mixed_case_count
# 至少要包含小写或大写字母的个数，默认为1。
validate_password_number_count
# 至少要包含的数字的个数，默认为1。
validate_password_policy 
# 强度等级，可设置为0、1、2。
    【0/LOW】：只检查长度。
    【1/MEDIUM】：在0等级的基础上多检查数字、大小写、特殊字符。
    【2/STRONG】：在1等级的基础上多检查特殊字符字典文件，此处为1。
validate_password_special_char_count
# 至少要包含的特殊字符的个数，默认为1。

所以初始设置密码比如大于8位，包含数字，大小写字母，特殊字符。

同时也可以修改上面这些配置减弱密码强度验证。

