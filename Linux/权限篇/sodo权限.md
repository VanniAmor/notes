## 设置普通用户的SUDO权限

- 进入root，编辑sudo配置文件

  ```shell
  # su 进入root用户
  # vim /etc/sudoers 编辑sudo的配置文件
  ```

- 修改soduers文件，给予普通用户权限，这里的普通用户是 `jingkong`

  ```shell
  [root@muguangjingkong jingkong]# vi /etc/sudoers
  
  jingkong ALL=(root)NOPASSWD:ALL
  ## Sudoers allows particular users to run various commands as
  ## the root user, without needing the root password.
  ##
  ```

  注:sudoers文件只是一个只读文件，可以用:`wq!`强制修改，但是如果你觉得不安全，可以在修改文件之前先赋予文件写权限（W），修改保存之后再收回写权限，操作如下：

  ```shell
  chmod u+w /etc/sudoers 
  
  -- 修改完成之后收回权限： 
  
  chmod u-w /etc/sudoers
  ```

  

### 测试

获取权限前

```shell
[root@muguangjingkong jingkong]# su - jingkong
[jingkong@muguangjingkong ~]$ sudo vi /etc/hosts
[sudo] password for jingkong: 
Sorry, try again.
[sudo] password for jingkong: 
jingkong is not in the sudoers file.  This incident will be reported.
```

获取权限后

```shell
[jingkong@muguangjingkong ~]$ sudo vi /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```



## 参考文献

https://blog.csdn.net/muguangjingkong/article/details/105599514