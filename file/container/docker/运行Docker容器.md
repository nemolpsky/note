### 运行Docker容器

Docker官方提供了Docker Desktop用来在Windows上运行Docker容器，使用起来更加方便，本质上其实就是封装了Docker的原生命令来操作。



#### 1. 拉取镜像

最新肯定都是在官方仓库中，比如在官网搜索redis可以看到第一条有一个OFFICIAL IMAGE的标志，这表示是官方镜像，其他的都是私人镜像。

```
https://hub.docker.com/
```

![image3](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/3.png)

直接在cmd窗口下执行拉取命令就可以把镜像下载到本地，如果使用Docker Desktop的话，那么正常情况下环境变量已经默认配置好了，可以直接执行命令。

```
docker pull redis
```

拉取完之后，就可以在Docker Desktop中看到本地列表里面有下载好的镜像。

![image4](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/4.png)

---


#### 2. 运行容器

使用Docker Desktop运行容器就方便多了，只要选中对应的镜像点击run按钮就可以。

![image5](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/5.png)

可以看到会有几个配置，第一行是容器名字，第二行是映射端口，比如nginx的镜像运行到容器中时对外暴露端口是80，如果我们填写是9001，那么这两个端口会自动映射上。第三行是挂载路径可以不填。

![image6](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/6.png)


可以使用下面的命令来查看正在运行的容器，或者查看Docker Desktop里的列表。

```
C:\Users\Half>docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED       STATUS       PORTS                  NAMES
fbc949e76c70   nginx:latest   "/docker-entrypoint.…"   2 hours ago   Up 2 hours   0.0.0.0:9003->80/tcp   nginx2
37ab71ed492b   nginx:latest   "/docker-entrypoint.…"   3 hours ago   Up 3 hours   0.0.0.0:9001->80/tcp   nginx
```

---

#### 3. 与容器交互

其实Docker容器是可以进入到内部执行一些操作的，比如下面的命令就是与容器交互，3是上面容器ID的第一个字符，Docker会自己匹配是否唯一。可以看到与容器交互也是使用linux风格，目录格式也是和linux差不多。

```
C:\Users\Half>docker exec -it 3 bash
root@37ab71ed492b:/#
```

比如我们运行的是Nginx容器，那么就可以在容器内部找到Nginx的配置文件，所以Docker本质上可以理解为超小型的系统。
```
root@37ab71ed492b:/etc/nginx/conf.d# cat default.conf
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    # redirect server error pages to the static page /50x.html
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```


#### 4. 挂载外部数据

容器最大的特点就是启动快，占用资源少，而且容器的推荐做法是如果容器挂了，直接再重启一个容器就可以了，那么就会有个问题产生，容器运行时产生的数据怎么办？所以Docker支持挂载外部目录来放置这些数据，一旦容器挂了，再启动时重新读取外部挂载的数据就可以继续使用。

其实上面运行容器时第三行选项就是来设置挂载目录的，举个例子上面Nginx中的配置文件输出日志是在/var/log/nginx/目录下，首页则是/usr/share/nginx/html下的html文件，那么其实可以把这两个目录挂载到外部。比如下面这样的配置方法，就相当于告诉容器运行的时候读取html文件在F盘的指定目录下，而输出日志也是在F盘的指定目录下，其实就可以理解为把真实使用的目录路径替换掉。



其实就可以理解为执行docker命令时使用-v参数，冒号前面是挂载的外部路径，后面则是容器内部的路径。

```
-v f:\docker-vm-data\nginx\html:/usr/share/nginx/html
```



---

#### 5. 查看容器的状态

使用Docker Desktop可以很方便的查看到运行的容器状态，点击Containers/Apps就可以看到运行的容器，比如这里就启动了两个端口不一样的Nginx。

![image7](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/7.png)

鼠标悬停在一个容器列表上，会发现还有启动，暂停，删除和进行交互的操作，就不用使用命令。


![image8](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/8.png)

点击容器列表，就可以看到运行容器的详细内容，分别是日志，容器配置和使用情况。

![image9](https://github.com/nemolpsky/Note/raw/master/file/container/docker/images/9.png)

而且还支持推送镜像到自己的Docker Hub中，这样如果把常用的软件环境配置好后推送到仓库中，使用的时候再拉取出来，会很方便。