### 安装Elasticsearch

在官网下载对应Linux版本的压缩包，解压，因为Elasticsearch不允许使用root账号运行，所以需要新建一个账号，赋予该账号操作Elasticsearch目录的权限。最后再运行bin目录下的elasticsearch脚本就可以。

```
tar -zxvf elasticsearch-7.10.1-linux-x86_64.tar.gz
sudo chown -R half:half /opt/elasticsearch-7.10.1
```

有时会提示这个错误，需要修改config目录下的elasticsearch.yml配置文件，添加两行对应配置即可。默认端口是9200，访问即可看到返回的信息。

```
java.lang.UnsupportedOperationException: seccomp unavailable: CONFIG_SECCOMP not compiled into kernel, CONFIG_SECCOMP and CONFIG_SECCOMP_FILTER are needed
        at org.elasticsearch.bootstrap.SystemCallFilter.linuxImpl(SystemCallFilter.java:342) ~[elasticsearch-7.10.1.jar:7.10.1]
```

```
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
```

---

### 安装Kibana
Kibana是官方提供的可视化工具，和Elasticsearch一样解压即运行bin目录下的脚本即可，默认端口是5601。选择左边目录Kibana下的Discover选项，添加对应索引的筛选条件，即可筛选对应的数据。

```
tar -zxvf kibana-7.10.1-linux-x86_64.tar.gz
sudo chown -R half:half /opt/kibana-7.10.1-linux-x86_64
```



> https://www.elastic.co/cn/downloads/elasticsearch

> https://www.elastic.co/cn/downloads/kibana