### nginx的基本使用

---

#### 1. Nginx基础命令

直接停止nginx，不管底层的任务有没有完成
```
nginx -s stop
```
等到底层的任务都结束了再关闭
```
nginx -s quit
```
修改配置文件后，重新加载配置文件
```  
- nginx -s reload
```
配置文件中设置日志名字是test，这时如果修改名字为test1还是会继续往test1文件中写入日志，执行此命令后才会重新创建一个test日志文件
```  
- nginx -s reopen
```
----

#### 2. 基础配置
   
Nginx指令，nginx由各种模块组成，这些模块由配置文件中指定的指令控制。指令分为可以大致分为简单指令和复杂指令。
  
简单指令就是由字符串组成的
```
index index.html index.htm;
```
复杂指令就是一对大括号中有多个简单指令
```
location ~ \.(html|js|css)$ {
  root   html;
  index  index.html index.htm;
}
```

指令上下文，每一个复杂指令就是一个指令上下文，因为它里面的简单指令都是在它下面才生效
  
- main，最外层的指令，所有的指令都是包含在main之内
- http，控制网络相关
- server，控制服务器配置
- location，控制各种访问路径配置

---

#### 3. location配置

```location```指令需要配置在```server```指令里，可以配置多个。
   
   
匹配```/```地址，访问```/data/www```这个路径。
  
``` 
location / { 
  root /data/www; 
}
```

匹配包含```/images/```的请求，访问```/data/images/```路径查找文件。比如访问```http://localhost/9090/images/test.jpg```，如果有```test.jpg```这个文件则会返回这个文件，否则会跳转到404页面
  
```
location /images/ { 
  root /data; 
}
```

匹配所有以```gif```、```jpg```或```png```结尾的请求，在```/data/images```路径查找文件。
```
location ~\.(gif | jpg | png)$ { 
  root /data/images; 
}
```