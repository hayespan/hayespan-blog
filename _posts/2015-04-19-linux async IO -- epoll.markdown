---
layout: post
title: "linux async IO -- epoll"
date:  2015-04-19 20:24:19 +0800
categories: 编程
---

今天体验了一下`Linux`的专有异步系统调用`epoll`（合理来说，还是同步的，只是利用了`event`结构比较高效地通知活跃`fd`从而避免线性扫描），总体还是比较容易用的，主要是3步操作：

{% highlight c linenos %}
// 创建能监听size个fd的epoll内核结构，返回的是epfd
int epoll_create(int size);

// 为epoll对象进行操作，添加修改删除（op）感兴趣的event
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// event中包含的源fd信息，相比select的轮询，epoll更高效，原因在此
typedef union epoll_data {
    void        *ptr; // 可以是fd文件指针
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;
// event结构体，用于描述epoll事件与源fd信息
struct epoll_event {
    uint32_t     events;      /* Epoll events */
    epoll_data_t data;        /* User data variable */
};

// 等待epoll监听，区别于select，不用反复FD_CLR与FD_SET
// events每次都会自动更新，maxevents不能大于size, timeout是毫秒数
int epoll_wait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout);
{% endhighlight %}

---

### 水平触发（Level-triggered）


先po默认模式（水平触发，level-triggered）的代码。水平触发是针对`fd`的某一事件来设置的，例如要监听`ev`的可读事件，使用：`ev.events = EPOLLIN`，默认是水平触发模式，也称为条件模式。那么什么是水平触发模式呢？举个例子，某一时刻，`epoll_wait`告诉我们，`fd = STDIN_FILENO`可读，但我们不理他，不进行读操作。在下一循环，`epoll_wait`仍会告诉我们可读。如果我们有去读取了，但只是读了一半，`epoll_wait`还是会告诉我们可读。所以这就是为何我们称之为条件模式，只要满足某一事件活跃的条件，`epoll_wait`就会义无反顾的通知我们。

下面是一个简单的例子，接受命令行输入，并把接收到的字符打印出来，每次只接收一个字符。

