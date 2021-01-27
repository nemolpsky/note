### netshoot镜像使用

Docker镜像使用起来很方便，如果用的熟练各种环境搭建以及依赖部署都可以几个命令搞定，不过还是有一点学习成本的，如果Linux基础不太好，命令行也用得少会比较坑，尤其是Docker容器之间的通信都是基于网络通信，即使是部署在同一台机器上也是如此，所以很多时候会因为网络配置导致环境容器之间依赖出问题，虽然Docker容器内部有提供命令行进行交互，但是里面基本上什么工具都没有装，netshoot就是专门用于测试Docker容器的一个工具镜像。

---


#### 1. 拉取镜像

先拉取镜像到本地，如果使用了镜像加速器应该会比较快，整个镜像大概350M左右。
```
docker pull nicolaka/netshoot
```


---

#### 2. 基本使用

Docker中是有网络命名空间的概念，host是宿主机网络的命名空间，也可以自己创建一个，如果你想测试哪个网络空间下的网络，按照下面命令启动就可以，

```
docker run -it --net host nicolaka/netshoot
```

然后就可以看到下面的界面，netshoot本质上就是一个装了各种各样网络工具的镜像，这个时候你就可以通过各种工具的命令来测试网络的连通性了。

```
PS C:\Users\Half> docker run -it --net bridge nicolaka/netshoot
                    dP            dP                           dP
                    88            88                           88
88d888b. .d8888b. d8888P .d8888b. 88d888b. .d8888b. .d8888b. d8888P
88'  `88 88ooood8   88   Y8ooooo. 88'  `88 88'  `88 88'  `88   88
88    88 88.  ...   88         88 88    88 88.  .88 88.  .88   88
dP    dP `88888P'   dP   `88888P' dP    dP `88888P' `88888P'   dP

Welcome to Netshoot! (github.com/nicolaka/netshoot)
root @ /
 [1] 🐳  → ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:13 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1102 (1.0 KiB)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

ping一下域名等操作，具体支持哪些工具，可以查看Docker Hub上的文档，https://hub.docker.com/r/nicolaka/netshoot 。
```
 [6] 🐳  → ping www.baidu.com
PING www.a.shifen.com (182.61.200.7) 56(84) bytes of data.
64 bytes from 182.61.200.7 (182.61.200.7): icmp_seq=1 ttl=37 time=58.7 ms
64 bytes from 182.61.200.7 (182.61.200.7): icmp_seq=2 ttl=37 time=55.9 ms
64 bytes from 182.61.200.7 (182.61.200.7): icmp_seq=3 ttl=37 time=58.1 ms
64 bytes from 182.61.200.7 (182.61.200.7): icmp_seq=4 ttl=37 time=63.0 ms
64 bytes from 182.61.200.7 (182.61.200.7): icmp_seq=5 ttl=37 time=60.9 ms
```