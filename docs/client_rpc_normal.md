# 概念
Channel

# 内存布局
以brpc自带的实例程序example/multi_threaded_echo_c++/client.cpp为例，结合Client端内存布局的变化过程，讲述无异常状态下的Client发送请求直到处理响应的过程。

这个程序的具体运行过程为：
1. 在main函数的栈上创建Channel，并创建N个bthread（后续设N=3），每个bthread的task函数指向static类型函数sender；
2. 惰性初始化TaskControl单例对象与多个TaskGroup对象，每个TaskGroup对应一个系统线程pthread，是pthread的线程私有对象，pthread启动后，阻塞在TaskGroup的代码处，等待任务；
3. 
