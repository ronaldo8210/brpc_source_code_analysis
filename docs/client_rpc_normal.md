# 概念
Channel

# 内存布局
以brpc自带的实例程序example/multi_threaded_echo_c++/client.cpp为例，结合Client端内存布局的变化过程，讲述无异常状态下的Client发送请求直到处理响应的过程。

这个程序的具体运行过程为：
1. 在main函数的栈上创建Channel，并创建N个bthread（后续设N=3），每个bthread的task函数指向static类型函数sender；
2. 惰性初始化下列全局对象：

   a. 一个TaskControl单例对象；
   
   b. 多个TaskGroup对象，每个TaskGroup对应一个系统线程pthread，是pthread的线程私有对象，每个pthread启动后执行TaskGroup的run_main_task函数，该函数是个无限循环，不停地获取bthread、执行bthread任务；
   
   c. 一个定时器对象。
   
3. 
