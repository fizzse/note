## Golang GC

### 标记清除
原理：
- GC开始时，进行STW(stop the world),标记所有不可达的对象(没有被使用)
- 进行清除，关闭STW

缺点：
- STW时间过长
- 扫描整个堆栈信息
- 内存碎片化严重(被清理对象之间内存不连续)

### 三色标记清除+混合写屏障

优化方向：
- 尽可能的缩短STW时间，甚至完全不使用STW

三色原理：
- GC开始时，将所有对象标记为白色
- 从根对象开始展开进行遍历(只遍历一次，不进行递归)，将可达对象标记为灰色
- 遍历所有灰色对象，将灰色对象标记为黑色。并将可达对象标记为灰色。逐层进行遍历
- 最后只有黑色的可达对象与白色的不可达对象，将白色不可达对象进行清除

混合写凭证：
- xxx


### 触发时机