同时我也遇到了一个很奇怪的问题，当使用默认的`read`（`blocking IO`）时，会导致输入`aaa`+`Ctrl-D`时，`epoll_wait`返回4次通知。并且在最后一个`read`直接阻塞掉。至今我还没有搞清楚这是怎么一回事，我的困惑是，合理来说，在水平触发模式下，通知的条件是存在活跃事件，所以在输入`aaa`时，缓冲区会有3个`a`字符，随着按下`Ctrl-D`，系统会把缓冲区`flush`出去，意味着服务端实际上只接收到3个`a`，由于使用了系统调用而不是带缓冲区的库函数，每次只从系统缓冲区读取一个字符`a`，所以合理来说应该只发生3次通知，而不是4次，结果呢？`epoll_wait`返回了4次可读的通知！这个问题我放在了stackoverflow上：[Why does the read() block in this case?(linux epoll)](http://stackoverflow.com/questions/29727210/why-does-the-read-block-in-this-caselinux-epoll)。
我挺现在怀疑这样的情况是特殊的规则。

下边的代码使用的是`O_NONBLOCK`，解决了read阻塞的问题，但根本上还是存在上述问题。

{% highlight c linenos %}
#include <sys/epoll.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <fcntl.h>
#include <errno.h>

int set_nonblock(int sfd) {
    int flags, s;
    flags = fcntl (sfd, F_GETFL, 0);
    if (flags == -1) {
        perror ("fcntl");
        return -1;

    }
    flags |= O_NONBLOCK;
    s = fcntl (sfd, F_SETFL, flags);
    if (s == -1) {
        perror ("fcntl");
        return -1;

    }
    return 0;
}

int main(int argc, const char *argv[]) {
    // create event
    struct epoll_event stdin_ev, events[10];

    // set event
    stdin_ev.events = EPOLLIN;
    stdin_ev.data.fd = STDIN_FILENO;

    // create epoll
    int epfd = epoll_create(1), i, rcnt;
    char c;

    // set nonblocking
    if(set_nonblock(STDIN_FILENO) != 0) {
        exit(EXIT_FAILURE);
    };
    
    // set monitoring STDIN_FILENO 
    epoll_ctl(epfd, EPOLL_CTL_ADD, STDIN_FILENO, &stdin_ev);

    while(1) {
        int ret = epoll_wait(epfd, events, 1, 1000);

        // timeout or failed
        if(ret == 0) {
            fprintf(stdout, "timeout\n");
            continue;
        } else if (ret < 0) {
            perror("ret<0");
            exit(EXIT_FAILURE);
        }

        // readable
        fprintf(stdout, "%d event(s) happened...\n", ret);
        for(i=0;i<ret;i++) {
            if(events[i].data.fd == STDIN_FILENO && events[i].events&EPOLLIN) {
                // read a char
                rcnt = read(STDIN_FILENO, &c, 1); 
                // if read 0 char, EOF
                if(rcnt == 0) {
                    fprintf(stdout, "EOF\n");
                    continue;
                } else if (rcnt == -1 && errno == EAGAIN) {
                    fprintf(stdout, "EAGAIN\n");
                    continue;
                }
                // else print ascii
                fprintf(stdout, "ascii code: %d\n", c);
            }
        } 
    }
    close(epfd);
    return 0;
}
{% endhighlight %}

----

### 边缘触发（Edge-triggered）

不同于水平触发，边缘触发只有在状态改变时才会提示事件，例如，某一刻，数据到了，`epoll_wait`返回提示可读，但我们不读取数据或者读取了部分数据，此时系统缓冲区仍有数据，那么在下个循环中，`epoll`是不会再提醒可读的，而是需要等到下一个事件到来。可理解为电平的变化。

（需要注意一点：水平触发可在 __阻塞/非阻塞__ 模式下工作，而边缘触发只能是 __非阻塞IO__ 。当使用边缘触发时，读操作必须保证读到`errno = EAGAIN`为止，否则，数据尽管还在缓冲区，但将与事件不同步，例如，第一次有100k到达，读取50k，第二次有30k到达，那么读取的仍为第一次的50k）

以下是将水平触发的代码进行简单的修改。

{% highlight c linenos %}
#include <sys/epoll.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <fcntl.h>
#include <errno.h>

int set_nonblock(int sfd) {
    int flags, s;
    flags = fcntl (sfd, F_GETFL, 0);
    if (flags == -1) {
        perror ("fcntl");
        return -1;

    }
    flags |= O_NONBLOCK;
    s = fcntl (sfd, F_SETFL, flags);
    if (s == -1) {
        perror ("fcntl");
        return -1;

    }
    return 0;
}

int main(int argc, const char *argv[])
{
    // create event
    struct epoll_event stdin_ev, events[10];

    // set event
    stdin_ev.events = EPOLLIN;
    stdin_ev.events = EPOLLIN|EPOLLET;
    stdin_ev.data.fd = STDIN_FILENO;

    // create epoll
    int epfd = epoll_create(1), i, rcnt;
    char c;

    // set nonblocking
    if(set_nonblock(STDIN_FILENO) != 0) {
        exit(EXIT_FAILURE);
    };
    
    // set monitoring STDIN_FILENO 
    epoll_ctl(epfd, EPOLL_CTL_ADD, STDIN_FILENO, &stdin_ev);

    while(1) {
        int ret = epoll_wait(epfd, events, 1, 1000);

        // timeout or failed
        if(ret == 0) {
            fprintf(stdout, "timeout\n");
            continue;
        } else if (ret < 0) {
            perror("ret<0");
            exit(EXIT_FAILURE);
        }

        // readable
        fprintf(stdout, "%d event(s) happened...\n", ret);
        for(i=0;i<ret;i++) {
            if(events[i].data.fd == STDIN_FILENO &&\
                    events[i].events&EPOLLIN) {
                // use loop to read all characters util read() == -1 && errno = EAGAIN
                while(1) {
                    rcnt = read(STDIN_FILENO, &c, 1);
                    if(rcnt == -1 && errno == EAGAIN) {
                        fprintf(stdout, "EAGAIN\n");
                        break;
                    }
                    fprintf(stdout, "ascii code: %d\n", c);
                }
            }
        } 
    }
    close(epfd);
    return 0;
}
{% endhighlight %}

测试一下，结果：

```
./a.out
timeout
hayespan // <-- "hayespan"+`Ctrl-D`
1 event(s) happened...
ascii code: 104
ascii code: 97
ascii code: 121
ascii code: 101
ascii code: 115
ascii code: 112
ascii code: 97
ascii code: 110
ascii code: 10
EAGAIN // <-- errno = EAGAIN
timeout
^C

```

----

一些不错的文章：

> 1. [How to use epoll? A complete example in C](https://banu.com/blog/2/how-to-use-epoll-a-complete-example-in-c/)
> 2. [Epoll精髓](http://www.cnblogs.com/onlyxp/archive/2007/08/10/851222.html) // 这篇主要是因为他翻译了man方便查询，233333

