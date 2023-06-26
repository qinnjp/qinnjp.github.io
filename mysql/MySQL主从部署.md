### 安装MySQL5.7

配置清华源（所有节点）

```bash
cat > /etc/yum.repos.d/mysql.repo <<-'EOF'
[mysql-connectors-community]
name=MySQL Connectors Community
baseurl=https://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql-connectors-community-el7-$basearch/
enabled=1
gpgcheck=1
gpgkey=https://repo.mysql.com/RPM-GPG-KEY-mysql

[mysql-tools-community]
name=MySQL Tools Community
baseurl=https://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql-tools-community-el7-$basearch/
enabled=1
gpgcheck=1
gpgkey=https://repo.mysql.com/RPM-GPG-KEY-mysql

[mysql-5.7-community]
name=MySQL 5.7 Community Server
baseurl=https://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql-5.7-community-el7-$basearch/
enabled=1
gpgcheck=1
gpgkey=https://repo.mysql.com/RPM-GPG-KEY-mysql
EOF
```

安装配置mysql（所有节点）

```bash
$ yum install mysql-server -y
```

修改mysql配置文件（所有节点）

```bash
cat > /etc/my.cnf <<-'EOF'
[client]
port		= 3306
socket = /opt/mysql/mysql.sock

[mysql]
auto_rehash
socket= /opt/mysql/mysql.sock
safe_updates

[mysqld_safe]
open_files_limit = 65535

[mysqldump]
port= 3306
socket= /opt/mysql/mysql.sock
default_character_set = utf8
quick
max_allowed_packet = 500M

[myisamchk]
key_buffer_size = 64M
sort_buffer_size = 2M
read_buffer = 2M
write_buffer = 2M

[mysqld]
port		= 3306
socket= /opt/mysql/mysql.sock
datadir = /opt/mysql/data
default_storage_engine = InnoDB

# ----------------------------------------------
# Enable the binlog for replication & CDC
# ----------------------------------------------

# log_bin           = mysql-bin
log_bin = /opt/mysql/mysql-bin
binlog_format = row
server-id = 1 # 从库必须修改server-id值
expire_logs_days = 30
slow_query_log=1
slow_query_log_file = /opt/mysql/slow.log
long_query_time=3
log_queries_not_using_indexes = 1
binlog_row_image  = full
log_error = /opt/mysql/mysqld.log

# ----------------------------------------------
# General configuration
# ----------------------------------------------
character_set_server = utf8
collation-server=utf8_general_ci
event_scheduler = on
skip_name_resolve = 1
symbolic_links = 0
lower_case_table_names = 1
interactive_timeout = 600
wait_timeout = 600
group_concat_max_len = 1024000
log_timestamps = SYSTEM
max_allowed_packet = 16MB
table_open_cache = 2048


# ----------------------------------------------
# Innodb
# ----------------------------------------------
innodb_buffer_pool_size = 16G
# innodb_additional_mem_pool_size = 1M
innodb_log_buffer_size = 1M

key_buffer_size = 64M
query_cache_size = 64M
tmp_table_size = 32M
max_connections = 1000
sort_buffer_size = 2M
read_buffer_size = 128K
read_rnd_buffer_size = 256K
join_buffer_size = 128K
thread_stack = 196K
binlog_cache_size = 256K

# ----------------------------------------------
# Enable GTIDs on this master 使用gtid搭建主从必须打开gtid模式，主库从库都需要配置
# ----------------------------------------------
gtid_mode                 = on
enforce_gtid_consistency  = on
log_slave_updates = 1

EOF
```

启动并初始化mysql（所有节点）

```bash
$ systemctl start mysqld

# 过滤密码
grep pass /opt/mysql/mysqld.log
2023-05-08T19:44:44.972644+08:00 1 [Note] A temporary password is generated for root@localhost: gq.:_>og(63K

# 登录MySQL修改密码
ALTER USER user() identified by 'xxx';

# 创建主从复制用户，授权   主库操作
create user repl@'%' identified by 'PM$BZivz8F5i';
grant replication slave on *.* to repl@'%';
```

从库配置主从

```bash
# GTID主从
CHANGE MASTER TO 
MASTER_HOST='192.168.20.203', 
MASTER_USER='repl', 
MASTER_PASSWORD='PM$BZivz8F5i', 
MASTER_AUTO_POSITION=1;
# binglog主从
CHANGE MASTER TO
MASTER_HOST='172.16.109.44',
MASTER_USER='repl',
MASTER_PASSWORD='PM$BZivz8F5i',
MASTER_PORT=3306,
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=154,
MASTER_CONNECT_RETRY=10;

启动主从
start slave;
```

 验证主从同步

```bash
# 从库执行
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.109.42
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: mysql-bin.000007
          Read_Master_Log_Pos: 34690071
               Relay_Log_File: dbslave-relay-bin.000020
                Relay_Log_Pos: 34690284
        Relay_Master_Log_File: mysql-bin.000007
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 34690071
              Relay_Log_Space: 34690540
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
                  Master_UUID: e3b17839-c47a-11ed-a1d2-005056913cdf
             Master_Info_File: /opt/mysql/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```

 