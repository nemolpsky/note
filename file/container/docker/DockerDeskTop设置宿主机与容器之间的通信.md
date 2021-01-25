### Docker DeskTop设置宿主机与容器之间的通信

Docker容器可以直接运行在Linux系统上，容器与宿主机之间的网络不是同一个，但它们之间是联通的，所以使用Nginx配置反向代理的时候如果是要代理到宿主机上的服务就不能使用localhost，而是需要使用宿主机上的IP，或者容器运行时指定使用和宿主机联通的网络。


---


#### 1. 容器使用的网络

容器启动的时候可以使用```--network [NAME]```命令指定网络，如果不使用则默认会使用bridge网络。注意，容器必须处在同一个网络下才能进行网络通信。比如下面的命令就可以看到目前Docker容器间有几个网络。host是宿主机的IP，bridge是默认使用的网络IP，none是无网络，其它的都是手动创建的，注意host可以和所有网络联通，但是其他网络之间则不能联通，比如bridge和esnet就无法联通。
```
PS C:\Users\Half> docker network list
NETWORK ID     NAME      DRIVER    SCOPE
1fa659264fc4   bridge    bridge    local
e7b5fc15d179   esnet     bridge    local
724cba8f7120   host      host      local
0667b8f89fe7   net1      bridge    local
59031ced76b9   none      null      local
```

还可以使用下面的命令来查看一个已经运行的容器的网络。
```
docker inspect nginx
```

默认是使用bridge，它的IP和host的IP肯定不一样，所以直接在Nginx中配置host的IP就可以正常进行反向代理了。
```
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "1fa659264fc465169fa940fc5dac0a2946115e8e8b77b12eeade4bdb9a44c5e4",
                    "EndpointID": "18c953c99eeef10f22da21c12603b095ebbbbdfa6d4762e5912de1b0c2fa76d6",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:02",
                    "DriverOpts": null
                }
            }
```

使用ifconfig命令，可以看到docker0是Docker引擎创建的一个虚拟网卡，也就是host网络的IP，eth0是真实物理网卡的IP，直接配置它也可以。
```
docker0   Link encap:Ethernet  HWaddr 02:42:6F:02:01:E8
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          inet6 addr: fe80::42:6fff:fe02:1e8/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:367 errors:0 dropped:0 overruns:0 frame:0
          collisions:0 txqueuelen:0
          RX bytes:630695 (615.9 KiB)  TX bytes:71952 (70.2 KiB)

eth0      Link encap:Ethernet  HWaddr 02:50:00:00:00:01
          inet addr:192.168.65.3  Bcast:192.168.65.15  Mask:255.255.255.0
          inet6 addr: fe80::50:ff:fe00:1/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:831 errors:0 dropped:0 overruns:0 frame:0
          TX packets:861 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:71237 (69.5 KiB)  TX bytes:79110 (77.2 KiB)
```

---

#### 2. Docker DeskTop配置宿主机IP

Docker DeskTop是官方提供的一个在Windows下使用的Docker工具，提供了Docker的运行环境，还提供了个可视化界面，但是实际上是利用了Win 10系统的Hyper-V虚拟机，所以如果想要容器访问Windows系统上的本地服务，除了上面的配置宿主机的IP之外，还要保证虚拟机和Win 10之间的网络是通的。这里对于Docker容器来说，虚拟机就是宿主机，但是虚拟机也有一个宿主机就是Win 10系统。

打开虚拟机设置，新增设置网络。

![image10](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/10.png)
![image11](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/11.jpg)
![image12](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/12.jpg)
![image13](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/13.jpg)

配置虚拟机网络共享Win 10系统网络。

![image14](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/14.jpg)
![image15](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/15.jpg)

给Docker虚拟机添加网络配置。

![image16](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/16.jpg)
![image17](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/17.jpg)

把虚拟网络获取IP和DNS都改成自动获取。

![image18](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/18.jpg)

不过Docker DeskTop在安装的时候已经安装了一个虚拟网络，所以可以直接使用这个配置，不需要新建。

![image19](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/19.png)

按上述配置就可以保证类似Nginx之类的容器，可以访问到Win 10本地的服务了。