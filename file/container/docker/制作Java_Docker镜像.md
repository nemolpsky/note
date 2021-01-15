### 制作Java Docker镜像


#### 1. 创建Maven项目

这是一个最简单的Maven项目，只有一个主类输出一句话，使用maven命令打包出jar包。

```
public class MainClass {

    public static void main(String[] args) {
        System.out.println("hello world！");
    }

}
```

```
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>org.example</groupId>
  <artifactId>JVMDemo</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>jar</packaging>

  <name>JVMDemo</name>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencyManagement>
    <dependencies>
    </dependencies>
  </dependencyManagement>

  <build>
    <pluginManagement><!-- lock down plugins versions to avoid using Maven defaults (may be moved to parent pom) -->
      <plugins>
        <plugin>
          <!-- Build an executable JAR -->
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-jar-plugin</artifactId>
          <version>3.1.0</version>
          <configuration>
            <archive>
              <manifest>
                <addClasspath>true</addClasspath>
                <classpathPrefix>lib/</classpathPrefix>
                <mainClass>MainClass</mainClass>
              </manifest>
            </archive>
          </configuration>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>
</project>
```

---

#### 2. 构建Docker镜像

Docker镜像最基本的要求就是需要有个Dockerfile文件，在这里面编写对应的依赖镜像、环境和启动参数等信息。

第一行是说要依赖```store/oracle/serverjre:8```镜像，这其实是Oracle JDK 8的一个官方镜像，第二行则是把当前目录的JVMDemo-1.0-SNAPSHOT.jar复制到容器内部的/opt/jar目录下，jar包就是上面maven项目生成的，最后一行是一条执行命令，也就是启动这个jar包。

```
FROM store/oracle/serverjre:8
COPY JVMDemo-1.0-SNAPSHOT.jar /opt/jar/JVMDemo-1.0-SNAPSHOT.jar
CMD java -jar /opt/jar/JVMDemo-1.0-SNAPSHOT.jar
```

执行```docker image build -t hello-java:latest .```命令来根据Dockerfile构建镜像，注意其中的```.```是表明在当前路径作为根路径，```-t```则是添加标签。
```
E:\docker\java-demo>docker image build -t hello-java:latest .
[+] Building 0.7s (7/7) FINISHED
 => [internal] load build definition from Dockerfile                                                                                                    0.1s
 => => transferring dockerfile: 181B                                                                                                                    0.0s
 => [internal] load .dockerignore                                                                                                                       0.1s
 => => transferring context: 2B                                                                                                                         0.0s
 => [internal] load metadata for docker.io/store/oracle/serverjre:8                                                                                     0.0s
 => [internal] load build context                                                                                                                       0.2s
 => => transferring context: 3.24kB                                                                                                                     0.0s
 => [1/2] FROM docker.io/store/oracle/serverjre:8                                                                                                       0.3s
 => => resolve docker.io/store/oracle/serverjre:8                                                                                                       0.0s
 => [2/2] COPY JVMDemo-1.0-SNAPSHOT.jar /opt/jar/JVMDemo-1.0-SNAPSHOT.jar                                                                               0.1s
 => exporting to image                                                                                                                                  0.1s
 => => exporting layers                                                                                                                                 0.1s
 => => writing image sha256:6de2db209e0e4175a00d99d91abb048ed3ca64da48a648e5620b99cc67fd6c66                                                            0.0s
 => => naming to docker.io/library/hello-java:latest
```

可以查看本地镜像会有刚才生成的hello-java

```
E:\docker\java-demo>docker image ls
REPOSITORY               TAG       IMAGE ID       CREATED          SIZE
hello-java               latest    6de2db209e0e   16 seconds ago   290MB
```

运行该镜像会输出对应信息，可以看到本质上就是只将所需要的环境集成在镜像内，以减小镜像大小，加快构建速度，响应的启动速度也会更快，占用资源也会更少。

```
E:\docker\java-demo>docker run --name java hello-java
hello world!
```


---

