---
layout: post
title:  "linux IO -- select"
date:   2015-04-16 20:24:19 +0800
categories: 编程
---

以前写过一些简单的服务端，读写`socket`一直都是采用多线程+阻塞IO的方式，一个线程读，一个线程写，当时一开始还不知道`socket`是双工的，于是傻傻的创建两个`socket`= =。

很久之前就听过了`异步IO`模型的`select`，但一直没去尝试它，今天学习了一下，贴下简单的代码。

（其实我觉得实际上应该称之为`同步非阻塞`，因为感觉内部实现应该是遍历描述符集合，采用非阻塞的方式，不可读写便跳过。虽然在程序宏观逻辑上，是异步的效果，具体可参考[1][2]）

{% highlight c %}
#include <sys/select.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

void sys_err(const char * err) {
    fprintf(stderr, "%s\n", err);
    exit(-1);
}

int main(int argc, const char *argv[])
{
    fd_set fds;
    struct timeval tv = {0, 0}; // init timeval to 0s
    int ret;
    char c;
    while(1) {
        tv.tv_sec = 1; // set timeval to 1s
        FD_ZERO(&fds); // should clear fd_set each time
        FD_SET(STDIN_FILENO, &fds); // set accoding fd for monitoring
        ret = select(STDIN_FILENO+1, &fds, NULL, NULL, &tv);
        if(ret < 0) { // fd is invalid
            sys_err("ret < 0");
        }
        if(ret == 0) { // timeout 
            fprintf(stdout, "timeout\n");
        }
        if(FD_ISSET(STDIN_FILENO, &fds)) { // check which fd is ready
            fprintf(stdout, "start reading...\n");
            c = fgetc(stdin); // read a char each time
            if(c!=EOF)
                fprintf(stdout, "ascii code: %d\n", c);
            /* while((c=fgetc(stdin))!=EOF) { */ // fgetc读取非普通文件时是阻塞的
                /* fputc(c, stdout); */
            /* } */
            fprintf(stdout, "read off...\n");
            continue;
        }
    }
    exit(0);
}
{% endhighlight %}

----

> 1. [select，poll，epoll实现分析—结合内核源代码](http://blog.csdn.net/vividonly/article/details/7539342)
> 2. [select函数实现原理分析](http://bbs.chinaunix.net/thread-2021810-1-1.html)

