[spinlock和内核提供的同步机制存在的不足](#spinlock和内核提供的同步机制存在的不足)

[Futex设计原理](#Futex设计原理)

[brpc中Futex的实现](#brpc中Futex的实现)

## spinlock和内核提供的同步机制存在的不足
在Futex出现之前，想要pthread系统线程等待一把锁，有两种实现方式：

1. 使用spinlock自旋锁，实现起来也很简单：

    ```c++
    std::atomic<int> flag;

    void lock() {
      int expect = 0;
      while (!flag.compare_exchange_strong(expect, 1)) {
        expect = 0;
      }
    }

    void unlock() {
      flag.store(0);
    }
    ```

   spinlock属于应用层的同步机制，直接运行在用户态，不涉及用户态-内核态的切换。spinlock存在的问题是，只适用于需要线程加锁的临界区代码段较小的场景，在这样的场景下可以认为一个线程加了锁后很快就会释放锁，等待锁的其他线程只需要很少的while()调用就可以得到锁；但如果临界区代码很长，一个线程加锁后会耗费相当一段时间去执行临界区代码，在这个线程释放锁之前其他线程只能不停地在while()中不断busy-loop，耗费了cpu资源，而且在应用程序层面又没有办法能让pthread系统线程挂起。

2. 直接使用Linux提供的pthread_mutex_lock系统调用或者编程语言提供的操作互斥锁的API（如C++的std::unique_lock），本质上是一样的，都是使用Linux内核提供的同步机制，可以让pthread系统线程挂起，让出CPU。但缺点是如果直接使用Linux内核提供的同步机制，每一次lock、unlock操作都是一次系统调用，需要进行用户态-内核态的切换，存在一定的性能开销，但lock、unlock的时候不一定会有线程间的竞争，在没有线程竞争的情况下没有必要进行用户态-内核态的切换。

## Futex设计原理
Futex机制可以认为是结合了spinlock和内核态的pthread线程锁，它的设计意图是：

1. 一个线程在加锁的时候，在用户态使用原子操作执行“尝试加锁，如果当前锁变量值为0，则将锁变量值更新为1，返回成功；如果当前锁变量值为1，说明之前已有线程拿到了锁，返回失败”这个动作，如果加锁成功，则可直接去执行临界区代码；如果加锁失败，则用类似pthread_mutex_lock这样的系统调用将当前线程挂起。由于可能有多个线程同时被挂起，所以必须将各个被挂起线程的信息存入一个与锁相关的等待队列中；

2. 一个线程在释放锁的时候，也是用原子操作将锁变量的值改回0，并且如果与锁相关的等待队列不为空，则释放锁的线程必须使用内核提供的系统调用去唤醒因等待锁而被挂起的线程，具体是唤醒一个线程还是唤醒全部线程视使用场景而定；

3. 由上可见，使用Futex机制，在没有线程竞争的情况下，在用户层就可以完成临界区代码的加锁解锁，只有在确实有线程竞争的情况下才会使用内核提供的系统调用实现线程的挂起与唤醒。

4. 在实现Futex的时候有一个细节需要注意，Futex的代码如果像下面这样写：

   ```c++
   void lock() {
     while (!trylock()) {
       wait();  // 使用内核提供的系统调用挂起当前线程
     }
   }
   ```
   
   这样的实现存在问题， 

## brpc中Futex的实现
brpc实现了Futex机制，主要代码在src/bthread/sys_futex.cpp中，主要有两个函数分别负责wait和wake：

- futex_wait_private函数

```c++
int futex_wait_private(void* addr1, int expected, const timespec* timeout) {
    if (pthread_once(&init_futex_map_once, InitFutexMap) != 0) {
        LOG(FATAL) << "Fail to pthread_once";
        exit(1);
    }
    std::unique_lock<pthread_mutex_t> mu(s_futex_map_mutex);
    SimuFutex& simu_futex = (*s_futex_map)[addr1];
    ++simu_futex.ref;
    mu.unlock();

    int rc = 0;
    {
        std::unique_lock<pthread_mutex_t> mu1(simu_futex.lock);
        if (static_cast<butil::atomic<int>*>(addr1)->load() == expected) {
            ++simu_futex.counts;
            if (timeout) {
                timespec timeout_abs = butil::timespec_from_now(*timeout);
                if ((rc = pthread_cond_timedwait(&simu_futex.cond, &simu_futex.lock, &timeout_abs)) != 0) {
                    errno = rc;
                    rc = -1;
                }
            } else {
                if ((rc = pthread_cond_wait(&simu_futex.cond, &simu_futex.lock)) != 0) {
                    errno = rc;
                    rc = -1;
                }
            }
            --simu_futex.counts;
        } else {
            errno = EAGAIN;
            rc = -1;
        }
    }

    std::unique_lock<pthread_mutex_t> mu1(s_futex_map_mutex);
    if (--simu_futex.ref == 0) {
        s_futex_map->erase(addr1);
    }
    mu1.unlock();
    return rc;
}
```

- futex_wake_private函数

```c++
int futex_wake_private(void* addr1, int nwake) {
    if (pthread_once(&init_futex_map_once, InitFutexMap) != 0) {
        LOG(FATAL) << "Fail to pthread_once";
        exit(1);
    }
    std::unique_lock<pthread_mutex_t> mu(s_futex_map_mutex);
    auto it = s_futex_map->find(addr1);
    if (it == s_futex_map->end()) {
        mu.unlock();
        return 0;
    }
    SimuFutex& simu_futex = it->second;
    ++simu_futex.ref;
    mu.unlock();

    int nwakedup = 0;
    int rc = 0;
    {
        std::unique_lock<pthread_mutex_t> mu1(simu_futex.lock);
        nwake = (nwake < simu_futex.counts)? nwake: simu_futex.counts;
        for (int i = 0; i < nwake; ++i) {
            if ((rc = pthread_cond_signal(&simu_futex.cond)) != 0) {
                errno = rc;
                break;
            } else {
                ++nwakedup;
            }
        }
    }

    std::unique_lock<pthread_mutex_t> mu2(s_futex_map_mutex);
    if (--simu_futex.ref == 0) {
        s_futex_map->erase(addr1);
    }
    mu2.unlock();
    return nwakedup;
}
```
