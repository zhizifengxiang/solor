# virtualbox 各种网络模式说明

>该文档于2018年2月3日翻译自VirtualBox官方文档。

## 0，综述
当最开始创建一个虚拟机是，virtualBox（此处简称为VB）给新进虚拟机分配一块虚拟网卡，并设置为NAT（network address translation）模式。图形界面可以设置四种模式，另外还可以使用VBoxManage进行其他网卡模式的设置。下面将详细介绍各种模式的设置。

VB共提供8中模式的virtual PCI Ethernet cards。四种模式可以使用图形界面进行设置，8种模式可以使用命令行模式：VBoxManage modifyvm来进行设置。

## 1，virtual networking hardware
VB可以虚拟出以下六种类型的网络硬件：
```
AMD PCNet PCI II (Am79C970A);

AMD PCNet FAST III (Am79C973, the default);

Intel PRO/1000 MT Desktop (82540EM);

Intel PRO/1000 T Server (82543GC);

Intel PRO/1000 MT Server (82545EM);

Paravirtualized network adapter (virtio-net).
```
以下为上面六种虚拟网卡的介绍，此处暂且不翻译：

<blockquote>
The PCNet FAST III is the default because it is supported by nearly all operating systems out of the box, as well as the GNU GRUB boot manager. As an exception, the Intel PRO/1000 family adapters are chosen for some guest operating system types that no longer ship with drivers for the PCNet card, such as Windows Vista.

The Intel PRO/1000 MT Desktop type works with Windows Vista and later versions. The T Server variant of the Intel PRO/1000 card is recognized by Windows XP guests without additional driver installation. The MT Server variant facilitates OVF imports from other platforms.

The "Paravirtualized network adapter (virtio-net)" is special. If you select this, then VirtualBox does not virtualize common networking hardware (that is supported by common guest operating systems out of the box). Instead, VirtualBox then expects a special software interface for virtualized environments to be provided by the guest, thus avoiding the complexity of emulating networking hardware and improving network performance. Starting with version 3.1, VirtualBox provides support for the industry-standard "virtio" networking drivers, which are part of the open-source KVM project.

The "virtio" networking drivers are available for the following guest operating systems:

Linux kernels version 2.6.25 or later can be configured to provide virtio support; some distributions also back-ported virtio to older kernels.

For Windows 2000, XP and Vista, virtio drivers can be downloaded and installed from the KVM project web page.[30]

VirtualBox also has limited support for so-called jumbo frames, i.e. networking packets with more than 1500 bytes of data, provided that you use the Intel card virtualization and bridged networking. In other words, jumbo frames are not supported with the AMD networking devices; in those cases, jumbo packets will silently be dropped for both the transmit and the receive direction. Guest operating systems trying to use this feature will observe this as a packet loss, which may lead to unexpected application behavior in the guest. This does not cause problems with guest operating systems in their default configuration, as jumbo frames need to be explicitly enabled.
</blockquote>

## 2，introduction to networking mode
8种模式的网卡有可以拆分成下面的几种基本模式：

> 1, not attached
该模式只是告诉虚拟机有一块网卡，但是没有网络连接——就好像没插网线。此法可以强制对客户机进行重新设置。

>2, networking address translation （NAT）
如你只是向在客户机上浏览网页，下载文件或者查看邮件，此默认模式对于你来说已足够。该模式仅是让客户机可以通过宿主机连接到internet，无法实现其他更复杂的网络配置。宿主机和客户机进行网络传输的限制则参见3.3节。

> 3, NAT network
该模式是在4.3版本中引进。具体参见第4节。

>4, Bridged networking
该模式用于network simulation和在宿主机中运行服务器，其将virtualbox直接与物理网卡相连，并进行数据传输，绕过操作系统网络层。

>5,Internal networking
该模式通过创建各种基于软件的网络，可以让其他指定的虚拟机看见当前虚拟机，宿主机或者外部网络无法看见本机。

>6, Host-only networking
该模式可创建包含宿主机以及其他客户机的网络，不需要宿主机的物理网络接口。通过创建虚拟网络来让宿主机和客户机进行通信（类似于Loopback interface).

> 7, Generic networking
不经常使用的mode共享一个generic network interface，允许用户自由选择VB内部或者扩展包中的驱动，该模式有两种子模式：

>7.1, UDP Tunnel
该模式可以让不同宿主机中的虚拟机进行通信。

>7.2, VDE (Virtual Distributed Ethernet) networking
该选项通常配置于Linux或者FreeBSD系统，可以构建虚拟分布式网络Virtual Distributed Ethernet 。该选项需要手动编译VB源代码，安装包没有此功能。

下表给出了主要网络模式的概览：

Table 6.1. Overview

               |VM ↔ Host |VM1 ↔ VM2   | VM → Internet |   VM ← Internet|
               ---------- | ------- | :------: | :-----: | --------: |
               Host-only  |    +    |     +    |   –     | –         |
               Internal   | –      |     +    | –       | –          |
               | Bridged  | +       |       +    | +   | + |
               | NAT      |   –     |        –    | +    | Port forwarding |
               | NAT Network |    – |       +    | +   | Port forwarding |










