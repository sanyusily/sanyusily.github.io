---
layout: article
title: 'Linux 知识之IO多路复用epoll'
tags: Linux epoll
---


# Linux基础知识之IO多路复用epoll 


## epoll 简介
epoll是linux为监听多路IO的状态所实现的方法.

学习视频链接：https://www.bilibili.com/video/BV1iJ411S7UA

![SyncBlock_IO.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7554fbf9c0b844afbb34bcc4328e3c90~tplv-k3u1fbpfcp-watermark.image?)
如上图所示,我们前面在介绍eventfd和socketpair的时候,**例子用的都是同步阻塞IO的方式**.在单一使用的时候,看不出明显的问题.但是当2者同时使用的时候,**如果你想同时监听eventfd和socketpair这2路IO状态时,就得创建多一个用户线程B.** 此时看起来似乎问题也不大,但是如果监听数目达到一定数量级的时候呢?  
  **Linux为解决这种情况,提供了IO多路复用的方法epoll**
  
  
![epoll_IO.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a813622fc5444750a787f88a118b0147~tplv-k3u1fbpfcp-watermark.image?)
如上图可以看出,**epoll能在同时监听多路IO状态的基础上又不需要额外的线程开销**

## 函数原型

### 1. 创建epoll文件描述符

**SYNOPSIS**

```
#include <sys/epoll.h>

int epoll_create(int size); /* 从Linux内核版本2.6.8起,形参size被忽略,但仍需大于0 */
```
**RETURN VALUE**  
  On success, these system calls return a nonnegative file descriptor.  
  On error, -1 is returned, and errno is set to indicate the error.


### 2. epoll监听文件描述符

**SYNOPSIS**
```
#include <sys/epoll.h>

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

**RETURN VALUE**  
  When successful, epoll_ctl() returns zero.  
  When an error occurs, epoll_ctl() returns -1 and errno is set appropriately.

#### 2.1 参数说明

| OP                | 说明              |
| ----------------- | --------------- |
| **EPOLL_CTL_ADD** | 添加监听文件描述符       |
| **EPOLL_CTL_MOD** | 修改监听文件描述符的event |
| **EPOLL_CTL_DEL** | 删除监听文件描述符       |

| Event Type       | 说明                                                                                                                  |
| ---------------- | ------------------------------------------------------------------------------------------------------------------- |
| **EPOLLIN**      | 事件可读                                                                                                                |
| **EPOLLOUT**     | 事件可写                                                                                                                |
| **EPOLLRDHUP**   | 连接断开(**针对socket**)                                                                                                  |
| **EPOLLERR**     | 事件异常                                                                                                                |
| **EPOLLHUP**     | 连接断开(**全类型文件描述符**)                                                                                                  |
| **EPOLLET**      | 边缘触发                                                                                                                |
| **EPOLLONESHOT** | 事件仅触发一次,**如需再次触发,则要再次调用epoll_ctl(op为EPOLL_CTL_MOD)**                                                                |
| **EPOLLWAKEUP**  | 在EPOLLONESHOT和EPOLLET都没设置并且进程拥有CAP_BLOCK_SUSPEND权限的前提下,**事件就绪时会申请唤醒锁,阻止系统进入suspend状态,直至事件处理完成再次调用epoll_wait时释放唤醒锁** |

**相关结构体**

```
typedef union epoll_data {      /* 联合体,联合体内成员无偏移,既共用首地址,该联合体内存占8字节 */
	void        *ptr;	
	int          fd;
	uint32_t     u32;
	uint64_t     u64;
} epoll_data_t;

struct epoll_event {
	uint32_t     events;    /* epoll 监听的事件类型 */
	epoll_data_t data;      /* 事件就绪时返回给用户进程 */
};
```

#### 2.2 EPOLLET边缘触发和EPOLLLT水平触发区别

```
// Eventpoll.c(kernel-4.9\fs)
static int ep_send_events_proc(struct eventpoll *ep, struct list_head *head,void *priv)
{
......
	for (eventcnt = 0, uevent = esed->events;
	     !list_empty(head) && eventcnt < esed->maxevents;) {
......
        list_del_init(&epi->rdllink);	/* 将事件从就绪队列中移除 */

		if (epi->event.events & EPOLLONESHOT)
			epi->event.events &= EP_PRIVATE_BITS;
		else if (!(epi->event.events & EPOLLET)) {	
        /* 当触发方式为非边缘触发时,会再次将该事件加入到就绪队列中.
         * 当用户下次调用epoll_wait时会检测事件是否处理完成,
         * 如果未处理完毕,则会再次报告有事件产生.
        */
		    list_add_tail(&epi->rdllink, &ep->rdllist);
		    ep_pm_stay_awake(epi);
		}
	}
	return eventcnt;
}
```
**从上面源码的中,我们举个例子来说明两种触发模式的区别**

  **线程A写了32个字节数据到socket fd触发EPOLLIN事件**  
  **线程B从epoll_wait中返回,只读取了10字节数据**

  **EPOLLLT水平触发: 由于未读取完毕,再次调用epoll_wait会直接返回.报告有事件可读**

  **EPOLLET边缘触发: 直接休眠等待下次socket fd的EPOLLIN事件触发(线程A的write动作)**

### 3. epoll等待监听的事件触发

**SYNOPSIS**

```
#include <sys/epoll.h>

