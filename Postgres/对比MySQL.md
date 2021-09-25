1. `Postgres` 有非常丰富的统计函数和统计语法支持。比如窗口函数 `window` ，还可以用多种语言来写存储过程，对于 `Madlib、R` 的支持也很好。这一点上 `MySQL` 就差的很远，很多分析功能都不支持。
2. 扩展性方面，`Postgres` 比 `Mysql` 也出色许多，`Postgres` 天生就是为扩展而生的，你可以在 `PG` 中用 `Python、C、Perl、TCL、PLSQL` 等等语言来扩展功能。另外，开发新的功能模块、新的数据类型、新的索引类型等等非常方便，只要按照 API 接口开发，无需对 PG 重新编译。PG 中 `contrib` 目录下的有很多第三方模块，如 `postgis` 空间数据库、`R`、`Madlib`、`pgcrypto` 各类加密算法、`gptext` 全文检索都是通过这种方式实现功能扩展的。
3. `Postgresql` 许可是仿照BSD许可模式的，没有被大公司控制，社区比较纯洁，版本和路线控制非常好，可让用户拥有更多自主性。反观 `MySQL` 的社区现状和众多分支（如MariaDB），确实够乱的。



#### 参考

[小小北漂 - 聊聊Greenplum的那些事](https://blog.csdn.net/u014589856/article/details/78744930)

