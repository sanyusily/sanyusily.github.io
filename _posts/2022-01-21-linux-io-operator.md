---
layout: article
title: 'Linux 知识之IO操作 '
tags: Linux AOSP
---

# Linux知识之IO操作 

## 简介

  IO顾名思义就是input/output.俗话说linux一切皆文件,对文件的操作不外乎就读和写.这里我们来介绍read和write函数.

## 函数原型

### read

**SYNOPSIS**
```
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
```
**RETURN VALUE**  
  On success, the number of bytes read is returned(zero indicates end of file)  
  On error, -1 is returned, and errno is set appropriately

### write

**SYNOPSIS**
```
#include <unistd.h>
ssize_t write(int fd, const void *buf, size_t count);
```
**RETURN VALUE**  
  On success, the number of bytes written is returned(zero indicates nothing was written)  
  On error, -1 is returned, and errno is set appropriately


## 例子

  这里以C语言为例.
  ```
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

int main(int argc __unused, char **argv __unused) {
	int fd[2] = { 0 };
	char *sendMsg = "Hello World";
	char recvMsg[128] = { 0 };	//作为例子,这里buffer只用了128字节.实际应以需求为准
	
	if (pipe(fd) < 0) {	        //管道,fd[0] 为读管道,fd[1] 为写管道
		printf("Create pipe fail, reason : %s\n", strerror(errno));
		return -1;
	}

	if (write(fd[1], sendMsg, strlen(sendMsg) + 1) < 0) {		
		printf("write fail, reason : %s\n", strerror(errno));
		return -1;
	}

	
	if (read(fd[0], recvMsg, sizeof(recvMsg)/sizeof(*recvMsg)) < 0) {		
		printf("read fail, reason : %s\n", strerror(errno));
		return -1;
	}

	printf("read message success : %s\n", recvMsg);
	
	//注意打开fd 和关闭fd 应成对出现,防止fd泄露.
	close(fd[0]);
	close(fd[1]);
	return 0;
}
```

**运行结果**
```
generic_arm64:/ # example
example
read message success : Hello World
```
 ## 关于errno

  **如果你对errno这个错误码值赋值流程感兴趣,可以继续往下看,需要一点汇编基础.否则本章完.**  
  **以下基于ARMv8 64bit体系架构分析**

**read.S**

```
#include <private/bionic_asm.h>
ENTRY(read)
    mov     x8, __NR_read       //read函数的系统调用号赋值给x8寄存器
    svc     #0                  //svc中断,进入系统调用

    cmn     x0, #(MAX_ERRNO + 1)
    cneg    x0, x0, hi
    b.hi    __set_errno_internal
    ret
END(read)
```

**系统调用返回的错误码都是相反数,这里以权限不足的错误码EACCES(13)为例来分析该流程.**

**1.系统调用read返回错误码EACCES的相反数-13,并将该值保存在x0寄存器.**

**2.cmn比较指令做加法操作,会引起PSTATE寄存器C标志位发生变化**  
**-13的16进制:0xFFFF FFFF FFFF FFF3,MAX_ERRNO + 1的16进制:0x1000.**  
**两者相加,会导致-13的最高位符号位溢出,因此PSTATE寄存器的C标志位 置1.**

**3.hi条件码代表PSTATE的C标志位为1且Z标志位为0.因此此时hi条件code为true**

**4.cneg条件否定指令,cneg x0,x0,hi 等价于if hi == true,then x0 = -x0 else x0 = x0**  
**因此此处指令执行完后x0 = -x0 也就是x0 = -(-13) = 13**

**5.b.hi条件跳转指令,当hi条件码成立,跳转执行__set_errno_internal函数,此处会跳转执行该函数**  

```
extern "C" __LIBC_HIDDEN__ long __set_errno_internal(int n) {
  errno = n;	//形参n的值为x0寄存器,也就是错误码13,此处完成对errno的赋值
  return -1;	//返回-1,该值会赋给x0寄存器,因此read函数发生错误时,得到的返回值为-1
}
```

-----------------------------------本节完-------------------------------------


**参考**  Linux基础知识之错误码表如下：