#### 3. 推送Docker镜像
Docker Hub是类似于Github一样的一个远程镜像仓库，里面有很多官方的镜像，也可以推送自己的上去。首先要登录自己的Doker Hub账号。

```
E:\docker\java-demo>docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: test
Password:
```

需要对镜像打标签，最后一段的格式是Docker Hub用户名/仓库名:标签名，如果没有对应仓库会自动创建一个仓库。

```
docker tag hello-java:latest nemohalf/hello-java:latest
```

最后使用push命令推送到远程仓库，下一次使用时，直接拉取到本地就可以。

```
E:\docker\java-demo>docker push nemohalf/hello-java:latest
The push refers to repository [docker.io/nemohalf/hello-java]
154fc6c09dec: Pushed
945301b2fd91: Pushed
11e019486fcd: Pushed
5102fc2ee26e: Pushed
latest: digest: sha256:12af8b4c6d474189d00b8eb3e9cb1b56106db5488619b0592f89babffce2da60 size: 1160
```

---


#### 4. docker-maven-plugin插件

maven还提供了docker构建的插件，只要添加下列配置即可把项目构建成镜像。

```
<plugin>
  <groupId>com.spotify</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>1.2.2</version>
    <configuration>
    <!-- <dockerHost>http://localhost:2375</dockerHost>-->
      <!--镜像名称-->
      <imageName>spotify-docker-java</imageName>
      <!--指定基础镜像-->
      <baseImage>java</baseImage>
      <!--ENTRYPOINT指令-->
      <entryPoint>["java", "-jar", "/${project.build.finalName}.jar"]</entryPoint>
      <resources>
        <resource>
          <targetPath>/</targetPath>
          <!--用于指定需要复制的根目录-->
          <directory>${project.build.directory}</directory>
          <!--用于指定需要复制的文件-->
          <include>${project.build.finalName}.jar</include>
        </resource>
      </resources>
    </configuration>
</plugin>
```

运行```mvn package docker:build```就可以执行构建命令，注意最下面会有构建步骤，但是需要Docker引擎对外暴露2375端口，否则会提示无法连接。

```
D:\download\JVMDemo>mvn package docker:build
[INFO] Scanning for projects...
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] Building JVMDemo 1.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO]
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ JVMDemo ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory D:\download\JVMDemo\src\main\resources
[INFO]
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ JVMDemo ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ JVMDemo ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory D:\download\JVMDemo\src\test\resources
[INFO]
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ JVMDemo ---
[INFO] No sources to compile
[INFO]
[INFO] --- maven-surefire-plugin:2.12.4:test (default-test) @ JVMDemo ---
[INFO]
[INFO] --- maven-jar-plugin:3.1.0:jar (default-jar) @ JVMDemo ---
[INFO]
[INFO] --- docker-maven-plugin:1.2.2:build (default-cli) @ JVMDemo ---
[INFO] Using authentication suppliers: [ConfigFileRegistryAuthSupplier]
[INFO] Copying D:\download\JVMDemo\target\JVMDemo-1.0-SNAPSHOT.jar -> D:\download\JVMDemo\target\docker\JVMDemo-1.0-SNAPSHOT.jar
[INFO] Building image spotify-docker-java
Step 1/3 : FROM java

 ---> d23bdf5b1b1b
Step 2/3 : ADD /JVMDemo-1.0-SNAPSHOT.jar //

 ---> Using cache
 ---> bdba5da04d67
Step 3/3 : ENTRYPOINT ["java", "-jar", "/JVMDemo-1.0-SNAPSHOT.jar"]

 ---> Using cache
 ---> 785f5ba92d2d
ProgressMessage{id=null, status=null, stream=null, error=null, progress=null, progressDetail=null}
Successfully built 785f5ba92d2d
Successfully tagged spotify-docker-java:latest
[INFO] Built spotify-docker-java
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 20.828 s
[INFO] Finished at: 2021-01-05T11:09:56+08:00
[INFO] Final Memory: 24M/144M
[INFO] ------------------------------------------------------------------------

```