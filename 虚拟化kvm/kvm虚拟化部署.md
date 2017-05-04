## 虚拟化之KVM

### KVM介绍
Kernel-based Virtual Machine的简称，是一个开源的系统虚拟化模块，自Linux 2.6.20之后集成在Linux的各个主要发行版本中。它使用Linux自身的调度器进行管理，所以相对于Xen，其核心源码很少。KVM目前已成为学术界的主流VMM之一。  
KVM的虚拟化需要硬件支持（如Intel VT技术或者AMD V技术)。是基于硬件的完全虚拟化。而Xen早期则是基于软件模拟的Para-Virtualization，新版本则是基于硬件支持的完全虚拟化。但Xen本身有自己的进程调度器，存储管理模块等，所以代码较为庞大。广为流传的商业系统虚拟化软件VMware ESX系列是基于软件模拟的Full-Virtualization。 
 
**1.环境准备**  

![](http://i.imgur.com/TnUzog5.png)
![](http://i.imgur.com/cglcKLQ.png)    

**[tightvnc-2.7.10下载链接](http://down.51cto.com/data/2305235)**  

![](http://i.imgur.com/0BATsEr.png)

**2.部署kvm**  

	[root@linux-node2 zhang]# cat /etc/redhat-release   
	CentOS release 6.5 (Final) 
 
	安装epel包：
	rpm -ivh http://mirrors.ustc.edu.cn/fedora/epel//6/x86_64/epel-release-6-8.noarch.rpm
	
	[root@linux-node2 zhang]#yum install -y qemu-kvm   //创建模块

	[root@linux-node2 zhang]#lsmod |grep kvm
	kvm_intel              54285  0 
	kvm                   333172  1 kvm_intel  
	
	检查系统是否支持虚拟化:
	[root@linux-node2 zhang]# grep -E 'vmx|svm' /proc/cpuinfo
	flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat
    pse36 clflush dts mmx fxsr sse sse2 ss syscall nx rdtscp lm constant_tsc up 
    arch_perfmon pebs bts xtopology tsc_reliable nonstop_tsc aperfmperf  
    unfair_spinlock pni pclmulqdq vmx ssse3 cx16 pcid sse4_1 sse4_2 x2apic popcnt  
    xsave avx f16c hypervisor lahf_lm ida arat epb xsaveopt pln pts dts tpr_shadow 
    vnmi ept vpid fsgsbase smep  
	//如果有输出内容，则支持，其中intel cpu支持会有vmx，amd cpu支持会有svm
  
	//管理工具:
	[root@linux-node2 zhang]#yum install virt-manager python-virtinst qemu-kvm-tools  libvirt -y
     
	//工具解释:
	[root@linux-node2 zhang]#  rpm -qa|grep -E 'qemu|libvirt|virt'
	[root@m01 kvm]# rpm -qa|grep -E 'qemu|libvirt|virt'
	libvirt-python-0.10.2-60.el6.x86_64      //libvirt的图形化虚拟机管理软件，需要图形界面操作系统
	virt-what-1.11-1.2.el6.x86_64            //基于Libvirt的图像化虚拟机管理软件，需要图形界面操作系统
	qemu-img-0.12.1.2-2.491.el6_8.1.x86_64   //用于操作虚拟机硬盘镜像的创建、查看和格式化转化
	gpxe-roms-qemu-0.9.7-6.15.el6.noarch     //虚拟机IPXE的启动固件，支持虚拟机从网络启动
	libvirt-client-0.10.2-60.el6.x86_64      //Libvirt的客户端，最重要的功能之一就是在宿主机关机时可以通过虚拟机也关机，使虚拟机系统正常关机，而不是被强制关机，造成数据丢失
	python-virtinst-0.600.0-29.el6.noarch    //一套Python的虚拟机安装工具
	virt-manager-0.9.0-31.el6.x86_64         //基于Libvirt的图像化虚拟机管理软件，需要图形界面操作系统
	qemu-kvm-0.12.1.2-2.491.el6_8.1.x86_64   //KVM在用户运行的程序
	libvirt-0.10.2-60.el6.x86_64             //用于管理虚拟机，它提供了一套虚拟机操作API
	qemu-kvm-tools-0.12.1.2-2.491.el6_8.1.x86_64

	//启动服务[libvirt]管理kvm:
	[root@linux-node2 zhang]# /etc/init.d/libvirtd  start
	Starting libvirtd daemon:                                  [  OK  ]  

	[root@linux-node2 zhang]# ifconfig    //生成了一个网卡:192.168.122.1 
	
	......
 	
	virbr0    Link encap:Ethernet  HWaddr 52:54:00:3D:A7:77  
          inet addr:192.168.122.1  Bcast:192.168.122.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:29 errors:0 dropped:0 overruns:0 frame:0
          TX packets:20 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:2413 (2.3 KiB)  TX bytes:2231 (2.1 KiB  

	[root@linux-node2 zhang]#ps -ef|grep dns|grep -v grep   //管理dhcp的一些功能。
	nobody    2746     1  0 04:57 ?        00:00:00 /usr/sbin/dnsmasq --strict-order  
   	--pid-file=/var/run/libvirt/network/default.pid --conf-file= --except-interface  
    lo --bind-interfaces --listen-address 192.168.122.1 --dhcp-range    
	192.168.122.2,192.168.122.254 --dhcp-leasefile=/var/lib/libvirt/dnsmasq/    
	default.leases --dhcp-lease-max=253 --dhcp-no-override --dhcp-hostsfile=/var/  
 	lib/libvirt/dnsmasq/default.hostsfile --addn-hosts=/var/lib/libvirt/dnsmasq/   
	default.addnhosts  
    
	//创建虚拟磁盘:
	[root@linux-node2 zhang]# qemu-img create -f raw /opt/kvm.raw 10G  
   	Formatting '/opt/kvm.raw', fmt=raw size=10737418240
	[root@linux-node2 zhang]# ll -h /opt/kvm.raw 
	-rw-r--r-- 1 root root 10G Dec  9 16:04 /opt/kvm.raw 
 
	[root@linux-node2 zhang]# qemu-img  info /opt/CentOS-6.6-x86_64.raw 
	image: /opt/CentOS-6.6-x86_64.raw
	file format: raw
	virtual size: 5.0G (5368709120 bytes)
	disk size: 1.4G

	//准备安装镜像:
	[root@master zhangyiling]#  dd if=/dev/cdrom  of=/opt/CentOS-6.6-x86_64.iso

	//命令帮助:
	[root@linux-node2 zhang]# virsh  --help 

	//创建虚拟机:
	[root@linux-node2 zhang]# virt-install --virt-type kvm --name kvm-demo --ram 512 --cdrom=/opt/   
	centos64.iso --network network=default --graphics vnc,listen=0.0.0.0 --  
	noautoconsole --os-type=linux --os-variant=rhel6 --disk path=/opt/kvm.raw  
	
	[root@linux-node2 zhang]# netstat -lnput|grep 5900
	tcp        0      0 0.0.0.0:5900                0.0.0.0:*                   LISTEN      4032/qemu-kvm 
	
	//查看支持的虚拟机:
	[root@linux-node2 zhang]# /usr/libexec/qemu-kvm -cpu ?
	x86       Opteron_G5  AMD Opteron 63xx class CPU                      
	x86       Opteron_G4  AMD Opteron 62xx class CPU                      
	x86       Opteron_G3  AMD Opteron 23xx (Gen 3 Class Opteron)          
	x86       Opteron_G2  AMD Opteron 22xx (Gen 2 Class Opteron)          
	x86       Opteron_G1  AMD Opteron 240 (Gen 1 Class Opteron)           
	x86        Broadwell  Intel Core Processor (Broadwell) 
 	
	......

**3.安装虚拟机**  

![](http://i.imgur.com/LcLPVOj.png)
	
	[root@linux-node2 zhang]# virsh start CentOS-6.6-x86_64  //启动虚拟机

	[root@linux-node2 zhang]# virsh  list --all
 	Id    Name                           State
	----------------------------------------------------
 	1     CentOS-6.6-x86_64              running //虚拟机状态  
	
	[root@linux-node2 zhang]# vir...   //生产中常用命令
	virsh  //libvirt安装                     virt-host-validate  virt-manager        
	virt-clone          virt-image           virt-pki-validate   
	virt-convert        virt-install        virt-xml-validate

	[root@linux-node2 zhang]#  cat /sys/kernel/mm/transparent_hugepage/enabled
	[always] madvise never

	[root@linux-node2 zhang]#virsh start CentOS-6.6-x86_64  //关闭虚拟机
	Domain CentOS-6.6-x86_64 started 
	
	//常用命令：
	查看启动的进程:netstat -tunlp|grep qemu-kvm 
	强制关闭:virsh undefine kvm-demo 
 	暂停:virsh resume kvm-demo  
	生成kvm虚拟机：virt-install
	查看在运行的虚拟机：virsh list
	查看所有虚拟机：virsh list --all
	查看kvm虚拟机配置文件：virsh dumpxml name
	启动kvm虚拟机：virsh start name
	正常关机：virsh shutdown name 
	非正常关机（相当于物理机直接拔掉电源）：virsh destroy name
	删除：virsh undefine name（彻底删除，找不回来了，如果想找回来，需要备份/etc/libvirt/qemu的xml文件）
	根据配置文件定义虚拟机：virsh define file-name.xml
	挂起，终止：virsh suspend name
	恢复挂起状态:virsh resume name
	 

**4.设置网桥：**  

	[root@linux-node1 zhang]# brctl  show
	bridge name	bridge id		STP enabled	interfaces
	virbr0		8000.5254003da777	yes		virbr0-nic
											vnet0
	//添加网卡：
	[root@linux-node1 zhang]# brctl  addbr br0
	bridge name	bridge id		STP enabled	interfaces
	br0			8000.000000000000	no		
	virbr0		8000.5254003da777	yes		virbr0-nic
											vnet0
	[root@linux-node1 zhang]# brctl addif br0 eth0 && ip addr del dev eth0 192.168.21.139/24 && ifconfig br0 192.168.21.139/24
	
	[root@linux-node1 zhang]# ifconfig 
	br0   Link encap:Ethernet  HWaddr 00:0C:29:7A:62:24  
          inet addr:192.168.21.139  Bcast:192.168.21.255  Mask:255.255.255.0
          inet6 addr: fe80::20c:29ff:fe7a:6224/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:38 errors:0 dropped:0 overruns:0 frame:0
          TX packets:29 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:2606 (2.5 KiB)  TX bytes:5778 (5.6 KiB)

	eth0  Link encap:Ethernet  HWaddr 00:0C:29:7A:62:24  
          inet6 addr: fe80::20c:29ff:fe7a:6224/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:55210 errors:0 dropped:0 overruns:0 frame:0
          TX packets:38004 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:56150912 (53.5 MiB)  TX bytes:6432187 (6.1 MiB)

	[root@linux-node1 zhang]# cp /etc/libvirt/qemu/CentOS-6.6-x86_64.xml /etc/libvirt/qemu/CentOS-6.6-x86_64.xmlba  
	
	[root@linux-node1 zhang]# virsh  edit CentOS-6.6-x86_64
	 <interface type='network'>
           <mac address='52:54:00:9f:ea:96'/>
           <source network='default'/>
           <model type='virtio'/>
           <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
     </interface>
		---->修改成下面:[其实就是后面openstack干的事情]
     <interface type='bridge'>   
           <mac address='52:54:00:9f:ea:96'/>
           <source bridge='br0'/>
           <model type='virtio'/>
	<address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>

	//重启虚拟机：
	[root@linux-node1 zhang]# virsh reboot CentOS-6.6-x86_64

	提示：kvm的网卡默认是开机不启动。

![](http://i.imgur.com/JTGkDob.png)  

![](http://i.imgur.com/RGLwydE.png)
