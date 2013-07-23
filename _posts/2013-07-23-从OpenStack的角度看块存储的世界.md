---
layout: default
title: 从OpenStack的角度看块存储的世界
---
# 从OpenStack的角度看块存储的世界

原文：
http://www.infoq.com/cn/articles/block-storage-overview

块存储，简单来说就是提供了块设备存储的接口。用户需要把块存储卷附加到虚拟机或者裸机上以与其交互。这些卷都是持久的：它们可以从运行实例上被解除或者重新附加而数据保持完整不变。

本文重点介绍块存储服务。我们对目前主流的块存储服务提供商和开源的块存储软件做了一个简要分析，希望能给从事块存储开发的工程师对于块存储一个全局的认识。

下面会先介绍常见的单机块设备工具，以建立对块存储的初步印象。

单机块存储

首先，一个硬盘是一个块设备。内核检测到硬盘后，在/dev/下会看到/dev/sda/。为了用一个硬盘来得到不同的分区来做不同的事，我们使用fdisk工具得到/dev/sda1、/dev/sda2等。这种方式通过直接写入分区表来规定和切分硬盘，是最死板的分区方式。

1. LVM & Device-mapper

LVM是一种逻辑卷管理器。通过LVM来对硬盘创建逻辑卷组和得到逻辑卷，要比fdisk方式更加弹性。如果你目前对LVM用途还不熟悉或者不大清楚，可参考以下链接：

通用线程：学习 Linux LVM，第 1 部分

通用线程：学习 Linux LVM，第 2 部分

LVM基于Device-mapper用户程序实现。Device-mapper是一种支持逻辑卷管理的通用设备映射机制，为存储资源管理的块设备驱动提供了一个高度模块化的内核架构。以下链接对Device-mapper架构进行了极好的说明：

Linux 内核中的 Device Mapper 机制
2. SAN & iSCSI

在接触了单机下的逻辑卷管理后，你需要了解SAN，目前主流的企业级存储方式。

大部分SAN使用SCSI协议在服务器和存储设备之间传输和沟通，通过在SCSI之上建立不同镜像层，可以实现存储网络的连接。常见的有iSCSI，FCP，Fibre Channel over Ethernet等。

SAN通常需要在专用存储设备中建立，而iSCSI是基于TCP/IP的SCSI映射，通过iSCSI协议和Linux iSCSI项目，我们可以在常见的PC机上建立SAN存储。

对于如何建立在PC机上的SAN，可以参考：

iSCSI建立
不过，这篇文章提到的iSCSI target管理方式不太方便，通常我们会用targetcli管理target。targetcli可以直接建立和管理不同backstone类型的逻辑卷和不同的export方式，如建立ramdisk并且通过iSCSI export，非常方便。操作方式见targetcli screencast Part 2 of 3: ISCSI – YouTube。

以上都是我们经常接触的单机块存储。接下来是本文主要分享的内容：公共云技术服务提供的块存储服务，开源的块存储框架，以及OpenStack目前对块存储的定义和支持情况。

分布式块存储

在面对极具弹性的存储需求和性能要求下，单机或者独立的SAN越来越不能满足企业的需要。如同数据库系统一样，块存储在scale up的瓶颈下也面临着scale out的需要。我们可以用以下几个特点来描述分布式块存储系统的概念：

分布式块存储可以为任何物理机或者虚拟机提供持久化的块存储设备

分布式块存储系统管理块设备的创建、删除和attach/detach

分布式块存储支持强大的快照功能，快照可以用来恢复或者创建新的块设备

分布式存储系统能够提供不同IO性能要求的块设备

3.1 Amazon EBS

Amazon作为领先的IaaS服务商，其API目前是IaaS的事实标准。Amazon EC2目前在大多数方面远超其他IaaS服务商。Amazon EC2的产品介绍是快速了解Amazon EC2的捷径。

EBS是Amazon提供的块存储服务。通过EBS，用户可以随时增删迁移volume和快照操作。

Amazon EC2实例可以将根设备数据存储在Amazon EBS或者本地实例存储上。使用Amazon EBS时，根设备中的数据将独立于实例的生命周期保留下来，使得在停止实例后仍可以重新启动使用，与笔记本电脑关机并在再次需要时重新启动相似。另一方面，本地实例存储仅在实例的生命周期内保留。这是启动实例的一种经济方式，因为数据没有存储到根设备中。

Amazon EBS提供两种类型的卷，即标准卷和预配置IOPS卷。它们的性能特点和价格不同，可以根据应用程序的要求和预算定制所需的存储性能。

