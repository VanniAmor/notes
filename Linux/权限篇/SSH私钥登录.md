修改以往传统的用户名密码登录方式，改为使用秘钥进行登录，主机安全性得以提高

按序执行以下步骤，均以root账号运行

- 生成公钥私钥
- 配置ssh登录方式
- 导入秘钥到对应账号



## 生成公钥、私钥

有多少方式可以生成可信任的公钥、私钥，包括但不限于git、ssh-keygen、MobaXterm，这里介绍ssh-keygen方式

```shell
sudo ssh-keygen -t rsa -b 4096
```

![](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/20180925145147317)



私钥发送给客户端，公钥保存在 authorized_keys



## 配置登录方式

```shell
vim  /etc/ssh/sshd_config 

# 是否使用公钥 
PubkeyAuthentication yes 
# 公钥保存位置 
AuthorizedKeysFile    .ssh/authorized_keys 
# 禁止使用密码登录 
PasswordAuthentication no
# 允许ROOT登录
PermitRootLogin yes
```



注意，修改完配置文件后，需要重启sshd服务

```shell
systemctl restart sshd
```





## 给ROOT账号设置秘钥登录

```shell
#导入秘钥（需要先生成秘钥, 秘钥地址指定为 /roottest_rsa）
cat test_rsa.pub >> /root/.ssh/authorized_keys

#检查确认
cat /home/zero/.ssh/authorized_keys

#修改权限
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys
```





## 指定账号设置秘钥登录

```shell
#添加用户
sudo  useradd  zero -d /home/zero

#设定sudo密码
sudo  passwd  zero
#请输入新用户sudo密码

#创建目录及文件
sudo mkdir /home/zero/.ssh
sudo touch /home/zero/.ssh/authorized_keys

#修正所有者
sudo chown -R zero. /home/zero/.ssh

#导入秘钥（需要先生成秘钥, 秘钥地址指定为 /home/zero/test_rsa）
sudo sh -c 'cat zero_rsa.pub >> /home/zero/.ssh/authorized_keys'

#检查确认
sudo cat /home/zero/.ssh/authorized_keys

#修改权限
sudo chmod 700 /home/zero/.ssh
sudo chmod 600 /home/zero/.ssh/authorized_keys
————————————————
版权声明：本文为CSDN博主「Poison_毒药」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_15504597/article/details/82839414
```





## 参考文献

https://blog.csdn.net/zhou920786312/article/details/114334427

https://zhidao.baidu.com/question/1734589244049891427.html

https://blog.csdn.net/qq_15504597/article/details/82839414