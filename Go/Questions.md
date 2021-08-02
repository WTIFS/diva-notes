##### 带缓冲 channel 是怎么判断为空的

`channel` 内部有个 `qcount` 字段，记录了队列内的元素个数。判断 `qcount == 0` 即可



