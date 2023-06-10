---
layout: article
title: 'Linux 知识之socketpair'
tags: socketpair socket linux
---

## 简介

  socketpair会创建一对无名套接字的描述符,具有全双工通信特性(描述符可读也可写),他的域只能为AF_UNIX(本地).

## 应用场景

  socketpair主要用于C/S模式的进程间通讯.**由于binder通信具有透传fd的特性,使得socketpair不再受限于亲缘关系进程**

## 函数原型

**SYNOPSIS**

```
#include <sys/types.h>    
#include <sys/socket.h>

int socketpair(int domain, int type, int protocol, int sv[2]);
```

**DESCRIPTION**

| TYPE               | 说明                                             |
| ------------------ | ---------------------------------------------- |
| **SOCK_STREAM**    | TCP协议,提供有序,面向连接,双向,可靠的传输                       |
| **SOCK_DGRAM**     | UDP协议,提供面向无连接,不可靠的数据包传输                        |
| **SOCK_SEQPACKET** | 提供有序,双向,可靠的传输                                  |
| **EFD_CLOEXEC**    | 该标志位设置后,当执行exec族函数时,**会自动关闭fd.防止越权和造成fd leak** |
| **EFD_NONBLOCK**   | 该标志位设置后,执行**IO操作时不会阻塞**,会立即返回.                 |

想必大家对SOCK_STREAM与SOCK_SEQPACKET有疑问,这2个描述基本一样那他们有没区别呢?我们直接查看源码

```
// Af_unix.c (kernel-4.9\net\unix)

static int unix_seqpacket_sendmsg(struct socket *sock, struct msghdr *msg,
				  size_t len)
{
	int err;
	struct sock *sk = sock->sk;

	err = sock_error(sk);
	if (err)
		return err;

	if (sk->sk_state != TCP_ESTABLISHED)
		return -ENOTCONN;

	if (msg->msg_namelen)
		msg->msg_namelen = 0;

	return unix_dgram_sendmsg(sock, msg, len);
}
```

可以看到SOCK_SEQPACKET类型的发送数据流走的是SOCK_DGAM(UDP协议)的发送接口,该接口不具备TCP协议的重传功能

**RETURN VALUE**  
  On success, zero is returned  
  On error, -1 is returned, and errno is set appropriately

## 例子

![fork_fd.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b8b0c8a09ea43f99beaace5e844a080~tplv-k3u1fbpfcp-watermark.image?)

```
/*本例子的流程为: 
* 1. 调用socketpair得到一对双向fd
* 2. 调用fork,产生子进程
* 3. 父进程向子进程发送信息,子进程接收并打印
* 4. 子进程向父进程发送信息,父进程接收并打印
*/
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <sys/types.h>         
#include <sys/socket.h>
#include <signal.h>

void childLoop(int fd) {
	char recvMsg[128] = { 0 };
	char *sendMsg = "I am Child";
	if (read(fd, recvMsg, sizeof(recvMsg) / sizeof(*recvMsg)) < 0) {
		printf("%s read fail reason: %s\n", __func__, strerror(errno));
		return;
	}
	
	printf("%s read msg : %s \n", __func__, recvMsg);

	
	if (write(fd, sendMsg, strlen(sendMsg) + 1) < 0) {
		printf("%s write fail reason: %s\n", __func__, strerror(errno));
		return;
	}
}

void parentLoop(int fd) {	
	char recvMsg[128] = { 0 };
	char *sendMsg = "I am Father";
	
	if (write(fd, sendMsg, strlen(sendMsg) + 1) < 0) {
		printf("%s write fail reason: %s\n", __func__, strerror(errno));
		return;
	}
	
	if (read(fd, recvMsg, sizeof(recvMsg) / sizeof(*recvMsg)) < 0) {
		printf("%s read fail reason: %s\n", __func__, strerror(errno));
		return;
	}
	
	printf("%s read msg : %s \n", __func__, recvMsg);	
}
int main(int argc __unused, char **argv __unused) {
	int pid;
	int fd[2];
	int err = 0;
	signal(SIGCHLD, SIG_IGN); //忽略子进程停止信号,防止产生僵尸进程
	err = socketpair(AF_UNIX, SOCK_SEQPACKET | SOCK_CLOEXEC, 0, fd);

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
generic_arm64:/ # example
example
childLoop read msg : I am Father
parentLoop read msg : I am Child