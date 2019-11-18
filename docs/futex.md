## 

## brpc中futex的实现

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
