**一. docker概念**

  [1. 容器与虚拟机有什么区别?](https://github.com/Myrecord/Docker/blob/master/README.md)
  [2. docker架构如何工作？](https://github.com/Myrecord/Docker/blob/master/README.md)
  [3. docker镜像如何实现？](https://github.com/Myrecord/Docker/blob/master/README.md)
  [4. docker中的网络模型](https://github.com/Myrecord/Docker/blob/master/README.md)
  [5. 数据如何持久化？](https://github.com/Myrecord/Docker/blob/master/README.md)
  [6. Dockerfile是什么？](https://github.com/Myrecord/Docker/blob/master/README.md)
  [7. docker私有仓库](https://github.com/Myrecord/Docker/blob/master/README.md)
  [8. 如何限制容器默认启动资源？](https://github.com/Myrecord/Docker/blob/master/README.md)

**二. docker使用**
    
  [1.基础命令] (https://github.com/Myrecord/Docker/blob/master/README.md)
  
  
----
### 一. docker概念
##### 1. 容器与虚拟机有什么区别？
![1.jpg](https://github.com/Myrecord/Docker/blob/master/1.jpg)
* 虚拟机是运行在宿主机操作系统之上，在宿主机操作系统上模拟宿主机的硬件、操作系统、内核用来进行隔离。继承在一台宿主机的多个虚拟机之间，内核、系统、用户空间互相不受影响，哪怕其中一台虚拟机挂掉也不会影响其他虚拟机的运行，可以做到彻底的隔离，但对于资源的开销比较大。
* 容器是在虚拟机的结构中去除虚拟机的内核层(系统)，共享宿主机的硬件资源，在宿主机的操上模拟多个用户空间，给进程提供运行环境，保护进程不被其他进程干扰，所以容器是基于用户空间进行隔离。用户空间如何隔离？每个用户空间中都有独立的UTS(主机名称)、Mount(文件系统)、IPS(通信消息队列)、PID(进程)、USER(用户)、NETWORK(网络)，这6种命名空间统称为**Namespaces**,Linux内核在3.8版本中加入了USER命名空间，所以想要使用Docker内核最低3.10
* 相比虚拟机容器隔离并不是很彻底，因为容器之间共享一个内核空间、硬件资源。假设一个容器使用的CPU为100%就会影响其他的容器。但Linux内核提供对资源限制分配的功能叫做Cgroups。它可以对系统资源进行限制分配、比如一个容器可以使用多少CPU核心、多少内存等。
* Linux系统内核已经实现的容器技术，早期**LXC(Linux Container)** 是用来创建管理容器工具。Docker是LXC的二次封装版，docker的出世大大简化了容器使用的难度并做到一次构建到处运行，保证环境的统一提高效率，但Docker自身对与分发、管理容器、运行顺序还是有一定缺陷，不得不借助于编排工具，目前随着Docker发展，Docker容器引擎从libcontainer变更为Runc。
##### 2. docker架构如何工作？
![2.jpg](https://github.com/Myrecord/Docker/blob/master/2.jpg)
* Docker是C/S架构模式运行,支持三种模式监听:ipv4、ipv6、Unixsocket文件(默认),Docker在后台启动Docker daemon守护进程监听本地的socket文件,Docker client执行命令启动容器时Docker server默认使用https从仓库中获取，如果本地不存在则从dokcer hub中获取镜像文件,镜像文件是分层构建,底层的镜像文件为只读模式,最终将多层镜像文件汇总成一个读写层提供给容器运行并存放到本地，一个仓库只包含一个镜像但可以是多个不同版本,例如：&lt;仓库名&gt;:&lt;标签&gt;，或者&lt;仓库名称&gt;/&lt;标签&gt;.
##### 3. Docker镜像如何实现？
![3.jpg](https://github.com/Myrecord/Docker/blob/master/3.jpg)
* 镜像其实就是一个包含完整的操作系统，在最低层为bootfs、其次rootfs(系统)都为只读模式,一个镜像必须包含基础操作系统，我们构建镜像过程中，实际上是在前两层之上制作，每层之间是独立的，例如：在第三层添加emacs并在第四层添加apache，第四层中并不包含amecs，最后其实是通过**联合挂载**的方式将所有层整合并提供一个读写层(最上层)容器.
* 早期docker版本使用的**Aufs(高级多层统一文件系统)** 新版本中docker默认使用文件系统为**overla2(联合挂载)** 层级可以复用、追加，假设同时有两个版本的nginx镜像底层会有相同依赖的层级不需要创建新的层级，也可以添加新的功能，但如果删除其中一个nginx镜像，并不会删除依赖的层级。
##### 4. docker中的网络模型
![4.jpg](https://github.com/Myrecord/Docker/blob/master/4.jpg)
* docker支持多种网络模式使用docker info命令可以查看，默认有三种bridge、host、none如果不指定，使用bridge作为默认的网络，在安装完docker后会创建一个docker0的网桥，docker0不仅是一个虚拟网卡在容器内部也充当交换机。容器创建后，会自动创建**一对网卡**，一端在容器内部，一端在物理机中，并且生成在物理机中的一端虚拟网卡接口都被插在docker0网桥中，通过使用brctl show命令查看。
* bridge：容器之间通信网络接口都连接到docker0网桥中，要想外部访问容器，就需要进行DNAT模式，docker会在iptables中自动创建转发规则，这种模型显然降低带宽的质量，但在测试环境比较适合。
* host：共享宿主机的网络，容器之间访问将不在使用docker0，而是与宿主机使用
同一个IP地址，在容器内开放的端口相当于开发宿主机的端口
* none：不设置任何网络模型，如果启动容器不需要网络可指定，容器内部只有一个lo接口。
* 另外一种网络模型容器之间也可以共享网络，两个容器使用同一个网络命名空间容器通过lo地址可互相访问，但文件系统、主机名称并不共享。
##### 5. 数据如何持久化？
* 很多有状态的服务需要保存数据比如Mysql、Redis，默认容器产生的数据会随着容器删除而被删除，docker中提供对数据持久保存的方式，容器中的文件或目录与宿主机中的目录文件进行绑定，在宿主机对数据修改或者删除时容器也会随之而改变。
* 同时多个容器可以同时共享挂载一个数据盘或复制前一个容器的数据盘
