# ClickHouse



https://km.woa.com/group/35929/articles/show/441101?kmref=search&from_page=1&no=2





Kylin：Cube方式；较多的预处理伴随着较高的生产成本；不支持明细数据的查询；配置过程繁琐，需要配置模型设计，并配合适当的“剪枝”策略

ClickHouse：MPP架构的列式存储；ROLAP类型；有非标准SQL，有学习成本；每个表都需要手动分区分片，在维护的表较多时，日常维护成本昂贵

TiDB：MPP架构；支持行存（TiKV），列存（TiFlash）；HTAP（混合事务分析）类型；组件较多，安装和学习麻烦

Doris：MPP架构的行存；ROLAP类型；支持向量化计算；去重统计用bitmap，速度较快





调研文章：

[DorisDB,ClickHouse,TiDB的对比与选型分析](https://itinycheng.github.io/2021/07/11/comparison-and-selection-of-dorisdb-tidb-clickhouse/#%E5%AE%9E%E6%97%B6%E4%BA%BA%E7%BE%A4%E5%9C%88%E9%80%89%E5%9C%BA%E6%99%AF%E9%9C%80%E6%B1%82%E8%AF%B4%E6%98%8E)