标准卷可为要求有适度或突发式I/O的应用程序提供存储。这些卷平均可以提供大约100 IOPS，最多可突增至数百IOPS。标准卷也非常适合用作引导卷，其突发能力可提供快速的实例启动时间（通常十几秒）。

预配置IOPS卷旨在为数据库等I/O密集型随机读写工作负载提供可预计的高性能。创建一个卷时，利用预置IOPS为卷确定IOPS速率，随之Amazon EBS在该卷的生命周期内提供该速率。Amazon EBS目前支持每预配置IOPS卷最多4000 个IOPS。您可以将多个条带式卷组合在一起，为应用程序提供每个Amazon EC2数千IOPS的能力。

EBS可以在卷连接和使用期间实时拍摄快照。不过，快照只能捕获已写入Amazon EBS 卷的数据，不包含应用程序或操作系统已在本地缓存的数据。如果需要确保能为实例连接的卷获得一致的快照，需要先彻底地断开卷连接，再发出快照命令，然后重新连接卷。

EBS快照目前可以跨regions增量备份，意味着EBS快照时间会大大缩短，从另一面增加了EBS使用的安全性。

总的来说，Amazon EBS是目前IaaS服务商最引入注目的服务之一，目前的OpenStack、CloudStack等其他开源框架都无法提供Amazon EBS的弹性和强大的服务。了解和使用Amazon EBS是学习IaaS块存储的最好手段。

3.2 阿里云存储

阿里云是国内的公共云计算服务商。阿里云磁盘目前仅支持在创建云主机的时候绑定云磁盘或者在升级云主机的进行云磁盘扩容，这从根本上就是传统的虚拟主机的特点而不是所谓的“云磁盘”。

从目前的阿里云磁盘的限制：

无法快速创建或删除volume，在进行扩容时需要升级云主机才能达到，而升级云主机只有在下月云主机套餐到期时才能生效（中国移动套餐的模式）

一个云主机最多只能绑定3个云磁盘

从阿里云磁盘目前的使用分析，以下是我对阿里云磁盘实现的推测：

阿里云主机是跟磁盘绑定的，这意味着阿里云的云磁盘是local volume（因此性能还是挺可观的）。用户需要扩容、减少都需要下个月更说明了这点，整个主机在扩容时去调度合适的有足够存储空间的host，然后进行扩容。

阿里云磁盘是分布式块存储系统，但是由于其QoS无法保证和其他资源调度原因，无法提供足够的块存储支持。

从演讲回顾：阿里云存储技术的演进，以及云服务用例最佳实践中了解到阿里云是基于自家的“盘古”系统，那么从实际使用来说，远没达到一般的分布式块存储系统的要求。

4.1 Ceph

Ceph是开源实现的PB级分布式文件系统，其分布式对象存储机制为上层提供了文件接口、块存储接口和对象存储接口。Inktank是Ceph的主要支持商，也是目前Ceph开源社区的主要力量。

