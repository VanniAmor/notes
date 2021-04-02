## 生成公钥、私钥

有多少方式可以生成可信任的公钥、私钥，包括但不限于git、ssh-keygen、MobaXterm，这里介绍ssh-keygen方式

![image-20210402201614963](C:%5CUsers%5Cwb.luoweile%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20210402201614963.png)



在`/root/.ssh`下，会生成以下文件：

- authorized_keys:  存放远程免密登录的公钥,主要通过这个文件记录多台机器的公钥

- id_rsa:  生成的私钥文件
- id_rsa.pub:   生成的公钥文件
- know_hosts:  已知的主机公钥清单

私钥发送给客户端，公钥保存在 authorized_keys



**注意，不同用户的公钥是保存在其自己的home目录下.ssh中的authorized_key文件中**



## 配置登录方式

```shell
vim  /etc/ssh/sshd_config 

StrictModes no
# 开启rsa验证 
RSAAuthentication yes 
# 是否使用公钥 
PubkeyAuthentication yes 
# 公钥保存位置 
AuthorizedKeysFile    .ssh/authorized_keys 
# 禁止使用密码登录 
PasswordAuthentication no: 
```



注意，修改完配置文件后，需要重启sshd服务

```shell
systemctl restart sshd
```



## 参考文献

https://blog.csdn.net/zhou920786312/article/details/114334427

https://zhidao.baidu.com/question/1734589244049891427.html

https://blog.csdn.net/qq_15504597/article/details/82839414