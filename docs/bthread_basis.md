[线程执行协程与线程调用函数的不同](#线程执行协程与线程调用函数的不同)

[协程的原理与实现方式](#协程的原理与实现方式)

[brpc的bthread实现](#brpc的bthread实现)

## 线程执行协程与线程调用函数的不同
一个pthread系统线程执行一个函数时，需要在pthread的线程栈上为函数创建栈帧，函数的形式参数和局部变量都分配在栈帧内。函数执行完毕后，按逆序销毁函数局部变量，再销毁栈帧。假设有一个线程A开始执行下面的foo函数：

```c++
void bar(int m) {
  // 执行点1
  // ...
}

void foo(int a, int b) {
  int c = a + b;
  bar(c);
  // 执行点2
  // ...
}
```

执行到foo函数中的执行点1时，线程A的栈帧如下图所示：

<img src="../images/bthread_basis_1.png" width="20%" height="20%"/>

线程A从bar()函数返回，执行到执行点2时，先销毁bar()函数的形参m，再销毁bar()的栈帧，从foo()函数返回后，先销毁局部变量c，接着销毁形参b、a，最后销毁foo()的栈帧。

像上述这种在foo()函数内调用bar()函数的过程，必须等到bar()函数return后，foo()函数才从bar()函数的返回点恢复执行。

一个协程可以看做是一个单独的任务，相应的也有一个任务处理函数，一个线程执行一个协程A的任务处理函数taskFunc_A时，如果想要去执行另一个协程B的任务处理函数taskFunc_B，不必等到taskFunc_A执行到return语句，可以在taskFunc_A内执行一个yield语句（yield的具体实现见下文描述），然后线程可以从taskFunc_A中跳出，去执行taskFunc_B。如果想让taskFunc_A恢复执行，则调用一个resume语句，让taskFunc_A从yield语句的返回点处开始继续执行，并且taskFunc_A的执行结果不受yield的影响。




## 协程的原理与实现方式
协程有三个组成要素：一个任务函数，一个存储寄存器状态的结构，一个私有栈空间（通常是malloc分配的一块内存，或者static静态区的一块内存）。

协程被称作用户级线程就是因为协程有其私有的栈空间，pthread系统线程调用一个普通函数时，函数的栈帧、形参、局部变量都分配在pthread线程的栈上，而pthread执行一个协程的任务函数时，协程任务函数的栈帧、形参、局部变量都分配在协程的私有栈上。

对协程来说最重要的两个操作是yield和resume，yield和resume有多种实现方式，可以使用posix的ucontext，boost的fcontext，或者直接用汇编实现。下面用ucontext讲述下如何实现协程的yield和resume：

1. posix定义的ucontext数据结构如下：

   ```c++
   
   ```
   
   用ucontext实现的一个协程的内存布局如下：
   
   

2. ucontext的api接口有如下四个：

   - int getcontext(ucontext_t *ucp)
   
   - int setcontext(const ucontext_t *ucp)
   
   - void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...)
   
   - int swapcontext(ucontext_t *oucp, ucontext_t *ucp)
   
下面通过一个示例程序，展现pthread系统线程执行多个协程时的内存变化过程：

```c++

```

在上述程序中，pthread执行到main函数的

## brpc的bthread实现