![](http://42.96.185.139/resource/articles/block-storage-overview/zh/resources/1.png)

Ceph目前是OpenStack的开源块存储实现系统（即Cinder项目的backend driver之一），其实现分为三个部分: OSD，Monitor，MDS。OSD是底层对象存储系统，Monitor是集群管理系统，MDS是用来支持POSIX文件接口的Metadata Server。从Ceph的原始论文《Ceph: Reliable, Scalable, and High-Performance Distributed Storage》来看，Ceph专注于扩展性，高可用性和容错性。Ceph放弃了传统的Metadata查表方式（HDFS）而改用算法（CRUSH）去定位具体的block。

利用Ceph提供的RULES可以弹性地制订存储策略和Pool选择，Monitor作为集群管理系统掌握了全部的Cluster Map，Client在没有Map的情况下需要先向Monitor请求得到，然后通过Object id计算相应的OSD Server。

Ceph的文档可以参考以下链接：

Ceph：一个 Linux PB 级分布式文件系统

分布式文件系统Ceph调研1 – RADOS

Ceph Architecture

Ceph的现状

ceph的CRUSH数据分布算法介绍

Ceph INTERNAL DEVELOPER DOCUMENTATION

Ceph支持传统的POSIX文件接口，因此需要额外的MDS（Meatadata Server）支持文件元信息（Ceph的块存储和对象存储支持不需要MDS服务）。Ceph将Data和Metadata分离到两个服务上，跟传统的分布式系统如Lustre相比可以大大增强扩展性。在小文件读写上，Ceph读写文件会有[RTT\*2]，在每次open时，会先去Metadata Server查询一次，然后再去Object Server。除了Open操作外，Ceph在Delete上也有问题，它需要到Metadata Server擦除对应的Metadata，是n(2)复杂度。Ceph在Metadata上并非只有坏处，通过Metadata Server，像目录列表等目录操作为非常快速，远超GlusterFS等其他分布式文件系统的目录或文件元操作。

如果要用Ceph作为块存储项目，有几个问题需要考虑：

Ceph在读写上不太稳定（有btrfs的原因）。目前Ceph官方推荐XFS作为底层文件系统

Ceph的扩展性较难，如果需要介入Ceph，需要较长时间

Ceph的部署不够简易并且集群不够稳定

4.2 Sheepdog

Sheepdog是另一个分布式块存储系统实现。与Ceph相比，它的最大优势就是代码短小好维护，hack的成本很小。Sheepdog也有很多Ceph不支持的特性，比如说Multi-Disk, Cluster-wide Snapshot等。

Sheepdog主要有两部分，一个是集群管理，另一个是存储服务。集群管理目前使用Corosync或者Zookper来完成，其存储服务的特点是在client和存储host有Cache的实现可以大大减小数据流量。

目前Sheepdog只在QEMU端提供Drive，而缺少library支持，这是Sheepdog目前最主要的问题。但是社区已经有相关的Blueprint在讨论这个问题。

了解Sheepdog的一些相关资料：

Sheepdog Overview

Sheepdog 淘宝核心系统团队

Sheepdog wiki：Sheepdog的一系列Wiki如同它的代码一样简短出色

目前Taobao是Sheepdog主要用户和社区贡献者。

5. Cinder

OpenStack是目前流行的IaaS框架，提供了与AWS类似的服务并且兼容其API。OpenStack Nova是计算服务，Swift是对象存储服务，Quantum是网络服务，Glance是镜像服务，Cinder是块存储服务，Keystone是身份认证服务，Horizon是Dashboard，另外还有Heat、Oslo、Ceilometer、Ironic等等项目。

OpenStack的存储主要分为三大类：

对象存储服务，Swift
块设备存储服务，主要是提供给虚拟机作为“硬盘”的存储。这里又分为两块：
本地块存储
分布式块存储，Cinder
数据库服务，目前是一个正在孵化的项目Trove，前身是Rackspace开源出来的RedDwarf，对应AWS里面的RDC。
Cinder是OpenStack中提供类似于EBS块存储服务的API框架。它并没有实现对块设备的管理和实际服务，而是为后端不同的存储结构提供了统一的接口，不同的块设备服务厂商在Cinder中实现其驱动支持以与OpenStack进行整合。后端的存储可以是DAS，NAS，SAN，对象存储或者分布式文件系统。也就是说，Cinder的块存储数据完整性、可用性保障是由后端存储提供的。在CinderSupportMatrix中可以看到众多存储厂商如NetAPP、IBM、SolidFire、EMC和众多开源块存储系统对Cinder的支持。

![](http://42.96.185.139/resource/articles/block-storage-overview/zh/resources/2.png)

从上图我们也可以看到，Cinder只是提供了一层抽象，然后通过其后段支持的driver实现来发出命令来得到回应。关于块存储的分配信息以及选项配置等会被保存到OpenStack统一的DB中。

总结

目前分布式块存储的实现仍然是由Amazon EBS领衔，其卓越稳定的读写性能、强大的增量快照和跨区域块设备迁移，以及令人惊叹的QoS控制都是目前开源或者其他商业实现无法比拟的。

不过Amazon EBS始终不是公司私有存储的一部分，作为企业IT成本的重要部分，块存储正在发生改变。EMC在一个月前发布了其ViPR平台，并开放了其接口，试图接纳其他厂商和开源实现。Nexenta在颠覆传统的的存储专有硬件，在其上软件实现原来只有专有SDN的能力，让企业客户完全摆脱存储与厂商的绑定。Inktank极力融合OpenStack并推动Ceph在OpenStack社区的影响力。这些都说明了，无论是目前的存储厂商还是开源社区都在极力推动整个分布式块存储的发展，存储专有设备的局限性正在进一步弱化了原有企业的存储架构。

在分布式块存储和OpenStack之间，我们可以打造更巩固的纽带，将块存储在企业私有云平台上做更好的集成和运维。

作者简介

王豪迈，UnitedStack工程师。本文根据作者的博客文章《块存储的世界（入门级）》修改而成。


