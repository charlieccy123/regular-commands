# MySQL

## 命令

```bash
# 查看哪个进程或者网络连接使用了哪个库表，如果遇到lock 这种
# 其实这个命令就是读取系统 information_schema.processlist 表
show full processlist;
# 查看哪个主机连接了哪个库
select * from information_schema.processlist where DB like 'UCENTER';
# 就可以结束连接
kill ID;

# 慢查询
select id, db,time,info from information_schema.processlist where time > '1' and command != 'Sleep'\G ;


# 查看是否开启了慢日志以及位置
mysql> show variables like '%slow%';
# 最大连接数
mysql> show variables like 'max_connections';
# 显示当前连接数
mysql> show global status like 'max_used_connections';

# 授权
update mysql.user set host='%' where user='root';
flush privileges;

```

## 查看数据大小

```bash
# 查看所有数据的容量大小
select table_schema as '数据库', table_name as '表名', table_rows as '记录数', truncate(data_length/1024/1024, 2) as '数据容量(MB)', truncate(index_length/1024/1024, 2) as '索引容量(MB)' from information_schema.tables order by data_length desc, index_length desc;

echo "select table_schema as '数据库', table_name as '表名', table_rows as '记录数', truncate(data_length/1024/1024, 2) as '数据容量(MB)', truncate(index_length/1024/1024, 2) as '索引容量(MB)' from information_schema.tables order by data_length desc, index_length desc;" | mysql -uroot  -p'MhRbof&Mnl6U' -h10.50.2.242 > total.xls
```

## 安装客户端工具

```bash
# 安装client
wget http://repo.mysql.com/mysql57-community-release-el7-10.noarch.rpm
rpm -Uvh mysql57-community-release-el7-10.noarch.rpm
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
yum install -y mysql-community-client-5.7.38

# 如果全安装
yum install mysql-community-server-5.7.34 mysql-community-libs-compat-5.7.34 mysql-community-libs-5.7.34 mysql-community-client-5.7.34 mysql-community-common-5.7.34

```

## 创建账号

```sql
CREATE USER 'pumpkin'@'%' IDENTIFIED BY '123!@#QWE';  # 创建所有网段用户
CREATE USER 'pumpkin'@'192.168.%.%' IDENTIFIED BY '123!@#QWE';  # 创建指定网段
GRANT SELECT ON *.* TO 'pumpkin'@'128.1.156.34';  # 只读
GRANT ALL PRIVILEGES ON *.* TO 'pumpkin'@'192.168.%.%';  # 所有权限
FLUSH PRIVILEGES;
```