---
layout: post
title:  "linux socket编程"
date:   2015-04-24 20:24:19 +0800
categories: 编程
---

以前只拿`python`写过`socket`编程，今晚用`c`写了简单的`hello world`。先贴代码。

服务端代码：

{% highlight c linenos %}
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <stdlib.h>

typedef struct sockaddr_in Sockaddr_in;

int main(int argc, const char *argv[]) {
    int listenfd, clifd; 
    if ((listenfd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        fprintf(stdout, "create listen socket failed: %s\n", strerror(errno));
        exit(EXIT_FAILURE);
    }
    Sockaddr_in seraddr, cliaddr;
    bzero(&seraddr, 0);
    seraddr.sin_family = AF_INET;
    // seraddr.sin_addr.s_addr = inet_addr("172.18.32.255");
    seraddr.sin_addr.s_addr = htonl(INADDR_ANY);
    seraddr.sin_port = htons(8000);
    if (bind(listenfd, (struct sockaddr *) &seraddr, sizeof(seraddr)) < 0) {
        fprintf(stdout, "bind listen socket failed: %s\n", strerror(errno));
        exit(EXIT_FAILURE);
    }
    listen(listenfd, 100);
    socklen_t clilen;
    const char * msg = "hello world\n";
    for (;;) {
        clilen = sizeof(cliaddr);
        if ((clifd = accept(listenfd, (struct sockaddr *) &cliaddr, &clilen)) < 0) {
            fprintf(stdout, "accept client socket failed: %s\n", strerror(errno));
            exit(EXIT_FAILURE);
        }
        char buf[16];
        printf("connection from ip: %s port: %d\n", inet_ntop(AF_INET, &cliaddr.sin_addr, buf, sizeof(buf)), ntohs(cliaddr.sin_port)); 
        printf("%d\n", clifd);
        write(clifd, msg, strlen(msg));
        if (close(clifd) < 0) {
            fprintf(stdout, "close client socket failed: %s\n", strerror(errno));
            exit(EXIT_FAILURE);
        }
    }

    exit(EXIT_SUCCESS);
}
{% endhighlight %}

客户端代码：

{% highlight c linenos %}
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <stdlib.h>

typedef struct sockaddr_in Sockaddr_in;
int main(int argc, const char *argv[])
{
    int sfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sfd < 0) {
        fprintf(stdout, "create client socket failed: %s\n", strerror(errno));
        exit(EXIT_FAILURE);
    }
    Sockaddr_in seraddr;
    bzero(&seraddr, 0);
    seraddr.sin_family = AF_INET;
    seraddr.sin_port = htons(8000);
    // inet_pton(AF_INET, "172.18.32.255", &seraddr.sin_addr);
    seraddr.sin_addr.s_addr = htonl(INADDR_ANY);
    if (connect(sfd, (struct sockaddr *) &seraddr, sizeof(seraddr)) < 0) {
        fprintf(stdout, "connect failed: %s\n", strerror(errno));
        exit(EXIT_FAILURE);
    }
    char buf[100];
    if (read(sfd, buf, 100) < 0) {
        fprintf(stdout, "read failed: %s\n", strerror(errno));
        exit(EXIT_FAILURE);
    }
    fprintf(stdout, "%s", buf);
    return 0;
}
{% endhighlight %}

