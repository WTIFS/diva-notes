**start as root**

```go
sudo mysqld --user=root
```



##### 查看DB大小

```
SELECT table_schema "DB Name",
        ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) "DB Size in MB" 
FROM information_schema.tables 
GROUP BY table_schema; 
```





##### 查看表大小

```mysql
select
table_schema as '数据库',
table_name as '表名',
table_rows as '记录数',
truncate(data_length/1024/1024, 2) as '数据容量(MB)',
truncate(index_length/1024/1024, 2) as '索引容量(MB)'
from information_schema.tables
order by data_length desc, index_length desc limit 10;
```





##### 查看 general_log 日志是否开启和存放目录 

general_log 会记录每个 sql 命令，可用于观测 mysql 操作记录。该日志会记录所有的数据库动作，增长幅度非常大，因此适合于在出现问题时临时开启一段时间，待问题排查解决后再进行关闭

```sql
show global variables like 'general%';
```



##### 开关 general_log
```sql
set global general_log=on;
set global general_log=off;
```



清理 binlog

```bash
PURGE BINARY LOGS BEFORE '2024-08-12 22:46:26';
```





##### dump

```mysql
# 导出
mysqldump -u 用户名 -p 密码 数据库名 表名 --where="column=''" > 文件名

# 导入
mysqldump -u 用户名 -p 密码 数据库名 < 文件名
```
