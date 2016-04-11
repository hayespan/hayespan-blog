---
layout: post
title:  "linux pthread多线程"
date:   2015-04-24 20:24:19 +0800
categories: 编程
---

线程与进程相比比较轻量，`pthread`通过包装`Linux`的`LWP`(Light-weight process，轻量级进程)来实现线程。

写了几个简单的例子，描述直接注释在代码中。

{% highlight c %}
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <errno.h>
#include <string.h>
#include <stdlib.h>

// 参数结构体
typedef struct Arg {
    int x;
    double y;
} Arg_t;

// 清理函数
void * cleanup(void * arg) {
    fprintf(stdout, "call cleanup%d\n", (int)arg);
    return arg;
}

// 线程1函数
void * thread1(void * arg) {
    fprintf(stdout, "hello world from %lu, arg: %d, %lf\n",\
            pthread_self(),\
            ((Arg_t *)arg)->x,\
            ((Arg_t *)arg)->y 
            );
    return (void *)pthread_self();
}

// 线程2函数
void * thread2(void * arg) {
    // cleanup_push/pop必须成对出现，因为是用宏实现的
    pthread_cleanup_push(cleanup, (void *) 2);
    pthread_cleanup_push(cleanup, (void *) 1);
    {
        // 在之间如果包含`{`则要有配对的`}`
    }
    pthread_cleanup_pop(0); // 0 表示出栈，但不执行
    pthread_cleanup_pop(1); // 非0 表示出栈并执行
    return NULL; // 普通的return不会调用清理函数
}

// 线程3函数
void * thread3(void * arg) {
    pthread_cleanup_push(cleanup, (void *) 1);
    sleep(1); // 等待被cancel
    if((int)arg == 1) {
        pthread_exit(0); // pthread_exit函数会自动调用清理函数
    }
    // 意味着在正常逻辑中要决定好是否调用清理函数
    pthread_cleanup_pop(0);
    return NULL; // 尽管普通的return并不会调用清理函数
}

int main(int argc, const char *argv[]) {
    int ret;
    pthread_t pt_id;

    // #1: 通过`void *`传递参数arg与返回值res
    Arg_t arg = {1, 3.14159};
    if ((ret = pthread_create(&pt_id, NULL, thread1, &arg)) != 0) {
        fprintf(stderr, "create pthread failed: %s\n", strerror(errno));
        exit(EXIT_FAILURE);
    }
    void *res;
    pthread_join(pt_id, &res);
    fprintf(stdout, "res: %lu\n", (unsigned long) res);

    // #2: 关于pthread_cleanup_pop(x)
    if ((ret = pthread_create(&pt_id, NULL, thread2, (void*) 1)) != 0) {
        fprintf(stderr, "create pthread failed: %s\n", strerror(errno));
        exit(EXIT_FAILURE);
    }
    pthread_join(pt_id, &res);

    // #3: 关于pthread_cancel，清理函数，pthread_join的行为
    if ((ret = pthread_create(&pt_id, NULL, thread3, (void *) 1)) != 0) {
        fprintf(stderr, "create pthread failed: %s\n", strerror(errno));
        exit(EXIT_FAILURE);
    }
    fprintf(stdout, "cancel thread3\n");
    pthread_cancel(pt_id); // 取消新建线程
    ret = pthread_join(pt_id, &res); // thread3已经被cancel，所以pthread_join立即返回0
    fprintf(stdout, "pthread_join return: %d\n", ret); 
    if (res == PTHREAD_CANCELED) {
        fprintf(stdout, "PTHREAD_CANCELED\n"); // res此时保存的是PTHREAD_CANCELED
    }
    
    return 0;
}
{% endhighlight %}

输出结果：

```
#1
hello world from 3075464000, arg: 1, 3.141590
res: 3075464000

#2
call cleanup2

#3
cancel thread3
call cleanup1
pthread_join return: 0
PTHREAD_CANCELED
```

此外关于线程同步的，还有`mutex`，`自旋锁`，`barrier`等，待补坑。

----

> [关于Linux的进程和线程](http://kenby.iteye.com/blog/1014039)

