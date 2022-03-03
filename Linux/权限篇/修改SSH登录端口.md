按序完成以下步骤，修改SSH登录端口，操作账号均为root

默认的SSH登录端口是22， 将其修改为 10010



### 修改sshd_config



```shell
vim /etc/ssh/sshd_config
#Port 22
#添加一行

Port 10010
```



### 检查端口放行情况

```shell
# 1.检查SSH端口放行情况
semanage port -l | grep ssh
> ssh_port_t                     tcp      22
# 2.在没有放行的情况下，主动添加放行端口
semanage port -a -t ssh_port_t -p tcp 10010
# 3.检查端口有没有在防火墙放行
firewall-cmd --permanent --query-port=10010/tcp
> no
# 4.在防火墙开启新的SSH端口
firewall-cmd --permanent --add-port=10010/tcp --permanent
> success
# 5.在防火墙移除旧的SSH端口
firewall-cmd --permanent --zone=public --remove-port=22/tcp
> Warning: NOT_ENABLED: 22:tcp
> success
firewall-cmd --reload
> success
systemctl restart sshd
systemctl restart firewalld.service
# 6.查看防火墙端口情况
firewall-cmd --list-ports
> 10010/tcp
```





http://blog.sina.com.cn/s/blog_979924390102yur9.html

