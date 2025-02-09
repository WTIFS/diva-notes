##### 查看 general_log 日志是否开启和存放目录 

general_log 会记录每个 sql 命令，可用于观测 mysql 操作记录。

```sql
show global variables like 'general%';
```



##### 开关 general_log
```sql
set global general_log=on;
set global general_log=off;
```