## 概述
考虑brpc自带的示例程序example/multi_threaded_echo_c++/client.cpp，use_bthread为true的情况下，多个bthread通过一条TCP长连接向服务端发送数据，而多个bthread又是运行在多个系统线程pthread上的，所以多个pthread如何高效且线程安全地向一个TCP连接写数据，通常是系统设计需要重点考虑的。

## 具体实现
brpc中的Socket类对象代表Client端与Server端的一条TCP连接，

Socket类对象中比较重要的成员变量：

_epollout_butex：

_write_head：

结合代码阐述下Socket类的几个主要函数的作用：

StartWrite函数：向TCP连接写数据的入口函数，在实际环境下通常会被多个pthread执行，必须要做到线程安全

```c++
int Socket::StartWrite(WriteRequest* req, const WriteOptions& opt) {
    // Release fence makes sure the thread getting request sees *req
    // 与当前
    WriteRequest* const prev_head =
        _write_head.exchange(req, butil::memory_order_release);
    if (prev_head != NULL) {
        // Someone is writing to the fd. The KeepWrite thread may spin
        // until req->next to be non-UNCONNECTED. This process is not
        // lock-free, but the duration is so short(1~2 instructions,
        // depending on compiler) that the spin rarely occurs in practice
        // (I've not seen any spin in highly contended tests).
        req->next = prev_head;
        return 0;
    }

    int saved_errno = 0;
    bthread_t th;
    SocketUniquePtr ptr_for_keep_write;
    ssize_t nw = 0;

    // We've got the right to write.
    req->next = NULL;
    
    // Connect to remote_side() if not.
    int ret = ConnectIfNot(opt.abstime, req);
    if (ret < 0) {
        saved_errno = errno;
        SetFailed(errno, "Fail to connect %s directly: %m", description().c_str());
        goto FAIL_TO_WRITE;
    } else if (ret == 1) {
        // We are doing connection. Callback `KeepWriteIfConnected'
        // will be called with `req' at any moment after
        return 0;
    }

    // NOTE: Setup() MUST be called after Connect which may call app_connect,
    // which is assumed to run before any SocketMessage.AppendAndDestroySelf()
    // in some protocols(namely RTMP).
    req->Setup(this);
    
    if (ssl_state() != SSL_OFF) {
        // Writing into SSL may block the current bthread, always write
        // in the background.
        goto KEEPWRITE_IN_BACKGROUND;
    }
    
    // Write once in the calling thread. If the write is not complete,
    // continue it in KeepWrite thread.
    if (_conn) {
        butil::IOBuf* data_arr[1] = { &req->data };
        nw = _conn->CutMessageIntoFileDescriptor(fd(), data_arr, 1);
    } else {
        nw = req->data.cut_into_file_descriptor(fd());
    }
    if (nw < 0) {
        // RTMP may return EOVERCROWDED
        if (errno != EAGAIN && errno != EOVERCROWDED) {
            saved_errno = errno;
            // EPIPE is common in pooled connections + backup requests.
            PLOG_IF(WARNING, errno != EPIPE) << "Fail to write into " << *this;
            SetFailed(saved_errno, "Fail to write into %s: %s", 
                      description().c_str(), berror(saved_errno));
            goto FAIL_TO_WRITE;
        }
    } else {
        AddOutputBytes(nw);
    }
    if (IsWriteComplete(req, true, NULL)) {
        ReturnSuccessfulWriteRequest(req);
        return 0;
    }

KEEPWRITE_IN_BACKGROUND:
    ReAddress(&ptr_for_keep_write);
    req->socket = ptr_for_keep_write.release();
    if (bthread_start_background(&th, &BTHREAD_ATTR_NORMAL,
                                 KeepWrite, req) != 0) {
        LOG(FATAL) << "Fail to start KeepWrite";
        KeepWrite(req);
    }
    return 0;

