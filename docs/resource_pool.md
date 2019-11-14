## 概述

柔性数组
分配pool时需要按cacheline对齐  考虑局部性原理

double check的几处使用

ResourcePoolBlockItemNum  BLOCK_NITEM   一个block里面容纳Item的数量
FREE_CHUNK_NITEM=BLOCK_NITEM

一个Block  64KB

RP_GROUP_NBLOCK  一个BlockGroup中的Block*的数量

