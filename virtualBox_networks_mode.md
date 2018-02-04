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

Table 1. Overview

                   mode       |VM ↔ Host |VM1 ↔ VM2 | VM → Internet | VM ← Internet
               ----------     | -------  | ------   | -----         | --------
               Host-only      |    +     |     +    |   –           | –
               Internal       | –        |     +    | –             | –
               Bridged        | +        |       +  | +             | +
               NAT            |   –      |        – | +             | Port forwarding
               NAT Network    |    –     |       +  | +             | Port forwarding


以下将详细介绍各个模式的具体内容。

## 3，network address transla（NAT）
由于其无需在宿主机和客户机上做额外设置就能访问互联网，因此其作为默认设置。使用NAT的虚拟就就好像通过“路由”在访问互联网，此处“路由”为virtualbox network engine，每个客户机与宿主机之间都有这种“路由”，这样每台虚拟机无法彼此通信。

不好的地方是路由外部的机器无法访问虚拟机，除非设置port forwarding，否则不能再虚拟机上运行服务器。
由客户机发出的数据帧(network frame)经由virtualbox NAT engine接受，并解析出TCP/IP数据，并再由宿主机系统发送给苏书记上的应用护着网络上的其他主机。就好像数据是由宿主机上的virtualbox软件发出，IP地址为宿主机地址。virtualbox监听网络上的回信，并重新打包发送给客户机。

virtualbox中的DHCP服务器自动分配给客户机网络地址和相应配置信息，因此，客户机的IP地址与宿主机的IP地址不同。若有更多机器分配到NAT中，则第一块网卡所在的的网络地址为：10.0.2.0,第二块网卡网络地址为：10.0.3.0。若想改变客户机的网络地址范围，参见9.11节： “Fine-tuning the VirtualBox NAT engine”.

### 3.1. Configuring port forwarding with NAT

由于每一台虚拟机独享一个虚拟网络，因此宿主机和其他虚拟机无法访问当前虚拟机上的服务。virtualbox可以向路由一样，通过port forwarding来提供一些服务，即virtualbox监听某些端口，并将端口收到的packet转发给虚拟机。

宿主机或其他物理、虚拟机看起来像是使用了一个代理，由此，我们不能在宿主机或其他虚拟机上，与当前虚拟机在相同端口运行服务，好处是将服务进行隔离。

Port Forwarding editor 中可以通过图形界面来进行设置，将宿主机的ports映射到客户机的ports行，这样网络数据就能流向客户机了。另一种配置方式是使用命令行工具VBoxMange，具体参见第8.8节, “VBoxManage modifyvm”.

可以使用任何未被使用端口，比如下面命令让客户机提供ssh服务：
>VBoxManage modifyvm "VM name" --natpf1 "guestssh,tcp,,2222,,22"

上面例子指出，凡是到达宿主机2222端口的TCP请求，都被重定向到客户机22端口处。上面的tcp指明了端口转发使用的主要协议（UDP协议也可以使用）。gustssh仅仅是描述性字段，不设置也会自动产生。--natpf之后的额数字表明网卡号，
若移除forwarding，则使用下面命令：

>VBoxManage modifyvm "VM name" --natpf1 delete "guestssh"


If for some reason the guest uses a static assigned IP address not leased from the built-in DHCP server, it is required to specify the guest IP when registering the forwarding rule:

VBoxManage modifyvm "VM name" --natpf1 "guestssh,tcp,,2222,10.0.2.19,22"

This example is identical to the previous one, except that the NAT engine is being told that the guest can be found at the 10.0.2.19 address.

To forward all incoming traffic from a specific host interface to the guest, specify the IP of that host interface like this:

VBoxManage modifyvm "VM name" --natpf1 "guestssh,tcp,127.0.0.1,2222,,22"

This forwards all TCP traffic arriving on the localhost interface (127.0.0.1) via port 2222 to port 22 in the guest.

It is possible to configure incoming NAT connections while the VM is running, see Section 8.13, “VBoxManage controlvm”.

### 3.2. PXE booting with NAT

