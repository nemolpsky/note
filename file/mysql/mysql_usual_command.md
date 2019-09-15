## 读写模式
- 设置数据库状态为"只读"
  
  ```
  set global read_only=1;
  ```
- 设置数据库状态为"读写"
  
  ```
  set global read_only=0;
  ```
- 查询数据库读写状态
  
  ```
  show global variables like "%read_only%";
  ```

---

## 用户管理
- 添加用户

  ```
  //创建用户user，'user'@'%'表示可以被远程访问
  //如果是'user’@'localhost'，表示只能本地访问
  CREATE USER 'user'@'%' IDENTIFIED BY '123456';
  ```

- 授予用户权限

  ```
  //为用户user授予对数据库queenoa下所有的操作权限
  GRANT ALL PRIVILEGES ON queenoa.* TO 'user'@'%';
  ```

- 查询用户权限

  ```
  SHOW GRANTS FOR 'user'@'%';
  SHOW GRANTS FOR USER;
  ```

- 删除权限
  
  ```
  // 删除用户对数据库queenoa下所有的操作权限
  REVOKE ALL ON queenoa.* FROM 'user'@'%';
  ```

- 删除匿名账户(解决创建新用户后 Access denied for user 'x'@'localhost' 无法登录问题)
  ```
  //删除数据库中的空白账户（匿名账户），依次执行以下两条指命
  DELETE FROM `mysql`.`user` WHERE `user`='';
  FLUSH PRIVILEGES;
  ```

- 修改密码
  
  ```
  alter user 'root'@'localhost' identified by '123';
  flush privileges;
  ```
---

