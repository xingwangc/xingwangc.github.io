---
layout: post
title: "[原]kubernetes1.1版本上遇到的坑"
date: 2018-01-03
categories: kubernetes
---

最近又开始倒腾kubernetes了，翻出了大概两年前kubernetes刚刚release 1.0时我们在1.0，1.1版本上踩过的坑的笔记。虽然现在都已经release 1.9.0了，当年踩过的坑，现在部署也都已经遇不到了，但是发现基础的东西还在，解决问题的方式还是需要考虑那些方方面面。所以在此把一些仍然可以在遇到问题时参考的在此分享出来，也算是对过去的付出有个交代。

## 1. flannel网络不通的问题

* 问题：flannel网络不通
	
	公司在阿里云上部署了一个测试环境，部署完成后发现各项功能都正常，但是不同机器之间的容器网络不通。最后定位到时flannel不通，准确的说是flannel没有把容器的包转发到指定的网卡上。
	
* 分析：route -n 发现机器上多创建了一条172.16.0.0的错误路由与flannel网络的172.16.0.0的路由冲突
* 解决：执行“route del -net 172.16.0.0&nbsp;net mask 255.240.0.0 dev eth0”删除错误的路由。

> kubernetes集群的网络比较复杂，包括pod内容器之间的网络，同一台机器pod和pod之间的网络，跨主机容器之间的网络，还包括service虚拟地址同pod之间的proxy等。遇到网络不通的时候，最好首先检查各种网络是否正常，如docker和flannel的IP是否在同一个网段，路由是否正常，iptables规则是否正常等。有时候iptables规则的顺序不对也可能会造成意想不到的网络问题。

## 2. iptables proxy问题

* 问题：使用iptables proxy模式，通过service ip访问服务有问题。

	问题的具体表现形式为：集群多台机器间，如果有A，B两个不同服务分别部署在a，b两台机器上，则在a机器上通过sevice IP访问A服务正常，通过service IP访问B服务不正常；类似的在b机器上通过service IP 访问B服务正常，访问A服务不正常。但是在a，b机器上分别通过容器IP访问对方服务都正常。
	
* 分析： 

	* 1）直接通过pod IP访问没有问题，说明网络通畅
	* 2）查询iptables规则，发现规则如下，对Service IP的访问将匹配自定义chain，自定义chain 50%概率随机选择两个pod的一个做DNAT转发。从IP规则来看并没有问题，本机访问正常也是使用相同iptables规则。

		```
		-A KUBE-SEP-PU5BJ5CJYE3SLHYK -s 172.16.8.4/32 -m comment --comment "default/svc-nginx:" -j MARK --set-xmark 0x4d415351/0xffffffff
     -A KUBE-SEP-PU5BJ5CJYE3SLHYK -p tcp -m comment --comment "default/svc-nginx:" -m tcp -j DNAT --to-destination 172.16.8.4:80
     -A KUBE-SEP-PUI35VG6I46PKTF6 -s 172.16.8.2/32 -m comment --comment "default/svc-nginx:" -j MARK --set-xmark 0x4d415351/0xffffffff
     -A KUBE-SEP-PUI35VG6I46PKTF6 -p tcp -m comment --comment "default/svc-nginx:" -m tcp -j DNAT --to-destination 172.16.8.2:80
     -A KUBE-SERVICES -d 10.254.180.23/32 -p tcp -m comment --comment "default/svc-nginx: cluster IP" -m tcp --dport 80 -j KUBE-SVC-KMMA4UO25W53IH36
     -A KUBE-SVC-KMMA4UO25W53IH36 -m comment --comment "default/svc-nginx:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-PUI35VG6I46PKTF6
     -A KUBE-SVC-KMMA4UO25W53IH36 -m comment --comment "default/svc-nginx:" -j KUBE-SEP-PU5BJ5CJYE3SLHYK
     ```
     
	* 3) 使用route查看路由表，没有问题

		```
		 [root@k-node1 ~]# route
          Kernel IP routing table
          Destination     Gateway         Genmask         Flags      Metric      Ref    Use      Iface
          default              gateway         0.0.0.0               UG         0              0      0      enp0s3
          link-local          0.0.0.0          255.255.0.0           U          1002        0      0      enp0s3
          172.16.0.0        0.0.0.0         255.240.0.0            U          0              0      0      flannel.1
          172.16.74.0      0.0.0.0         255.255.255.0        U          0              0      0      docker0
          192.168.2.0      0.0.0.0         255.255.255.0        U          0              0      0      enp0s3
		```
		
	* 4) 使用tcpdump抓包分析数据包走向

	1.首先尝试在a上访问本机服务A，同时使用tcpdump监控本机flannel网卡，docker0网卡。访问本机服务时，虽然经过iptables代理，但是数据包并没有走flannel网卡，而是直接到了docker网卡。符合kubernetes的网络架构。
	
	2.继续尝试在a上直接通过pod IP访问b上的服务B，同时使用tcpdump分别监控a机器的flannel虚拟网卡，b机器的flannel网卡，以及docker bridge的入口网卡。各个网卡上监听到的数据请求如下（删除options部分）：
	
	```
	a机器：flannel.1：13:53:55.611789 IP k-node1.36479 > 172.16.8.4.http: Flags [S], seq 1460874007, win 28200
   b机器：flannel.1：13:53:55.611621 IP 172.16.74.0.36479 > 172.16.8.4.http: Flags [S], seq 1460874007, win 28200
   b机器：docker0：13:53:55.611621 IP 172.16.74.0.36479 > 172.16.8.4.http: Flags [S], seq 1460874007, win 28200
	```
	
	 	从上面的截取可以看到，首先在a上的请求经过iptables后，到达a机器的flannel网卡时，源地址为host名（映射到192.168.2.0/24的地址），端口为随机值。然后请求到达另一端b机器的flannel网卡时，源地址已经变成a机器的flannel网卡地址（172.16.74.0），端口不变。然后数据包继续转发到docker0网卡，最后到达容器。
	 	
	3.在a机器上使用service IP访问b机器上服务的B，执行相同的抓包操作：
	
	```
	a机器：flannel.1：13:56:14.916152 IP k-node1.35581 > 172.16.8.4.http: Flags [S], seq 3885034081, win 29200
   b机器：flannel.1：13:56:14.915632 IP k-node1.35581 > 172.16.8.4.http: Flags [S], seq 3885034081, win 29200
   b机器：docker0：nothing
	```

	从上面的截取可以看到，数据请求到达a机器flannel网卡时与前面相同，源地址都为host名；但是到了对端b机器的flannel时仍然为host名（对应192.168.2.0/24的IP）。然后docker0网卡上没有任何数据包。所以flannel可能发现源IP地址不在它的路由范围内，所以并未对包进行转发。
	
