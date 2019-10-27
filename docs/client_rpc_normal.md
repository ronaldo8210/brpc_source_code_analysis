# 概念
Channel

# 内存布局
以brpc自带的实例程序example/multi_threaded_echo_c++/client.cpp为例，结合Client端内存布局的变化过程，讲述无异常状态下的Client发送请求直到处理响应的过程。

这个程序的具体运行过程为：
1. 在main函数的栈上创建Channel；
2. 在heap内存上惰性初始化下列全局对象：

   a. 一个TaskControl单例对象；
   
   b. 多个TaskGroup对象，每个TaskGroup对应一个系统线程pthread，是pthread的线程私有对象，每个pthread启动后执行TaskGroup的run_main_task函数，该函数是个无限循环，不停地获取bthread、执行bthread任务；
   
   c. 一个定时器对象。
   
3. 在TaskMeta对象池中创建N个TaskMeta对象（每个TaskMeta相当于一个bthread）（后续设N=3），每个TaskMeta的fn函数指针指向static类型函数sender，sender就是bthread的任务处理函数。每个TaskMeta创建完后，按照散列规则将其唯一标识tid压入一个TaskGroup对象的_remote_rq队列中。（TaskGroup所属的pthread线程称为worker线程，worker线程自己产生的bthread的tid会被压入_rq队列，这个实例中的main函数所在线程不属于worker线程，所以main函数的线程生成的bthread的tid会被压入TaskGroup的_rq队列）；

4. main函数执行到这里，不能直接结束，需要等待N个bthread任务全部执行完成后，才能结束。等待的实现机制是将main函数所在线程的信息存储在ButexPthreadWaiter中，并加入到bthread对应的TaskMeta的Butex的waiters队列中，等到TaskMeta的任务函数fn执行结束后，从waiters中找到等待中的pthread线程，将其唤醒。main函数所在的系统线程在join第一个bthread 1的时候就被挂起，等待在wait_pthread函数处。bthread 1执行结束后，main函数的线程才会被唤醒，继续向下执行，去join 下一个bthread。此时bthread 2可能是已经结束的状态，代码中也有相应的处理，判断出bthread 2工作结束，main的线程不会被挂起。

程序运行到此的状态是，三个bthread A、B、C已经创建完毕，bthread id已经被压入TaskGroup对象的任务队列_remote_rq，TaskGroup所属的pthread线程即将拿到bthread id，main函数所在线程被挂起，等待bthread A的结束。此时刻的系统中的内存布局如下：

<img src="../images/client_send_req_1.png" width="70%" height="70%"/>


5. 各个TaskGroup所在的pthread从_remote_rq中拿到bthread id，进而从对象池中找到TaskMeta，开始执行TaskMeta的任务函数，即client.cpp中的static类型的sender函数。由于各个bthread有各自的私有栈空间，所以sender中的局部变量被分配在bthread的私有栈内存上。

6. 
