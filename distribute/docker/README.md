## Docker

[官方文档](https://docs.docker.com/)

### Docker是如何实现环境隔离的？

Linux 的 Namespace 机制。比如 Linux User 的 Namespace，也是使用 Namespace 来对不同的 User 进行隔离。

Docker的隔离实现机制与此类似。在创建容器进程时，指定了这个进程所需要启用的一组 Namespace 参数。这样，容器就只能“看”到当前 Namespace 所限定的资源、文件、设备、状态，或者配置。而对于宿主机以及其他不相关的程序，它就完全看不到了。

**容器是一个“单进程”模型。**由于一个容器的本质就是一个进程，用户的应用进程实际上就是容器里 PID=1 的进程，也是其他后续创建的所有进程的父进程。

**所以说，容器，其实是一种特殊的进程而已。**

### 虚拟机 VS Docker

![虚拟机 VS Docker 原理](../../src/distribute/docker_principle_0.png)

这幅图的左边，画出了虚拟机的工作原理。其中，名为 Hypervisor 的软件是虚拟机最主要的部分。它通过硬件虚拟化功能，模拟出了运行一个操作系统需要的各种硬件，比如 CPU、内存、I/O 设备等等。然后，它在这些虚拟的硬件上安装了一个新的操作系统，即 Guest OS。

而这幅图的右边，则用一个名为 Docker Engine 的软件替换了 Hypervisor。**这副图这样画却并不严谨。**

在使用 Docker 的时候，并没有一个真正的“Docker 容器”运行在宿主机里面。Docker 项目帮助用户启动的，还是原来的应用进程，都由宿主机操作系统统一管理，只不过在创建这些进程时，Docker 为它们加上了各种各样的 Namespace 参数。**Namespace 技术实际上修改了应用进程看待整个计算机“视图”，即它的“视线”被操作系统做了限制，只能“看到”某些指定的内容**。而 Docker 项目在这里扮演的角色，更多的是旁路式的辅助和管理工作。所以，上面的原理图应该像下面这样画：

![虚拟机 VS Docker 原理](../../src/distribute/docker_principle_1.jpeg)

用户应用运行在虚拟机里面，它对宿主机操作系统的调用就不可避免地要经过虚拟化软件的拦截和处理，这本身又是一层性能损耗，尤其对计算资源、网络和磁盘 I/O 的损耗非常大。而相比之下，容器化后的用户应用，依然还是一个宿主机上的普通进程，使用 Namespace 作为隔离手段的容器并不需要单独的 Guest OS，这就使得容器额外的资源占用几乎可以忽略不计。

**使用 Namespace 作为隔离手段的劣势：**

隔离得不彻底。

既然容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是同一个宿主机的操作系统内核。如果要在 Windows 宿主机上运行 Linux 容器，或者在低版本的 Linux 宿主机上运行高版本的 Linux 容器，都是行不通的。

在 Linux 内核中，有很多资源和对象是不能被 Namespace 化的，最典型的例子就是：时间。如果你的容器中的程序使用 settimeofday(2) 系统调用修改了时间，整个宿主机的时间都会被随之修改，这显然不符合用户的预期。

### Docker资源限制

借助 Linux Cgroups（Linux Control Group）限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。

**一个正在运行的 Docker 容器，其实就是一个启用了多个 Linux Namespace 的应用进程，而这个进程能够使用的资源量，则受 Cgroups 配置的限制。**

### Docker镜像（rootfs）



### Docker使用小技巧

使用[阿里云加速镜像](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)可大大提高拉取官方镜像的速度

