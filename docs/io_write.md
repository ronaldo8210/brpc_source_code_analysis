## 概述


## 具体实现
brpc中的Socket类对象代表Client端与Server端的一条TCP连接，

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

