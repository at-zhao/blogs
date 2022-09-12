# 容器技术原理

{% hint style="warning" %}
本文所介绍的容器原理是基于 linux 操作系统的实现，Mac OS、Windows 下的容器实现原理和 linux 完全不同。
{% endhint %}

本文假设读者了解容器如 docker 等技术的基本概念，如不熟悉可以参考以下文章：

* [https://www.redhat.com/zh/topics/containers/what-is-docker#docker-%E7%9A%84%E7%BC%BA%E7%82%B9](https://www.redhat.com/zh/topics/containers/what-is-docker#docker-%E7%9A%84%E7%BC%BA%E7%82%B9)
* [https://security.feishu.cn/link/safety?target=https%3A%2F%2Fruanyifeng.com%2Fblog%2F2018%2F02%2Fdocker-tutorial.html\&scene=ccm\&logParams=%7B"location"%3A"ccm\_docs"%7D\&lang=zh-CN](https://security.feishu.cn/link/safety?target=https%3A%2F%2Fruanyifeng.com%2Fblog%2F2018%2F02%2Fdocker-tutorial.html\&scene=ccm\&logParams=%7B%22location%22%3A%22ccm\_docs%22%7D\&lang=zh-CN)

## 实现原理

{% hint style="success" %}
容器的本质是一个特殊的进程
{% endhint %}

容器是基于操作系统内核实现的隔离机制，而非虚拟机等技术基于硬件系统。

<figure><img src="../.gitbook/assets/virtualization-vs-containers_transparent.png" alt=""><figcaption></figcaption></figure>

而之所以说容器的本质是一个特殊的进程是因为基于操作系统的隔离机制的最小操作单位是进程，操作系统通过对单个进程施加限制来实现容器。

&#x20;Linux下容器的实现主要依赖于 Namespace 和 Cgroups 技术，Namespace 是用来隔离资源，或者说是修改进程视图，而 Cgroups 是用来限制资源使用的主要手段。

### Namespace

> A namespace wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource. Changes to the global resource are visible to other processes that are members of the namespace, but are invisible to other processes. One use of namespaces is to implement containers.

以上是 Linux 操作手册对`Namespace`的解释，`Namespace`提供了一种内核级别隔离系统资源的方法，通过将系统的全局资源放在不同的`Namespace`中来实现资源隔离的目的。在不同`Namespace`下的程序拥有一份 "看起来" 独立的系统资源。

目前 Linux 中提供了七类系统资源的隔离机制，分别是：

| Namespace Type | Flag             | 作用                         |
| -------------- | ---------------- | -------------------------- |
| Cgroup         | CLONE\_NEWCGROUP | 隔离 Cgroup 目录               |
| IPC            | CLONE\_NEWIPC    | 隔离文件系统挂载点                  |
| Network        | CLONE\_NEWNET    | 隔离主机名和域名信息                 |
| Mount          | CLONE\_NEWNS     | 隔离进程间通信，如管道、信号量、消息队列与共享内存等 |
| PID            | CLONE\_NEWPID    | 隔离进程 ID                    |
| User           | CLONE\_NEWUSER   | 隔离网络资源                     |
| UTS            | CLONE\_NEWUTS    | 隔离用户和用户组                   |

以上这些资源的隔离正是 Docker 等容器的核心能力。

#### Namespace 的创建

Namespace 的创建需要使用 `clone` 系统函数，此函数的定义如下：

```
int clone (int (*__fn) (void *__arg), void *__child_stack, int __flags, void *__arg, ...)
```

clone 是 linux 下创建进程的系统函数之一，它会创建一个新的子进程，并返回子进程的 pid。

如果在创建子进程的时候 flags 参数指定了上面表格中的 `CLONE_NEW*` 等常量，那么 linux 就会为此进程创建对应的 Namespace。如以下代码：

```
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/wait.h>

int container_main(void *arg) {
    printf("子进程认为自己的 pid: %d\n", getpid());
    return 0;
}

int main() {
    int stack_size = 1024 * 1024;
    void *stack = malloc(stack_size);
    void *child_stack = (char *) stack + stack_size; // 栈地址从高到低
    int flags = CLONE_NEWPID | CLONE_NEWUSER | CLONE_NEWNS | SIGCHLD; // sched.h
    int son_pid = clone(container_main, child_stack, flags, NULL);
    if (son_pid == -1) {
        printf("create process failed");
        return 1;
    }
    waitpid(son_pid, 0, 0);
    printf("父进程视角下子进程的 pid: %d\n", son_pid );
    return 0;
    
    
```

