**这里列出** **常见的Linux错误码** **,方便核对系统调用发生err时,出错的原因是什么.**  
**你也可以在代码中使用** **strerror** **打印错误描述.**
```
#include <errno.h>
#include <string.h>

printf("err reason : %s\n", strerror(errno));
```
**错误码表**
```
// Errno-base.h (kernel-4.9\include\uapi\asm-generic)
#define	EPERM		 1	/* Operation not permitted */
#define	ENOENT		 2	/* No such file or directory */
#define	ESRCH		 3	/* No such process */
#define	EINTR		 4	/* Interrupted system call */
#define	EIO		 5	/* I/O error */
#define	ENXIO		 6	/* No such device or address */
#define	E2BIG		 7	/* Argument list too long */
#define	ENOEXEC		 8	/* Exec format error */
#define	EBADF		 9	/* Bad file number */
#define	ECHILD		10	/* No child processes */
#define	EAGAIN		11	/* Try again */
#define	ENOMEM		12	/* Out of memory */
#define	EACCES		13	/* Permission denied */
#define	EFAULT		14	/* Bad address */
#define	ENOTBLK		15	/* Block device required */
#define	EBUSY		16	/* Device or resource busy */
#define	EEXIST		17	/* File exists */
#define	EXDEV		18	/* Cross-device link */
#define	ENODEV		19	/* No such device */
#define	ENOTDIR		20	/* Not a directory */
#define	EISDIR		21	/* Is a directory */
#define	EINVAL		22	/* Invalid argument */
#define	ENFILE		23	/* File table overflow */
#define	EMFILE		24	/* Too many open files */
#define	ENOTTY		25	/* Not a typewriter */
#define	ETXTBSY		26	/* Text file busy */
#define	EFBIG		27	/* File too large */
#define	ENOSPC		28	/* No space left on device */
#define	ESPIPE		29	/* Illegal seek */
#define	EROFS		30	/* Read-only file system */
#define	EMLINK		31	/* Too many links */
#define	EPIPE		32	/* Broken pipe */
#define	EDOM		33	/* Math argument out of domain of func */
#define	ERANGE		34	/* Math result not representable */

// Errno.h (kernel-4.9\include\uapi\asm-generic)
#define	EDEADLK		35	/* Resource deadlock would occur */
#define	ENAMETOOLONG	36	/* File name too long */
#define	ENOLCK		37	/* No record locks available */
#define	ENOSYS		38	/* Invalid system call number */
#define	ENOTEMPTY	39	/* Directory not empty */
#define	ELOOP		40	/* Too many symbolic links encountered */
#define	EWOULDBLOCK	EAGAIN	/* Operation would block */
#define	ENOMSG		42	/* No message of desired type */
#define	EIDRM		43	/* Identifier removed */
#define	ECHRNG		44	/* Channel number out of range */
#define	EL2NSYNC	45	/* Level 2 not synchronized */
#define	EL3HLT		46	/* Level 3 halted */
#define	EL3RST		47	/* Level 3 reset */
#define	ELNRNG		48	/* Link number out of range */
#define	EUNATCH		49	/* Protocol driver not attached */
#define	ENOCSI		50	/* No CSI structure available */
#define	EL2HLT		51	/* Level 2 halted */
#define	EBADE		52	/* Invalid exchange */
#define	EBADR		53	/* Invalid request descriptor */
#define	EXFULL		54	/* Exchange full */
#define	ENOANO		55	/* No anode */
#define	EBADRQC		56	/* Invalid request code */
#define	EBADSLT		57	/* Invalid slot */
#define	EDEADLOCK	EDEADLK
#define	EBFONT		59	/* Bad font file format */
#define	ENOSTR		60	/* Device not a stream */
#define	ENODATA		61	/* No data available */
#define	ETIME		62	/* Timer expired */
#define	ENOSR		63	/* Out of streams resources */
#define	ENONET		64	/* Machine is not on the network */
#define	ENOPKG		65	/* Package not installed */
#define	EREMOTE		66	/* Object is remote */
#define	ENOLINK		67	/* Link has been severed */
#define	EADV		68	/* Advertise error */
#define	ESRMNT		69	/* Srmount error */
#define	ECOMM		70	/* Communication error on send */
#define	EPROTO		71	/* Protocol error */
#define	EMULTIHOP	72	/* Multihop attempted */
#define	EDOTDOT		73	/* RFS specific error */
#define	EBADMSG		74	/* Not a data message */
#define	EOVERFLOW	75	/* Value too large for defined data type */
#define	ENOTUNIQ	76	/* Name not unique on network */
#define	EBADFD		77	/* File descriptor in bad state */
#define	EREMCHG		78	/* Remote address changed */
#define	ELIBACC		79	/* Can not access a needed shared library */
#define	ELIBBAD		80	/* Accessing a corrupted shared library */
#define	ELIBSCN		81	/* .lib section in a.out corrupted */
#define	ELIBMAX		82	/* Attempting to link in too many shared libraries */
#define	ELIBEXEC	83	/* Cannot exec a shared library directly */
#define	EILSEQ		84	/* Illegal byte sequence */
#define	ERESTART	85	/* Interrupted system call should be restarted */
#define	ESTRPIPE	86	/* Streams pipe error */
#define	EUSERS		87	/* Too many users */
#define	ENOTSOCK	88	/* Socket operation on non-socket */
#define	EDESTADDRREQ	89	/* Destination address required */
#define	EMSGSIZE	90	/* Message too long */
#define	EPROTOTYPE	91	/* Protocol wrong type for socket */
#define	ENOPROTOOPT	92	/* Protocol not available */
#define	EPROTONOSUPPORT	93	/* Protocol not supported */
#define	ESOCKTNOSUPPORT	94	/* Socket type not supported */
#define	EOPNOTSUPP	95	/* Operation not supported on transport endpoint */
#define	EPFNOSUPPORT	96	/* Protocol family not supported */
#define	EAFNOSUPPORT	97	/* Address family not supported by protocol */
#define	EADDRINUSE	98	/* Address already in use */
#define	EADDRNOTAVAIL	99	/* Cannot assign requested address */
#define	ENETDOWN	100	/* Network is down */
#define	ENETUNREACH	101	/* Network is unreachable */
#define	ENETRESET	102	/* Network dropped connection because of reset */
#define	ECONNABORTED	103	/* Software caused connection abort */
#define	ECONNRESET	104	/* Connection reset by peer */
#define	ENOBUFS		105	/* No buffer space available */
#define	EISCONN		106	/* Transport endpoint is already connected */
#define	ENOTCONN	107	/* Transport endpoint is not connected */
#define	ESHUTDOWN	108	/* Cannot send after transport endpoint shutdown */
#define	ETOOMANYREFS	109	/* Too many references: cannot splice */
#define	ETIMEDOUT	110	/* Connection timed out */
#define	ECONNREFUSED	111	/* Connection refused */
#define	EHOSTDOWN	112	/* Host is down */
#define	EHOSTUNREACH	113	/* No route to host */
#define	EALREADY	114	/* Operation already in progress */
#define	EINPROGRESS	115	/* Operation now in progress */
#define	ESTALE		116	/* Stale file handle */
#define	EUCLEAN		117	/* Structure needs cleaning */
#define	ENOTNAM		118	/* Not a XENIX named type file */
#define	ENAVAIL		119	/* No XENIX semaphores available */
#define	EISNAM		120	/* Is a named type file */
#define	EREMOTEIO	121	/* Remote I/O error */
#define	EDQUOT		122	/* Quota exceeded */
#define	ENOMEDIUM	123	/* No medium found */
#define	EMEDIUMTYPE	124	/* Wrong medium type */
#define	ECANCELED	125	/* Operation Canceled */
#define	ENOKEY		126	/* Required key not available */
#define	EKEYEXPIRED	127	/* Key has expired */
#define	EKEYREVOKED	128	/* Key has been revoked */
#define	EKEYREJECTED	129	/* Key was rejected by service */
#define	EOWNERDEAD	130	/* Owner died */
#define	ENOTRECOVERABLE	131	/* State not recoverable */
#define ERFKILL		132	/* Operation not possible due to RF-kill */
#define EHWPOISON	133	/* Memory page has hardware error */
```










