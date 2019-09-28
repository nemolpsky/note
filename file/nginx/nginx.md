## nginx的基本使用

1.启动、停止和重新加载配置文件
- 执行目录下nginx.exe就可以运行
- nginx -s stop
  - 直接停止nginx，不管底层的任务有没有完成
- nginx -s quit
  - 等到底层的任务都结束了再关闭
- nginx -s reload
  - 修改配置文件后，重新加载配置文件
- nginx -s reopen
  - 配置文件中设置日志名字是test，这时如果修改名字为test1还是会继续往test1文件中写入日志，执行此命令后才会重新创建一个test日志文件

2. 基础配置
   
   指令，nginx由各种模块组成，这些模块由配置文件中指定的指令控制。指令分为可以大致分为简单指令和复杂指令。
  
   - 简单指令就是由字符串组成的

     ```
     index index.html index.htm;
     ```
   - 复杂指令就是一对大括号中有多个简单指令
    
     ```
     location ~ \.(html|js|css)$ {
          root   html;
          index  index.html index.htm;
     }
     ```
   - 指令上下文，每一个复杂指令就是一个指令上下文，因为它里面的简单指令都是在它下面才生效
  
     - main，最外层的指令，所有的指令都是包含在main之内
     - http，控制网络相关
     - server，控制服务器配置
     - location，控制各种访问路径配置

3. location配置

   ```location```指令需要配置在```server```指令里，可以配置多个，nginx会选择匹配到的最长的前缀的那个。
   - 下面这个配置表示会用```/```匹配访问过来的请求地址，如果匹配到了就会访问```/data/www```这个路径。
  
   ``` 
   location / { 
     root / data / www; 
   }
   ```

   - 这个配置则是匹配包含```/images/```的请求，如果匹配到了则访问```/data/images/```路径查找文件。比如访问```http://localhost/9090/images/test.jpg```，如果有```test.jpg```这个文件则会返回这个文件，否则会跳转到404页面
  
   ```
   location /images/ { 
     root /data; 
   }
   ```

   - 这个配置则是匹配所有以```gif```、```jpg```或```png```结尾的请求会到```/data/images```路径查找文件。
   ```
   location ~\.(gif | jpg | png)$ { 
     root /data/images; 
   }
   ```