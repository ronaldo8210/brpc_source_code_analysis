[多线程下内存分配与回收的设计原理](#多线程下内存分配与回收的设计原理)

[brpc中ResourcePool的源码实现](#brpc中ResourcePool的源码实现)

[多线程下内存分配与回收的具体示例](#多线程下内存分配与回收的具体示例)

## 多线程下内存分配与回收的设计原理


## brpc中ResourcePool的源码实现


## 多线程下内存分配与回收的具体示例

多线程环境下高效的内存分配与回收

柔性数组
分配pool时需要按cacheline对齐  考虑局部性原理

double check的几处使用

ResourcePoolBlockItemNum  BLOCK_NITEM   一个block里面容纳Item的数量
FREE_CHUNK_NITEM=BLOCK_NITEM

一个Block  64KB

RP_GROUP_NBLOCK  一个BlockGroup中的Block*的数量

fetch_add  返回原子对象的旧值

_cur_block_index  block在全局中的索引号