* 解决：

	前面已经找到了原因，继续通过抓包做进一步分析。假设我们在b机器上部署的服务是nginx。
	
	1. 查询所有关于nginx的iptables规则。看到nginx所对应的两个pod的ip/network，都有一条规则将它们mark为0x4d415351，事实上这个是kubernetes集群为service打的一个唯一标签，所有同一集群内的service pod ip都有一个对应的规则被标记为这个值。如下图所示：

			-A KUBE-SEP-PU5BJ5CJYE3SLHYK -s 172.16.8.4/32 -m comment --comment "default/svc-nginx:" -j MARK --set-xmark 0x4d415351/0xffffffff
			
		![nginx-iptables]({{ site.url }}/assets/images/nginx-iptables)

	2. 继续查询iptables，显示所有的iptables MASQUERADE规则，可以看到，有一条规则指示所有mark为0x4d415351的规则都将执行MASQUERADE。

			-A POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark
			
		![nginx-masquerade]({{ site.url }}/assets/images/nginx-masquerade)
		
	3. 所以经过上面的分析，增加一条规则对所有源地址为192.168.2.0/24，目的地是172.16.0.0/16的包进行MASQUERADE。

			iptables -t nat -A POSTROUTING -s 192.168.2.0/24 -d 172.16.0.0/16 -j MASQUERADE
			
	4. 增加后的规则如下，再进行访问就可以了

		![nginx-masquerade2]({{ site.url }}/assets/images/nginx-masquerade)
		
> 遇到这个问题时，kubeproxy的默认模式还是不是iptables proxy模式，iptables proxy模式当时还是一个相当于完成开发的新功能，所以这个问题可能是当时iptables proxy模式在创建iptables规则时的bug造成的。
		
## 集群压测过程中资源相关的问题

1. 内存耗尽

	1. 所有Service都是Best-Effort的类型；有一个service是java开发的，比较耗内存，在集群长时间运行后，发现该service经常重启，查看容器log，没有任何错误日志，只显示被killed。
	2. 怀疑是系统内存不足，造成service进程被集群kill，然后造成容器也跟着重启。
	3. 设定该service的memory requests和limits，将其类型变成guaranteed类型，优先保证该service内存，结果造成该service所在节点内存超额使用的情况下，其他service相继被kill，然后集群kubelet也被停止以释放resource，内存还不够时该service继续在机器节点中漂移，引起连锁反应，大量service向其他节点漂移，节点一个一个相继因为kubelet被kill状态变成not ready。

2. CPU耗尽

	压测过程中，有service占用CPU资源随负载增大而增大，当该service占用一个节点大量CPU资源后，同样造成节点中其他service漂移，节点中还有超额使用情况时，同样也会造成User service漂移，节点挂掉。
	
> 与内存超额使用，系统会以OOM kill service不同，CPU超额使用时，service不会被kill，只会造成service得不到CPU而响应变慢。