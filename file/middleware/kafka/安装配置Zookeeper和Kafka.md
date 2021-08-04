### 安装配置Zookeeper和Kafka

1. 安装配置JDK

    ```
    // 解压
    tar  zxvf  jdk-8u181 - linux-x64.tar.gz

    // 配置环境变量
    export JAVA_HOME=/opt/idk1.8.0 181
    export JRE_HOME=$JAVA HOME/jre

    // 验证版本
    java  - version 
    ```

2. 安装配置Zookeeper

    ```
    // 解压
    tar zxvf zookeeper-3.4.12.tar.gz

    // 配置环境变量可以直接调用bin目录下的shell脚本
    export  ZOOKEEPER _HOME=/opt/zookeeper-3.4 . 12
    export  PATH=$PATH:$ZOOKEEPER _HOME/bin
    ```

    设置配置文件的基本参数，注意，需要在数据目录下创建一个myid文件，里面填写一个数字作为节点编号，不重复即可，如果是单节点就无需配置。

    ```
    # 心跳时间，2000毫秒
    tickTime=2000
    # 投票选举新leader的初始化时间
    initLimit=10
    # leader与follower心跳检测最大容忍时间,响应超过syncLimit*tickTime, leader认为follower "死掉” ,从服务器列表中删除follower
    syncLimit=5
    # 数据目录
    dataDir=/tmp/zookeeper/data
    # 日志目录
    dataLogDir=/tmp/zookeeper/log
    # 客户端对外暴露的端口
    clientPort=2181
    ```

    如果想要集群，需要在不同的机器上配置下列配置，server.后面的数字就是节点编号，后面的值格式是IP地址：节点之间同步数据的端口：节点之前选举的端口。

    ```
    server.0=192.168.0.2:2888:3888
    server.1=192.168.0.3:2888:3888
    server.2=192.168.0.4:2888:3888
    ```

    Zookeeper的bin目录下自带启动脚本和客户端脚本，支持很多参数命令，比如前台启动可以打印日志。

    ```
    ./zkServer.sh  start    // 开启
    ./zkServer.sh  status   // 查看状态
    ./zkClient.sh   // 通过客户端连接，可以使用ls / 和get等命令查看节点下数据
    ```

3. 安装配置Kafka

    ```
    // 解压
    tar -zxvf kafka_2.11-2.0.0.tgz

    // 可一样配置环境变量
    ```

    设置配置文件，conf目录下的server.properties文件

    ```
    # 节点编号，需要保证唯一性
    broker.id=0
    # 对外提供服务的ip地址和端口，比如Java调用时就要通过这个链接，如果不配置这个或者配置错了连接的时候就会提示找不到Kafka的节点
    # listeners=PLAINTEXT://112.17.0.1:9092，在腾讯云上面要注意机器有自己的内网IP，不能输入127.0.0.1，要输入内网IP，ifconfig查看内网IP
    listeners=PLAINTEXT://localhost:9092
    # 日志存放目录，也是实际存放消息日志的地方
    log.dirs=/opt/tmp/kafka-logs
    # zookeeper地址，下面这个方式，会直接在zookeeper根目录创建节点，可以创建指定根目录，例如localhost:2181/kafka
    zookeeper.connect=localhost:2181
    ```

    同样的，Kafka的bin目录下也有各种辅助脚本。

    ```
    // 前台启动
    ./kafka-server-start.sh config/server.properties
    // 后台启动
    ./kafka-server-start.sh -daemon config/server.properties
    ./kafka-server-start.sh config/server.properties &
    ```
    
4. 安装kafka-manager
    
    使用docker安装，拉取稳定版本的镜像。启动的时候设置zookeeper的地址，主要要和上面的内网IP保持一致。
    
    ```
    docker pull hlebalbau/kafka-manager:stable
    docker run --name kafka-manager -d -p 49000:9000 -e ZK_HOSTS="http://112.17.0.1:2181" hlebalbau/kafka-manager:stable
    ```
