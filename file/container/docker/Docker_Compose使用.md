### Docker Compose使用
Docker Compose是Docker官方提供的编排工具，作用在于一次性启动多个容器，其实就是配置一个yaml文件，然后基于这个配置文件运行容器。

---

#### 1. 安装
如果是使用的Docker Desktop那么就已经自带了Docker Compose，不然的话就要去官网下载，https://docs.docker.com/compose/install/。

```
PS F:\Docker-vm-data\compose> docker-compose -v
docker-compose version 1.27.4, build 40524192
```

---

#### 2. 编排启动多个容器

本质上就是通过yaml文件来定义好所有容器的运行配置，然后使用一个命令启动所有容器，因为Docker建议是一个容器一个应用，但是多应用之间的依赖又无法避免，所以提供了这个方法来解决运维问题。

下面就是一个yaml文件，名字是固定的docker-compose.yml，这个配置文件是直接启动nginx和redis，并且还把两个容器的数据都挂载到外部磁盘。这里要注意，yaml文件所在的目录就会被当做根目录，所以./就是代表根目录。

当然这里这是一个最简单的例子，其实还可以支持很多复杂的命令，比如构建你自己的项目然后将所依赖的环境都整合到一个yaml文件中 。具体的可以查看官网，https://docs.docker.com/get-started/08_using_compose/。

```
version: "3.7"
services:
# redis配置
  redis:
    # 镜像标签
    image: redis:6.0.10
    # 端口映射
    ports:
      - 6379:6379
    # 映射目录，./redis/data/是yml文件所在目录，/data是根目录
    volumes:
      - ./redis/data/:/data
# nginx配置
  nginx:
    # 镜像标签
    image: nginx:1.19.6
    # 端口映射
    ports:
      - 80:80
    # 映射目录
    volumes:
      - ./nginx/html/:/usr/share/nginx/html
      - ./nginx/log/:/var/log/nginx

```


---

#### 3. 运行

```
# 启动
PS F:\Docker-vm-data\compose> docker-compose up -d
Creating network "compose_default" with the default driver
Creating compose_redis_1 ... done
Creating compose_nginx_1 ... done

# 运行
PS F:\Docker-vm-data\compose> docker-compose down
Stopping compose_nginx_1 ... done
Stopping compose_redis_1 ... done
Removing compose_nginx_1 ... done
Removing compose_redis_1 ... done
Removing network compose_default
PS F:\Docker-vm-data\compose>
```

---

#### 4. 更多实例

这个仓库下，有很多编排的实例，https://github.com/docker/awesome-compose。