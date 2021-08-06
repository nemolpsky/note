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
    ```

    设置配置文件，conf目录下的server.properties文件

    ```
    # 节点编号，需要保证唯一性
    broker.id=0
    # 对外提供服务的ip地址和端口，比如Java调用时就要通过这个链接，如果不配置这个或者配置错了连接的时候就会提示找不到Kafka的节点
    # listeners=PLAINTEXT://:9092，本地配置127.0.0.1就可以
    listeners=PLAINTEXT://localhost:9092

    # 像在腾讯云这样的包含内网IP和外网IP的需要在host.name上配置内网IP，在advertised.listeners中配置外网IP
    host.name=10.0.4.7
    advertised.listeners=PLAINTEXT://142.246.329.137:9092

    # 日志存放目录，也是实际存放消息日志的地方
    log.dirs=/opt/tmp/kafka-logs
    # zookeeper地址，下面这个方式，会直接在zookeeper根目录创建节点，可以创建指定根目录，例如localhost:2181/kafka
    zookeeper.connect=localhost:2181
    # 这个必须加，否则主题删除不了
    delete.topic.enable=true
    # 这个也最好加，禁止创建默认主题，比如监听了topic1，但是没有创建，会自动创建一个分区和副本都为1的主题，后续修改很麻烦。但是注意，它不会禁止spring中的TopicBuilder去创建主题。
    auto.create.topics.enable=false
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
    
    拉取kafka主题信息的时候，也要输入内网IP，不能输入127.0.0.1，在云服务器上127.0.0.1指向的是公网IP。

    ```
    docker pull hlebalbau/kafka-manager:stable
    docker run --name kafka-manager -d -p 49000:9000 -e ZK_HOSTS="http://112.17.0.1:2181" hlebalbau/kafka-manager:stable
    ```

