# 聚簇 VS 非聚簇

聚簇索引的优点：

1. 能够提高读取速度，因为不需要回表。每个聚簇索引唯一指向一行数据，可以直接得到需要查询的数据。
2. 相邻的行数据在页内连续存储。因此，范围查找、排序时，后续索引值的行在物理相邻，可以连续读取页上的行，效率更高。
3. 节约存储空间，因为聚簇索引不需要额外的索引结构，因此占用的空间较小。

非聚簇索引的优点：

1. 聚簇索引只能有一个，非聚簇可以有多个
2. 聚集索引存储记录是物理上连续存在，物理存储按照索引排序，而非聚集索引是逻辑上的连续。因次聚簇索引的插入、更新、删除操作可能较慢，因此需要涉及到数据的重新排序和移动。



# B+树 VS 哈希索引

B+树索引的优点：

1. 支持区间查找，也就是说在查找某个范围内的数据时，哈希索引不支持。
2. 可以通过 B+树的叶子节点顺序进行顺序遍历（ORDER BY），哈希索引不支持。
3. 支持单索引的最左匹配、联合索引最左匹配（LIKE '%'），哈希不支持。

哈希索引的优点：

1. 哈希索引的查询效率非常高，O(1) 的查询复杂度可以保证极快的查找速度。
2. 插入、更新或删除数据时，不会有节点分裂的问题。