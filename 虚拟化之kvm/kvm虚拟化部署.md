## 虚拟化之KVM

### KVM介绍
Kernel-based Virtual Machine的简称，是一个开源的系统虚拟化模块，自Linux 2.6.20之后集成在Linux的各个主要发行版本中。它使用Linux自身的调度器进行管理，所以相对于Xen，其核心源码很少。KVM目前已成为学术界的主流VMM之一。  
KVM的虚拟化需要硬件支持（如Intel VT技术或者AMD V技术)。是基于硬件的完全虚拟化。而Xen早期则是基于软件模拟的Para-Virtualization，新版本则是基于硬件支持的完全虚拟化。但Xen本身有自己的进程调度器，存储管理模块等，所以代码较为庞大。广为流传的商业系统虚拟化软件VMware ESX系列是基于软件模拟的Full-Virtualization。 
 
**1、环境准备**  

![](http://i.imgur.com/TnUzog5.png)
![](http://i.imgur.com/cglcKLQ.png)  

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


	


