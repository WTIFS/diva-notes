##### 远程连接
```bash
psql -h IP地址 -p 端口  -U 数据库名
# 之后会要求输入数据库密码
```



##### 查看安装位置

```bash
psql -U postgres -c 'SHOW config_file'
```



##### 安装

CentOS7
```bash
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
# 客户端
sudo yum install -y postgresql12 
# 服务器
sudo yum install -y postgresql12-server
```



##### 更改默认的schema

```sql
show search_path;
create schema my_schema;
grant all on schema my_schema to my_user;

-- 会话内修改
set search_path to yay;

-- 永久修改
alter database "databasename" set search_path to "yay";
```



##### 访问数据库
| 功能          | 命令                    |
| ------------- | ----------------------- |
| 列举数据库    | `\l`                    |
| 选择数据库    | `\c 数据库名`           |
| 查看所有表    | `\dt` 或 `\dt+`         |
| 查看表结构    | `\d 表名` 或 `\d+ 表名` |
| 显示字符集    | `\encoding`             |
| 退出psgl      | `\q`                    |
| 选取所有enums | `\dT+`                  |
| 展示空值      | `\pset null 'NULL'`     |
| 关闭换行      | `\pset pager off`       |



##### 查看空间：
```sql
select pg_size_pretty(pg_database_size('库名'));
select pg_size_pretty(pg_relation_size('表名'));
select pg_size_pretty(pg_indexes_size('索引名'));

-- 查看top空间占用表
select relname, indexrelname, pg_size_pretty(pg_relation_size(relid)) as table_size, pg_size_pretty(pg_relation_size(indexrelid)) as idx_size, idx_scan, idx_tup_read, idx_tup_fetch from pg_stat_user_indexes where pg_relation_size(indexrelid)>100000 order by relname, pg_relation_size(indexrelid) desc;
select relname, (table_size/rows + index_size)::bigint as table_raw, pg_size_pretty(table_size/rows + index_size) as total_size, ((table_size/rows + index_size)::numeric/10.24/(25001024)/1024)::numeric(8,2) as rate,
       pg_size_pretty(table_size/rows) as table_size,
       pg_size_pretty(index_size) as index_size
from (
         select relname,
                sum(pg_relation_size(relid) ) as table_size,
                sum(pg_relation_size(indexrelid) ) as index_size,
                count(1) AS rows
         from pg_stat_user_indexes
         group by relname
     ) a
where table_size>5001024*1024 
order by table_size/rows + index_size desc;

-- 查看特定表空间占用
select relname, (table_size/rows + index_size)::bigint as table_raw, pg_size_pretty(table_size/rows + index_size) as total_size, ((table_size/rows + index_size)::numeric/10.24/(2500*1024)/1024)::numeric(8,2) as rate,
       pg_size_pretty(table_size/rows) as table_size,
       pg_size_pretty(index_size) as index_size
from (
         select relname,
                sum(pg_relation_size(relid) ) as table_size,
                sum(pg_relation_size(indexrelid) ) as index_size,
                count(1) AS rows
         from pg_stat_user_indexes
         group by relname
     ) a
where relname='latest_app_open_device';

-- 查看所有索引的大小
select indexrelname, pg_size_pretty(pg_relation_size(indexrelid))
from pg_stat_user_indexes
where schemaname = 'yay'
order by pg_relation_size(indexrelid) desc
```



# DDL

##### 索引重命名

```sql
alter index if exists old_name rename to new_index_name;
```



##### 导出

```bash
# 导出csv
psql -c "
    \COPY (
      SELECT *
      FROM products
    )
    TO '/path/to/output.csv'
    WITH (format csv, header);
"

# 导出txt
# 保存以下命令为bash脚本
table=$1
column=*
where=$2
psql -h slave.marketdb1.tt -d putong-market -p 5432 -U postgres -c "\copy (select $column from $table where 1=1 $2) to 'txt/$table.txt' with (delimiter '|');"

scp txt/$table.txt pj:~/psql/txt/

# 执行上面脚本 + 表名 + where条件
```



##### 导入

```bash
# 保存以下命令为bash脚本
psql -d putong-market -U putong -c "truncate $1"
psql -d putong-market -U putong -c "\copy $1 from 'txt/$1.txt' with (delimiter '|');"

# 执行以上脚本 + 表名
```



##### 导出表结构

```bash
pg_dump -h localhost  -p 5432 -U postgres -n yay -s $table -t $table > txt/$table.sql
```



##### 清理空间

```sql
vacuum
vacuum full (锁全表，清理的更深度，归还磁盘空间)
```



##### 设置pg_xlog大小

修改 `postgres.conf` 里 `wal` 开头的东西



##### 查看正在执行的sql

```sql
SELECT
procpid,
start,
now() - start AS lap,
current_query
FROM
(SELECT
backendid,
pg_stat_get_backend_pid(S.backendid) AS procpid,
pg_stat_get_backend_activity_start(S.backendid) AS start,
pg_stat_get_backend_activity(S.backendid) AS current_query
FROM
(SELECT pg_stat_get_backend_idset() AS backendid) AS S
) AS S
WHERE
current_query <> '<IDLE>'
ORDER BY
lap DESC;
```