FAIL_TO_WRITE:
    // `SetFailed' before `ReturnFailedWriteRequest' (which will calls
    // `on_reset' callback inside the id object) so that we immediately
    // know this socket has failed inside the `on_reset' callback
    ReleaseAllFailedWriteRequests(req);
    errno = saved_errno;
    return -1;
}
```

KeepWrite
```c++
void* Socket::KeepWrite(void* void_arg) {
    g_vars->nkeepwrite << 1;
    WriteRequest* req = static_cast<WriteRequest*>(void_arg);
    SocketUniquePtr s(req->socket);

    // When error occurs, spin until there's no more requests instead of
    // returning directly otherwise _write_head is permantly non-NULL which
    // makes later Write() abnormal.
    WriteRequest* cur_tail = NULL;
    do {
        // req was written, skip it.
        if (req->next != NULL && req->data.empty()) {
            WriteRequest* const saved_req = req;
            req = req->next;
            s->ReturnSuccessfulWriteRequest(saved_req);
        }
        const ssize_t nw = s->DoWrite(req);
        if (nw < 0) {
            if (errno != EAGAIN && errno != EOVERCROWDED) {
                const int saved_errno = errno;
                PLOG(WARNING) << "Fail to keep-write into " << *s;
                s->SetFailed(saved_errno, "Fail to keep-write into %s: %s",
                             s->description().c_str(), berror(saved_errno));
                break;
            }
        } else {
            s->AddOutputBytes(nw);
        }
        // Release WriteRequest until non-empty data or last request.
        while (req->next != NULL && req->data.empty()) {
            WriteRequest* const saved_req = req;
            req = req->next;
            s->ReturnSuccessfulWriteRequest(saved_req);
        }
        // TODO(gejun): wait for epollout when we actually have written
        // all the data. This weird heuristic reduces 30us delay...
        // Update(12/22/2015): seem not working. better switch to correct code.
        // Update(1/8/2016, r31823): Still working.
        // Update(8/15/2017): Not working, performance downgraded.
        //if (nw <= 0 || req->data.empty()/*note*/) {
        if (nw <= 0) {
            // 如果是由于fd的inode输出缓冲区已满导致write操作返回值小于等于0，则需要挂起执行KeepWrite的bthread，让出cpu，
            // 让该bthread所在的pthread去任务队列中取出下一个bthread去执行。等到epoll返回告知inode输出缓冲区有可写空间时，
            // 再唤起执行KeepWrite的bthread，继续向fd写入数据
            g_vars->nwaitepollout << 1;
            bool pollin = (s->_on_edge_triggered_events != NULL);
            // NOTE: Waiting epollout within timeout is a must to force
            // KeepWrite to check and setup pending WriteRequests periodically,
            // which may turn on _overcrowded to stop pending requests from
            // growing infinitely.
            const timespec duetime =
                butil::milliseconds_from_now(WAIT_EPOLLOUT_TIMEOUT_MS);
            const int rc = s->WaitEpollOut(s->fd(), pollin, &duetime);
            if (rc < 0 && errno != ETIMEDOUT) {
                const int saved_errno = errno;
                PLOG(WARNING) << "Fail to wait epollout of " << *s;
                s->SetFailed(saved_errno, "Fail to wait epollout of %s: %s",
                             s->description().c_str(), berror(saved_errno));
                break;
            }
        }
        if (NULL == cur_tail) {
            for (cur_tail = req; cur_tail->next != NULL;
                 cur_tail = cur_tail->next);
        }
        // Return when there's no more WriteRequests and req is completely
        // written.
        if (s->IsWriteComplete(cur_tail, (req == cur_tail), &cur_tail)) {
            CHECK_EQ(cur_tail, req);
            s->ReturnSuccessfulWriteRequest(req);
            return NULL;
        }
    } while (1);

    // Error occurred, release all requests until no new requests.
    s->ReleaseAllFailedWriteRequests(req);
    return NULL;
}
```

