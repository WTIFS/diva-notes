# 布谷鸟过滤器

不储存元素的原始值，而是存其指纹信息 `fingerprint`。

将元素转为 `hash`。每个元素计算 `2` 个 `hash` 作为下标。一个主位置，一个备选位置。

插入元素时，如果主位置已经被占了，则将占用的元素 `A` 踢到 `A` 的备选位置上，如果备选位置上还有元素 `B`，再把 `B` 踢到 `B` 备选位置上，依次进行直到成功或到达次数上线。

如果到达次数上限，则说明该扩容了。

`Redisbloom` 里的布谷鸟过滤器还支持多次重复添加/删除一个元素，做法是使用多组桶 `filter`。

```c
//插入部分代码
static CuckooInsertStatus Filter_KOInsert(CuckooFilter *filter, SubCF *curFilter, const LookupParams *params) {

    while (counter++ < maxIterations) {
        
        uint8_t *bucket = &curFilter->data[ii * bucketSize]; // 计算元素应放入的bucket
        swapFPs(bucket + victimIx, &fp);                     // 将bucket的头元素和当前元素互换
        ii = getAltHash(fp, ii) % numBuckets;                // 为换出来的头元素（这里叫受害者）找下一个位置
        
        // Insert the new item in potentially the same bucket
        uint8_t *empty = Bucket_FindAvailable(&curFilter->data[ii * bucketSize], bucketSize);  // 下一个位置可以在本bucket内
        if (empty) { // 如果下个位置是空的，则直接插入
            *empty = fp;
            return CuckooInsert_Inserted;
        }
        
        victimIx = (victimIx + 1) % bucketSize;              // 如果换出来的受害者没有地方去了，换下一个受害者尝试
    }

    // If we weren't able to insert, we roll back and try to insert new element in new filter
    // 超过maxIterations会触发扩容，扩容后这里再试一次
    // 扩容会加个 filter
    // ...

    return CuckooInsert_NoSpace;
}
```

