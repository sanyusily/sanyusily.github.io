---
layout: article
title: 'Linux 知识之eventfd'
tags: Linux AOSP
---


# Linux 知识之eventfd

## eventfd简介
  eventfd顾名思义它就是一个用于事件通知的fd.多用于用户态进程中多线程之间相互通知,也可用于内核事件通知
  
## 应用场景

  如上图所示,在相同进程中,**乙线程无事件可处理时会进行休眠等待**.一旦来事件需要处理,**甲线程会通过eventfd通知乙线程唤醒**并处理事件.

## [](https://zhenkunhuang.github.io/2019/04/20/linux-eventfd/#函数原型 "函数原型")函数原型

**SYNOPSIS**
```
#include <sys/eventfd.h>
int eventfd(unsigned int initval, int flags);
```
**DESCRIPTION**

| FLAG             | 说明                            |
| ---------------- | --------------------------------------------------- |
| **EFD_CLOEXEC**  | 该标志位设置后,当执行exec族函数时,**会自动关闭eventfd.防止越权和造成fd leak** |
| **EFD_NONBLOCK** | 该标志位设置后,执行**IO操作时不会阻塞**,会立即返回.                      |

**RETURN VALUE**  
  On success, eventfd() returns a new eventfd file descriptor  
  On error, -1 is returned and errno is set to indicate the error

**注意:eventfd的read和write函数的形参buf要用uint64_t类型**


## 例子

这里以C语言为例.
  
```
/*本例子的流程为: 
* 1.主线程等待终端输入字符串
* 2.readThread线程监听evnetfd,休眠等待事件唤醒
* 3.终端输入字符串,主线程通过写入eventfd唤醒readThread线程
* 4.readThread线程从监听eventfd中唤醒,处理事务(打印终端输入的字符串)
* 5.goto 1
*/
#include <stdio.h>
#include <sys/eventfd.h>
#include <pthread.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

#define true 1
#define false 0

typedef unsigned char boolean ;

struct Transfer_Data {
	int eventFd;
	char *msg;
	boolean exit;
};
void *thread_func(void *arg) {  
	struct Transfer_Data *ptd = (struct Transfer_Data *)arg;
	uint64_t counter;
	int err = 0;
	while (ptd -> exit == false) {
		printf("wait for event\n");
		//读取事件,如果无事件读取,会将线程状态置为TASK_INTERRUPTIBLE进行休眠,等待唤醒
		if ((err = read(ptd -> eventFd, &counter, sizeof(uint64_t))) < 0) {	
			printf("read eventfd fail reason: %s\n",strerror(errno));
			ptd -> exit = true;
			break;
		}
		printf("wake up, read the msg : %s\n", ptd -> msg);
	}
	return NULL;
}  

int main(int argc __unused, char **argv __unused) {
	int err;
	uint64_t inc = 1;
	char msg[128] = { 0 };  //作为例子,这里buffer只用了128字节.实际应以需求为准
	
	int mWakeEventFd = eventfd(0, EFD_CLOEXEC); //创建eventfd,设置标志位为EFD_CLOEXEC
	if (mWakeEventFd < 0) {
		printf("Create eventfd fail reason: %s\n",strerror(errno));
		return -1;
	}

	pthread_t readThread;
	struct Transfer_Data td = {mWakeEventFd, msg, false};       //构建要传输的数据:eventfd, msg, exit标志位
	err = pthread_create(&readThread, NULL, thread_func, &td);  //创建readThread线程
	if (err != 0) {
		printf("Create thread fail reason: %s\n", strerror(err));
		return -1;
	}

	while (td.exit == false) {
		fgets(msg, sizeof(msg) / sizeof(*msg), stdin);
		if (write(mWakeEventFd, &inc, sizeof(uint64_t)) < 0) {  //写入事件,唤醒readThread线程
			printf("write eventfd fail reason: %s\n", strerror(err));
			goto EXIT;
		}
	}
EXIT:
	td.exit = true;
	pthread_join(readThread, NULL);
	close(mWakeEventFd);
	return 0;
}
```

**运行结果**
```
generic_arm64:/ # example
wait for event
Hello World	  /*<--- 这是终端输入的字符串*/
wake up, read the msg : Hello World
wait for event