PXE booting is now supported in NAT mode. The NAT DHCP server provides a boot file name of the form vmname.pxe if the directory TFTP exists in the directory where the user's VirtualBox.xml file is kept. It is the responsibility of the user to provide vmname.pxe.

### 3.3. NAT limitations

There are four limitations of NAT mode which users should be aware of:

ICMP protocol limitations:

Some frequently used network debugging tools (e.g. ping or tracerouting) rely on the ICMP protocol for sending/receiving messages. While ICMP support has been improved with VirtualBox 2.1 (ping should now work), some other tools may not work reliably.
Receiving of UDP broadcasts is not reliable:

The guest does not reliably receive broadcasts, since, in order to save resources, it only listens for a certain amount of time after the guest has sent UDP data on a particular port. As a consequence, NetBios name resolution based on broadcasts does not always work (but WINS always works). As a workaround, you can use the numeric IP of the desired server in the \\server\share notation.
Protocols such as GRE are unsupported:

Protocols other than TCP and UDP are not supported. This means some VPN products (e.g. PPTP from Microsoft) cannot be used. There are other VPN products which use simply TCP and UDP.
Forwarding host ports < 1024 impossible:

On Unix-based hosts (e.g. Linux, Solaris, Mac OS X) it is not possible to bind to ports below 1024 from applications that are not run by root. As a result, if you try to configure such a port forwarding, the VM will refuse to start.

These limitations normally don't affect standard network use. But the presence of NAT has also subtle effects that may interfere with protocols that are normally working. One example is NFS, where the server is often configured to refuse connections from non-privileged ports (i.e. ports not below 1024).


## 4. Network Address Translation Service

The Network Address Translation (NAT) service works in a similar way to a home router, grouping the systems using it into a network and preventing systems outside of this network from directly accessing systems inside it, but letting systems inside communicate with each other and with systems outside using TCP and UDP over IPv4 and IPv6.

A NAT service is attached to an internal network. Virtual machines which are to make use of it should be attached to that internal network. The name of internal network is chosen when the NAT service is created and the internal network will be created if it does not already exist. An example command to create a NAT network is:

VBoxManage natnetwork add --netname natnet1 --network "192.168.15.0/24" --enable

Here, "natnet1" is the name of the internal network to be used and "192.168.15.0/24" is the network address and mask of the NAT service interface. By default in this static configuration the gateway will be assigned the address 192.168.15.1 (the address following the interface address), though this is subject to change. To attach a DHCP server to the internal network, we modify the example as follows:

VBoxManage natnetwork add --netname natnet1 --network "192.168.15.0/24" --enable --dhcp on

or to add a DHCP server to the network after creation:

VBoxManage natnetwork modify --netname natnet1 --dhcp on

To disable it again, use:

VBoxManage natnetwork modify --netname natnet1 --dhcp off

DHCP server provides list of registered nameservers, but doesn't map servers from 127/8 network.

To start the NAT service, use the following command:

VBoxManage natnetwork start --netname natnet1

If the network has a DHCP server attached then it will start together with the NAT network service.

VBoxManage natnetwork stop --netname natnet1

stops the NAT network service, together with DHCP server if any.

To delete the NAT network service use:

VBoxManage natnetwork remove --netname natnet1

This command does not remove the DHCP server if one is enabled on the internal network.

Port-forwarding is supported (using the --port-forward-4 switch for IPv4 and --port-forward-6 for IPv6):

VBoxManage natnetwork modify --netname natnet1 --port-forward-4 "ssh:tcp:[]:1022:[192.168.15.5]:22"

This adds a port-forwarding rule from the host's TCP 1022 port to the port 22 on the guest with IP address 192.168.15.5. Host port, guest port and guest IP are mandatory. To delete the rule, use:

VBoxManage natnetwork modify --netname natnet1 --port-forward-4 delete ssh

It's possible to bind NAT service to specified interface:

VBoxManage setextradata global "NAT/win-nat-test-0/SourceIp4" 192.168.1.185

To see the list of registered NAT networks, use:





