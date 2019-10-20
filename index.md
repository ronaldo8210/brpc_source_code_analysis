## 目录

* [线程模型](thread_model.md)
  * [client](client.md)
  * [server](server.md)
* bthread 
  * 调度算法
  * [同一RPC过程中各个bthread间的同步](bthread_sync.md)
    * [线程futex同步](futex.md)
* 容灾容错
  * 请求重试
  * 服务限流
  * 防雪崩
* 性能监控
* 基础库
  * 对象池
  * 定时器
  * 计数器
