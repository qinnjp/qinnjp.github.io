# MySQL面试题整理

#### 1.忘记管理员密码，怎么处理

关闭数据库

使用mysqld_safe带参数启动数据库

--skip-grant-tables     # 启动时不加载授权表

--skip-networking 	 # 本地连接，拒绝TCP/IP连接

启动命令

```bash
$ sudo /usr/local/mysql/bin/mysqld_safe --skip-grant-tables --skip-networking &
```

在mysql会话框加载授权表，并修改密码

```sql
flush privileges;
alter user root@'localhost' identified by '123';
```

重启服务

#### 2.drop truncate delete的区别？

| 语句              | 含义                                                         |
| ----------------- | ------------------------------------------------------------ |
| drop table t1     | 清除表结构和表数据                                           |
| truncate table t1 | 清空表数据，保留表结构，降低高水位线。立即释放空间           |
| delete from t1    | 逻辑删除（删除标记），不会降低高水位线。不会立即释放磁盘空间 |

#### 3. 5.7以后的SQL_mode=only_full_group_by

Expression #3 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'world.city.Name' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by

回答：

- 如果select之后的查询列表既不是group by的查询条件，又不在聚合函数中存在。就会和SQL_mode不兼容。如果group by 后的列是主键或者唯一键时，可以忽略。

- 原理：

  group by 执行过程

  取数据 ---> 对group by 之后的列进行排序和去重，在对其他列进行聚合，如果出现没有聚合的条件列值，结果集就会出现1个值对应多个值，出现不规则表结构

#### 4.information_schema.tables 案例

背景：

由于历史遗留问题，导致数据库中有部分表示myisam引擎。

问题：业务繁忙的时候，导致业务网站卡住，断电情况下，会有部分数据（索引损坏）丢失，主从1年多没有同步了。



分析问题：

1. 确认版本

   ```sql
   select version();
   ```

2. 确认业务表的引擎

   ```sql
   select TABLE_NAME,TABLE_SCHEMA,ENGINE from information_schema.tables where TABLE_SCHEMA not in ('mysql','sys','performance_schema','information_schema');
   
   -- 查询业务库下非innodb引擎的表
   select TABLE_NAME,TABLE_SCHEMA,ENGINE from information_schema.tables where TABLE_SCHEMA not in ('mysql','sys','performance_schema','information_schema') and ENGINE != 'InnoDB';
   ```

3. 监控锁的情况

   ```sql
   show status like '%lock%';
   -- 发现有很多table_lock信息
   ```

4. 检查主从状态

   ```sql
   show slave status \G;
   ```



处理方案：

- 将所有非innodb表查出来

  ```sql
  select TABLE_NAME,TABLE_SCHEMA,ENGINE from information_schema.tables where TABLE_SCHEMA not in ('mysql','sys','performance_schema','information_schema') and ENGINE != 'InnoDB';
  
  -- 修改数据库中非innodb引擎的表，为innodb
  -- 注意：生产环境更换表的引擎需要在测试环境测试
  
  -- 拼接修改表引擎语句
  select concat("alter table ",TABLE_SCHEMA, ".",TABLE_NAME," engine=innodb;") from information_schema.tables where TABLE_SCHEMA not in ('mysql','sys','performance_schema','information_schema') and ENGINE != 'InnoDB';
  
  -- 将语句导入文件
  select concat("alter table ",TABLE_SCHEMA, ".",TABLE_NAME," engine=innodb;") from information_schema.tables where TABLE_SCHEMA not in ('mysql','sys','performance_schema','information_schema') and ENGINE != 'InnoDB' into outfile '/tmp/alter.sql';
  
  -- 当执行这一条语句时会出现报错
  ERROR 1290 (HY000): The MySQL server is running with the --secure-file-priv option so it cannot execute this statement
  
  -- 解决方法，修改mysql配置文件在mysqld下面添加
  secure_file_priv='/tmp'
  
  -- 然后在mysql中运行刚才导出的文件
  source /tmp/alter.sql;
  ```

  