int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);
```
#### 3.1 参数说明

| 参数            | 说明                             |
| ------------- | ------------------------------ |
| **返回值**       | 0代表等待超时,小于0代表错误发生,大于0代表触发的事件个数 |
| **timeout**   | 超时时间,毫秒为单位.当为负数时,会一直阻塞等待事件触发   |
| **maxevents** | 最大触发事件的个数(<=events容量)          |
| **events**    | 事件触发时,该数组会被填充                  |

## 例子

```
/*本例子的流程为: 
* 1. 调用socketpair得到一对双向fd
* 2. 调用fork,产生子进程.子进程每5秒向父进程发送信息
* 3. 父进程创建eventfd,用于通知终端输入情况
* 4. 父进程创建epoll,并将socketpair和eventfd加入监听列表中
* 5. 父进程调用epoll_wait等待监听事件触发
*/
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <sys/types.h>         
#include <sys/socket.h>
#include <sys/epoll.h>
#include <sys/eventfd.h>
#include <pthread.h>

#define true 1
#define false 0
#define EPOLL_INIT 1

typedef unsigned char boolean ;

struct Transfer_Data {
	int eventFd;
	char *msg;
	char len;
	boolean exit;
};

int addFd(int epollFd, int fd, int events) {
	struct epoll_event eventItem;
	memset(&eventItem, 0, sizeof(struct epoll_event)); 
	eventItem.events = events;
	eventItem.data.fd = fd;
	if (epoll_ctl(epollFd ,EPOLL_CTL_ADD, fd, &eventItem) < 0) {
		printf("add fd fail reason: %s\n", strerror(errno));
		return -1;
	}
	return 0;
}

void childLoop(int fd) {	
	char recvMsg[128] = { 0 };
	char *sendMsg = "I am Child";

	while (true) {
		sleep(5);
		if (write(fd, sendMsg, strlen(sendMsg) + 1) < 0) {
			printf("%s write fail reason: %s\n", __func__, strerror(errno));
			return;
		}
	}
}

int awoken(int wakeFd) {
	uint64_t counter;
	if (read(wakeFd, &counter, sizeof(uint64_t)) < 0) {
		printf("read wakeFd fail reason: %s\n", strerror(errno));
		return -1;
	}
	return 0;
}

void *thread_func(void *arg) {  
	struct Transfer_Data *ptd = (struct Transfer_Data *)arg;
	uint64_t inc = 1;
	int err = 0;
	for (;ptd -> exit == false;) {
		fgets(ptd -> msg, ptd -> len, stdin);
		if ((err = write(ptd -> eventFd, &inc, sizeof(uint64_t))) < 0) {	
			printf("write eventfd fail reason: %s\n",strerror(errno));
			ptd -> exit = true;
			break;
		}
	}
	return NULL;
}  

void parentLoop(int fd) {	
	char msg[128] = { 0 };
	char recvMsg[128] = {0};
	int mEpollFd = -1;
	int eventCount = 0;
	int err = 0;
	struct epoll_event events[8];
	
	int wakeEventFd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK); 
	if (wakeEventFd < 0) {
		printf("Create eventfd fail reason: %s\n",strerror(errno));
		return;
	}

	pthread_t readThread;
	struct Transfer_Data td = {wakeEventFd, msg, 128, false};
	err = pthread_create(&readThread, NULL, thread_func, &td);
	if (err != 0) {
		printf("Create thread fail reason: %s\n", strerror(err));
		goto EXIT;
	}

	if ((mEpollFd = epoll_create(EPOLL_INIT)) < 0) {
		printf("epoll_create fail reason: %s\n", strerror(errno));
		goto EXIT;
	}

	if (addFd(mEpollFd, fd, EPOLLIN) || addFd(mEpollFd, wakeEventFd, EPOLLIN)) {
		printf("addFd fail\n");
		goto EXIT;
	}
	
	while (td.exit == false) {
		printf("%s epoll_wait\n",__func__);
		eventCount = epoll_wait(mEpollFd, events, sizeof(events) / sizeof(*events), -1);		
		if (eventCount < 0) {	
			printf("epoll_wait fail reason: %s\n", strerror(errno));
			goto EXIT;
		}else if (eventCount == 0) {
			printf("epoll_wait timeout,continue wait\n");
			continue;
		}else {
			for (int i =0; i < eventCount; i++) {
				if (events[i].data.fd == wakeEventFd) {
					if (awoken(events[i].data.fd) < 0)
						goto EXIT;
					printf("%s read msg : %s\n", __func__, msg);
				}else {
					if (read(events[i].data.fd, recvMsg, sizeof(recvMsg) / sizeof(*recvMsg)) < 0) {
						printf("%s read fail reason: %s\n", __func__, strerror(errno));
						goto EXIT;
					}
					printf("%s read msg : %s\n", __func__, recvMsg);
				}
			}
		}
	}
EXIT:
	td.exit = true;
	pthread_join(readThread, NULL);
    /* 创建和close应成对出现,防止FD泄露 */
	close(wakeEventFd);
	close(mEpollFd);
	return;
}

int main(int argc __unused, char **argv __unused) {
	int pid;
	int fd[2];
	int err = 0;
	
	err = socketpair(AF_UNIX, SOCK_SEQPACKET | SOCK_CLOEXEC | SOCK_NONBLOCK, 0, fd);

	if (err != 0) {
		printf("socketpair fail reason: %s\n", strerror(errno));
		return -1;
	}

	pid = fork();

	if (pid < 0) {
		printf("fork fail reason: %s\n", strerror(errno));
		return -1;

	}

	if (pid == 0) {			//子进程
		close(fd[0]);		//关闭fd[0]
		childLoop(fd[1]);
		close(fd[1]);		//关闭fd[1]
		return 0;
	}else {				//父进程
		close(fd[1]);		//关闭fd[1]
		parentLoop(fd[0]);
		close(fd[0]);		//关闭fd[0]
		return 0;
	}
	return 0;
}
```

**运行结果**
```
generic_arm64_ab:/ # example
parentLoop epoll_wait
parentLoop read msg : I am Child
parentLoop epoll_wait
Hello World		<---终端输入
parentLoop read msg : Hello World
parentLoop epoll_wait