##### 修改编辑模式

```
control + command + j
```



##### 在脚本中自动输入密码

1. 在~/目录下创建隐藏文件 `.pgpass`
2. 文件内容：`host:port:dbname:username:password`
3. `bash_profile` 里 `export PGPASSFILE=~/.pgpass`



##### 修改表结构

```sql
ALTER TABLE
Name
ALTER TABLE -- change the definition of a table
Synopsis
ALTER TABLE [ ONLY ] name [ * ]
    action [, ... ]
ALTER TABLE [ ONLY ] name [ * ]
    RENAME [ COLUMN ] column TO new_column
ALTER TABLE name
    RENAME TO new_name
ALTER TABLE name
    SET SCHEMA new_schema
where action is one of:
    ADD [ COLUMN ] column data_type [ COLLATE collation ] [ column_constraint [ ... ] ]
    DROP [ COLUMN ] [ IF EXISTS ] column [ RESTRICT | CASCADE ]
    ALTER [ COLUMN ] column [ SET DATA ] TYPE data_type [ COLLATE collation ] [ USING expression ]
    ALTER [ COLUMN ] column SET DEFAULT expression
    ALTER [ COLUMN ] column DROP DEFAULT
    ALTER [ COLUMN ] column { SET | DROP } NOT NULL
    ALTER [ COLUMN ] column SET STATISTICS integer
    ALTER [ COLUMN ] column SET ( attribute_option = value [, ... ] )
    ALTER [ COLUMN ] column RESET ( attribute_option [, ... ] )
    ALTER [ COLUMN ] column SET STORAGE { PLAIN | EXTERNAL | EXTENDED | MAIN }
    ADD table_constraint [ NOT VALID ]
    ADD table_constraint_using_index
    VALIDATE CONSTRAINT constraint_name
    DROP CONSTRAINT [ IF EXISTS ]  constraint_name [ RESTRICT | CASCADE ]
    DISABLE TRIGGER [ trigger_name | ALL | USER ]
    ENABLE TRIGGER [ trigger_name | ALL | USER ]
    ENABLE REPLICA TRIGGER trigger_name
    ENABLE ALWAYS TRIGGER trigger_name
    DISABLE RULE rewrite_rule_name
    ENABLE RULE rewrite_rule_name
    ENABLE REPLICA RULE rewrite_rule_name
    ENABLE ALWAYS RULE rewrite_rule_name
    CLUSTER ON index_name
    SET WITHOUT CLUSTER
    SET WITH OIDS
    SET WITHOUT OIDS
    SET ( storage_parameter = value [, ... ] )
    RESET ( storage_parameter [, ... ] )
    INHERIT parent_table
    NO INHERIT parent_table
    OF type_name
    NOT OF
    OWNER TO new_owner
    SET TABLESPACE new_tablespace
and table_constraint_using_index is:
    [ CONSTRAINT constraint_name ]
    { UNIQUE | PRIMARY KEY } USING INDEX index_name
    [ DEFERRABLE | NOT DEFERRABLE ] [ INITIALLY DEFERRED | INITIALLY IMMEDIATE ]
```



##### 创建用户

```sql
CREATE USER name [ [ WITH ] option [ ... ] ]
where option can be:
      SUPERUSER | NOSUPERUSER
    | CREATEDB | NOCREATEDB
    | CREATEROLE | NOCREATEROLE
    | INHERIT | NOINHERIT
    | LOGIN | NOLOGIN
    | REPLICATION | NOREPLICATION
    | BYPASSRLS | NOBYPASSRLS
    | CONNECTION LIMIT connlimit
    | [ ENCRYPTED ] PASSWORD 'password' | PASSWORD NULL
    | VALID UNTIL 'timestamp'
    | IN ROLE role_name [, ...]
    | IN GROUP role_name [, ...]
    | ROLE role_name [, ...]
    | ADMIN role_name [, ...]
    | USER role_name [, ...]
    | SYSID uid
```




##### 重置自增ID

```
SELECT setval('xxx_id_seq', (SELECT max(id) FROM xxx));
```



# DML

##### 列转行

```sql
select array(select column from table);
```



##### 联表更新

```sql
UPDATE table1 SET xxx FROM table2 WHERE table1 关联 table2
```



##### 排序

```sql
RANK() OVER(PARTITION BY f1 ORDER BY f2 DESC)
```



##### 提取时间差
```sql
-- 提取秒
extract(EPOCH from (a.created_time-b.click_time))

-- 提取日期
date_part('day' from (a.created_time-b.click_time))
```



##### 字符串拼接

```sql
SELECT concat(a, b, c) FROM table WHERE xxx;
```



##### 字符串替换

```sql
SELECT replace(column, old, new) FROM table WHERE xxx;
```

