### Docker Desktop安装配置

Docker Desktop是Docker官方提供的一个在Windows下运行的软件，可以使用可视化界面配置和运行Docker环境，本质上就是利用Win10自带的Hyper-V来创建个虚拟机运行Docker环境。可以直接在官网下载安装。

>https://www.docker.com/products/docker-desktop

下载安装完后需要配置下仓库镜像，可以选阿里的或者网易的，否则官方镜像速度会比较慢。还可以设置虚拟机的内存，磁盘大小等数据。

![image1](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/2.png)

```
{
  "registry-mirrors": [
    "https://hub-mirror.c.163.com/"
  ],
  "insecure-registries": [],
  "debug": false,
  "experimental": true
}
```

Docker Desktop还可以登录在Docker Hub上的账号，这样你就可以看到远程仓库里的镜像或者是本地的镜像。

![image2](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/2.png)