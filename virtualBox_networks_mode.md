# virtualbox 各种网络模式说明

>该文档于2018年2月3日翻译自VirtualBox官方文档。

## 1，综述
当最开始创建一个虚拟机是，virtualBox（此处简称为VB）给新进虚拟机分配一块虚拟网卡，并设置为NAT（network address translation）模式。图形界面可以设置四种模式，另外还可以使用VBoxManage进行其他网卡模式的设置。下面将详细介绍各种模式的设置。

VB共提供8中模式的virtual PCI Ethernet cards。四种模式可以使用图形界面进行设置，8种模式可以使用命令行模式：VBoxManage modifyvm来进行设置。

## 2，virtual networking hardware
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













