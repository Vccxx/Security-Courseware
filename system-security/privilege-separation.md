#  特权分离

### 哈尔滨工业大学 网络与信息安全 张宇 2016

---

本节课学习特权分离概念，一种web服务器安全体系OKWS。


## 访问控制

[访问控制（access control (AC)）](https://en.wikipedia.org/wiki/Access_control)：对资源消耗，访问，使用的的选择性限制。

- “访问”包含三个元素：主体，对象（客体），操作
- 认证（authentication）过程识别主体的真正身份
	- “你知道的”：例如网站口令
	- “你拥有的”：例如手机sim卡
	- “你是什么”：例如指纹，虹膜
	- 恢复机制：口令恢复通道（email，短信）；你所知道的其他知识
	- 双因子（two-factor）认证：例如在ATM上取款需要银行卡和口令
- 对访问资源的许可称为“授权（authorization）”
- 强制访问控制（Mandatory Access Control，MAC），系统强制执行，例如特权用户与非特权用户
- 自主访问控制（Discretionary Access Control，DAC），资源拥有者自定义，例如文件权限

实现访问控制依赖于 [引用监视器（reference monitor）](https://en.wikipedia.org/wiki/Reference_monitor)：在操作系统中，对引用验证机制（reference validation mechanism，RVM）的一个设计需求集合，来强制实现访问控制策略，需满足一下需求：

- 不可绕过（Non-bypassable）：RVM必须始终被调用（always invoked），完全介入（complete mediation）
- 防篡改（tamperproof）：否则，不能保证不可绕过
- 可验证（verifiable）：RVM必须足够小，以进行分析，测试，确保不可绕过

实现引用监视器依赖于 [可信计算基（trusted computing base）](https://en.wikipedia.org/wiki/Trusted_computing_base)：实现安全策略的关键硬件、固件、软件的集合，若发生故障则无法实现安全；区别于其他即使发生故障也不会影响安全的组件

[TOCTOU (time-of-check-time-of-use)](https://en.wikipedia.org/wiki/Time_of_check_to_time_of_use)：指访问控制的检查时刻与使用时刻之间相关资源已改变。

受害程序：

```c
if (access("file", W_OK) != 0) {
   exit(1);
}

fd = open("file", O_WRONLY);
// Actually writing over /etc/passwd
write(fd, buffer, sizeof(buffer));
```

攻击程序：

```c
// After the access check
symlink("/etc/passwd", "file");
// Before the open, "file" points to the password database
```


## UNIX操作系统中的特权

[特权（privilege）](https://en.wikipedia.org/wiki/Privilege_(computing))：允许用户（主体）对一个计算机系统（客体）执行特定操作的授权。

Unix中的主体与客体：

- Unix系统包含一个内核和多个**进程**
- 用**文件**作为所有持久性系统对象的概念
- **进程**有各自的**内存地址空间**
- **进程**与**用户**相关联，**用户**与**文件**访问权限相关联

Unix系统采用强制访问控制，**‘root’**特权用户（UID=0）拥有全部特权，而一般用户不具备的特权包括：

- 调整运行级别，内核选项
- 调整`ulimit`或磁盘份额
- 更改系统文件，或其他用户的文件
- 更改文件的所有者或组,`chown/chgrp`
- 创建或删除用户或组，`adduser/deluser`
- 执行`sbin/`目录中内容
- 绑定小于1024的端口，创建原始套接字
- 启动或终止精灵进程
- 向其他用户的进程发信号
- 创建设备节点，挂载/卸载（`mount/unmount`）磁盘分卷

[文件系统权限（permission）](https://en.wikipedia.org/wiki/File_system_permissions)：作为一种自主访问控制，文件拥有者自主定义访问控制，表示为模式位（mode bit）：


```
   0     123    456    789 bit
  [type][owner][group][other]    
  [-   ][rwx  ][rwx  ][rwx  ]    +—> [s]etuid/gid
   |     |                       |—> s[t]icky
   |     |——> [r]ead, [w]rite, e[x]ecutable
   |
   |——> [-]ordinary, [d]irectory, [l]ink,
   |——> [s]ocket, [p]ipe, [c]haracter, [b]lock
```

- `s[t]icky`位: 在目录上设置粘滞位，目录内文件的所有者或者root才可以删除或移动该文件。例如，临时文件目录`/tmp`，对所有用户可读写，但一个用户不能删除或改动其他用户文件。
- `[s]etuid/setgid`位：执行该文件时，进程`euid`为文件拥有者。例如，`/bin/ping`。


[用户ID与进程](https://en.wikipedia.org/wiki/User_identifier)：当用户登录到系统时创建与其关联的进程

```
|————————————|———————————————————|———————————————|
|            |     Set           |     Get       |
|————————————|———————————————————|———————————————|
| Effective  | login,            |geteuid        |
| UID (euid) | setuid(ruid)      |               |
|            | exec (owner),     |setreuid       |
|            | setreuid(ruid)    |               |
|————————————|———————————————————|———————————————|
| Real UID   | login, setuid,    |setuid, getuid |
| (ruid)     | setreuid(euid)    |               |
|————————————|———————————————————|———————————————|
| Saved UID  | login, exec(euid),|setuid         |
| (suid)     | setuid            |               |
|————————————|———————————————————|———————————————|
```
- 缺省情况下，例如登录后`euid = ruid = suid`
- `euid`：进程多数权限依赖于`euid`，例如新创建文件的拥有者。`euid=0`的进程为特权进程。(Linux中还有`fsuid`)
- `ruid`：进程真正拥有者（启动者），拥有发信号权限
- `suid`：临时保存`euid`，待之后恢复。用于特权用户临时降低特权，之后再恢复特权
- Linux中的`fsuid/gid`用于文件系统访问控制，通常与`euid/gid`一致
- 进程`fork`或`exec`时，继承或保留3个uid
- `setuid()`函数族设置`uid/gid`，但实际情况比较复杂，详见[Setuid Demystified (USENIX Security 2002)](https://www.usenix.org/legacy/events/sec02/full_papers/chen/chen.pdf)
- `sudo`命令：临时以特定用户（缺省root）特权来执行命令。例如，`sudo apt-get`

[特权扩大（priviledge escalation）](https://en.wikipedia.org/wiki/Privilege_escalation)：利用操作系统或软件应用中的bug、设计缺陷或配置疏漏来获得访问被保护资源的特权。

- 垂直特权扩大：即特权提升（priviledge elavation），低特权用户获得高特权用户的特权，通常获得系统管理权。例如，Unix系统中越狱(jailbreakig)打破`chroot`或`jail`限制，以及Andriod中获取root。
	- [`chroot`](https://en.wikipedia.org/wiki/Chroot)命令：
		- 该命令只有root可以执行：`chroot /tmp/guest`, `su guest`
		- 改变当前进程的根目录，将进程文件系统特权限制在指定jail目录下
		- `open("/etc/passwd", "r")` -> 		    `open("/tmp/guest/etc/passwd", "r")` 
- 水平特权扩大：一个用户获得另一用户的特权。例如，在一个网银中，用户甲通过cookie劫持来访问用户乙的银行账户。

[**最小特权原则（principle of least privilege）**](https://en.wikipedia.org/wiki/Principle_of_least_privilege)：限制每个主体只具有执行合法操作所必须的最小特权。

## 特权分离

参考文献：[Preventing Privilege Escalation (USENIX Security 2003)](http://www.citi.umich.edu/u/provos/papers/privsep.pdf)

[特权分离（privilege separation）](https://en.wikipedia.org/wiki/Privilege_separation)：将特权代码最小化，同时不影响功能。

将一个程序划分为多个组件，根据执行特定任务的需求来限制每个组件的特权。攻破某个组件只能获取该组件特权，而不会得到其他特权。因此，也称为“compartmentalisation（分解法）”。已被应用于OpenSSH, Google Chromium浏览器。

- 服务分为一个特权父进程monitor和多个非特权子进程slave
- slave去权限通过UID/GID实现
- slave与monitor通过socketpair通信
- slave请求monitor提供三种权限：信息，能力，UID/GID改变
	- 信息：slave向monitor发送信息请求(请求类型，长度)
	- 能力：通过file descriptor passing机制来访问特权文件
	- UID改变：保留slave状态，终止slave，启动有新UID的新slave

---

## OKWS

参考文献：[Building Secure High-Performance Web Services with OKWS (USENIX ATC 2004) [local]](supplyments/okws.pdf)

实验系统`zookws`源自于OKWS，一个特权分离web服务器。OKWS实现以下目标：

1. `chroot`所有服务到一个jail目录
	- 以操作系统级jail来隐藏敏感文件。在jail中，每个进程的特权只包括，在启动时读取共享库，以及在异常终止时dump core。所有服务从来不访问文件系统，也没有该特权。
- 每个服务以一个唯一的非特权用户来运行
	- 即使在`chroot`环境下，特权用户可以绑定端口，跟踪系统调用或发送信号。
- 服务器进程应只具有最小数据库访问特权
	- 在web服务和数据库之间插入一个结构化RPC接口，替代直接SQL查询接口，使用简单的认证机制来在不同进程划分上划分数据库访问方法。
- 独立的功能应分离到独立的进程
	- 每个web服务作为一个独立进程。否则，攻击者可以查看内存中数据结构，其中可能包括会话管理数据，或用于认证的秘密token；攻击者劫持存在的数据库连接；监控改变网络流量。

OKWS体系结构：

- `okld`：启动/重启定制服务
- `okd`：将到来的请求分发到适合的服务
- `pubd`：具有有限权限：读取磁盘中配置文件和HTML模板文件
- `oklogd`：专门的日志精灵进程在磁盘上写入日志
- `svc1/2/3`：定制服务

```
      |     +———————+      +————————+   
      |     |  okld +—————>| oklogd | 
      |     +———+———+      +————————+             
      |         |———————+       
      |         |       |               
    HTTP    +———v———+   |  +———————+     +——————————+
 <====|===> |  okd  |<——+—>|  svc1 |     | Database |
      |     +———+———+   |  +————+——+     +————^—————+ 
      |         |       |       +—————+       |
      |     +———v———+   |  +———————+  |   +———+——+
      |     |  pubd |   +—>|  svc2 +——+——>| Data2 |
      |     +———————+   |  +———————+      +———————+
      |                 |  +———————+      +———————+   
      |                 +—>|  svc3 +—————>| Data3 |
                           +———————+      +———————+
```

考虑到安全和性能之间的取舍，每个服务一个进程，而不是每个客户一个进程。OKWS权限分离方案：

```
|—————————|——————————————————|———————————————|——————|——————|
| process |chroot jail       |run directory  |uid   | gid  |
|—————————|——————————————————|———————————————|——————|——————|
| okld    | /var/okws/run    | /             |root  |wheel || pubd    | /var/okws/htdocs | /             |www   |www   |
| oklogd  | /var/okws/log    | /             |oklogd|oklogd|| okd     | /var/okws/run    | /             |okd   |okd   |
| svc1    | /var/okws/run    | /cores/51001  |51001 |51001 |
| svc2    | /var/okws/run    | /cores/51002  |51002 |51002 |
| svc3    | /var/okws/run    | /cores/51003  |51003 |51003 |
|—————————|——————————————————|———————————————|——————|——————|
```

OKWS安全优点：

- 本地文件系统：除了执行定制代码，几乎不访问文件系统
- 其他特权：进程间彼此分离，除了`okld`，没有进程能绑定特权端口
- 数据库访问：所有数据库访问通过RPC通道使用独立鉴别机制；攻击者直接面对RPC协议声明，无法用SQL客户端访问
- 进程隔离与特权分离：将容易出bug的HTTP语法分析与其他敏感区域分离，并赋予最低特权；拥有特权的`okld`只处理信号，不处理其他消息

OKWS安全缺点：

- 目前只支持C++来开发服务；需要安全字符串类来生成输出，不能阻止程序员是使用不安全编程技术
- 一旦被攻破，攻击者可以访问其他用户的在内存中的状态（因为相同服务的用户共享一个进程）
- 仍然受核心库中bug影响

---
