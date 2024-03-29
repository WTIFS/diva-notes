##### 核心技术

1. 容器化技术：Docker 使用 Linux 容器技术（LXC）实现虚拟化，从而将应用程序及其所有依赖项打包成一个轻量级的、可移植的容器。与传统虚拟化技术不同，Docker 容器不需要额外的操作系统，而是共享主机操作系统的内核，因此更加轻量级和快速。
2. 镜像管理：Docker 使用镜像来构建容器，镜像是一个静态的文件，其中包含了一个完整的文件系统和应用程序运行所需的所有依赖项。可以通过 Dockerfile 文件来构建镜像，或者从 Docker Hub 等镜像仓库中获取预先构建好的镜像。
3. 分层存储：Docker 镜像使用分层存储的方式进行构建。每个镜像层都包含了一个文件系统的快照，并且镜像层可以共享。这种方式可以大大减小镜像的体积，加快构建速度，并使得镜像的维护更加简单。
4. 网络管理：Docker 使用虚拟网络来管理容器之间的通信。每个容器都可以分配一个唯一的 IP 地址，并可以使用 Docker 网络插件来扩展网络功能。例如，可以创建一个自定义的 Docker 网络，并将多个容器连接到该网络，实现容器之间的通信。
5. 数据持久化：Docker 使用卷（Volume）来进行数据持久化。卷是一个特殊的文件系统，可以在容器之间共享数据，并且可以在容器删除后仍然保留。可以将主机上的目录挂载为一个卷，以便容器可以访问该目录并在其中写入数据。



##### 虚拟机

虚拟机的原理是在主机操作系统上，又模拟了一层操作系统，以及硬件、内核、网络等资源

与虚拟机相比，Docker 容器是基于宿主机操作系统的轻量级进程隔离技术，它们共享主机的内核和一些资源，如文件系统、网络、进程等。由于容器不需要模拟自己的操作系统内核和硬件，因此资源消耗和启动时间都比虚拟机更少、更快。



##### LXC

Linux 容器技术（LXC）是一种轻量级的虚拟化技术，它允许用户在同一台宿主机上运行多个相互隔离的容器，每个容器都有自己的文件系统、网络、进程和用户空间等资源。

LXC 的核心是基于 Linux 内核的 Cgroups（控制组）和命名空间（Namespace）技术。Cgroups 用于限制容器可以使用的系统资源，例如CPU、内存、磁盘和网络带宽等。命名空间则用于隔离不同容器之间的进程、网络和文件系统（进程和网络实际上也是文件）等资源，使得不同的容器可以互相独立、隔离和安全地运行在同一台物理机上。

LXC 提供了一个简单而强大的API，用户可以通过该API创建、启动、停止、销毁和管理容器。LXC 还支持快照和恢复容器，使得用户可以方便地备份和还原容器的状态。

与传统的虚拟机相比，LXC 具有更低的开销和更高的性能，因为它不需要模拟整个操作系统和硬件环境，而是共享宿主机的内核和系统资源。因此，LXC 被广泛应用于云计算、DevOps、容器化应用程序和测试环境等场景中。



##### Namespace

在 Linux 系统中，Namespace 是一种资源隔离的机制，它可以将一组系统资源隔离开来，使得它们只在指定的 Namespace 中可见和可操作。Docker 利用了 Linux Kernel 的 Namespace 机制来实现容器的隔离，将容器的进程、文件系统、网络、IPC 等资源分别隔离在不同的 Namespace 中，从而实现了容器之间的隔离。

Docker 使用以下 6 种 Namespace：

1. Mount Namespace：每个容器有自己的文件系统，Mount Namespace 隔离了各个容器的文件系统，使得容器之间看不到对方的文件系统。
2. PID Namespace：每个容器有自己的进程号（PID），PID Namespace 隔离了各个容器的进程树，使得容器之间看不到对方的进程。
3. Network Namespace：每个容器有自己的网络空间，Network Namespace 隔离了各个容器的网络设备、IP 地址、路由表等网络资源，使得容器之间看不到对方的网络信息。
4. IPC Namespace：每个容器有自己的进程间通信（IPC）资源，IPC Namespace 隔离了各个容器的消息队列、共享内存、信号量等 IPC 资源，使得容器之间看不到对方的 IPC 资源。
5. UTS Namespace：每个容器有自己的主机名和域名，UTS Namespace 隔离了各个容器的主机名和域名，使得容器之间看不到对方的主机名和域名。
6. User Namespace：每个容器有自己的用户和组 ID，User Namespace 隔离了各个容器的用户和组 ID，使得容器之间看不到对方的用户和组 ID。



##### k8s

Kubernetes 是一个容器编排引擎，用来管理 Docker 容器集群的。为应用程序提供负载平衡、自动伸缩、自动恢复等功能。

