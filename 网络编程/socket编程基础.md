# TCP

![img](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/1169746-20181001222742129-1182603054.png)

**accept函数是在三次握手之后的**

# UDP

![](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/20230110204837.png)



# 套接字函数

使用套接字通信函数需要包含头文件 <arpa/inet.h>，包含了这个头文件 <sys/socket.h> 就不用在包含了。

```c
// 创建一个套接字
int socket(int domain, int type, int protocol);
```

- 参数
  - domain: 使用的地址簇协议
    - AF_INET: 使用IPv4格式的ip地址
    - AF_INET6：使用IPv6格式的ip地址
  - type
    - SOCK_STREAM：使用流式的传输协议
    - SOCK_DGRAM：使用报式（报文）的传输协议
  - protocal，一般填0即可，使用默认的协议
    - SOCK_STREAM: 流式传输默认使用的是 tcp
    - SOCK_DGRAM: 报式传输默认使用的 udp
- 返回值
  - 成功：int，可用于套接字通信的文件描述符
  - 失败：-1

函数的返回值是一个文件描述符，通过这个文件描述符可以操作内核中的某一块内存，网络通信是基于这个文件描述符来完成的。

---

```c
// 将文件描述符和本地的IP与端口进行绑定   
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

- 参数
  - sockfd: 监听的文件描述符，通过 socket () 调用得到的返回值
  - addr: 传入参数，要绑定的 IP 和端口信息需要初始化到这个结构体中，IP和端口要转换为网络字节序
  - addrlen: 参数 addr 指向的内存大小，sizeof (struct sockaddr)

- 返回值：
  - 成功返回 0
  - 失败返回 - 1

---

```c
// 给监听的套接字设置监听
int listen(int sockfd, int backlog);
```

- 参数
  - sockfd: 文件描述符，可以通过调用 socket () 得到，在监听之前必须要绑定 bind ()
  - backlog: 同时能处理的最大连接要求，最大值为 128

- 返回值：
  - 成功返回 0
  - 失败返回 - 1

---

```c
// 等待并接受客户端的连接请求, 建立新的连接, 会得到一个新的文件描述符(通信的)		
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

- 参数
  - sockfd: 监听的文件描述符
  - addr: 传出参数，里边存储了建立连接的客户端的地址信息
  - addrlen: 传入传出参数，用于存储 addr 指向的内存大小

- 返回值：
  - 函数调用成功，得到一个文件描述符，用于和建立连接的这个客户端通信，调用失败返回 -1

`这个函数是一个阻塞函数，当没有新的客户端连接请求的时候，该函数阻塞；当检测到有新的客户端连接请求时，阻塞解除，新连接就建立了，得到的返回值也是一个文件描述符，基于这个文件描述符就可以和客户端通信了。`

---

```c
// 接收数据
ssize_t read(int sockfd, void *buf, size_t size);
ssize_t recv(int sockfd, void *buf, size_t size, int flags);
```

- 参数
  - sockfd: 用于通信的文件描述符，accept () 函数的返回值
  - buf: 指向一块有效内存，用于存储接收是数据
  - size: 参数 buf 指向的内存的容量
  - flags: 特殊的属性，一般不使用，指定为 0

- 返回值：
  - 大于 0：实际接收的字节数
  - 等于 0：对方断开了连接
  - -1：接收数据失败了

`如果连接没有断开，接收端接收不到数据，接收数据的函数会阻塞等待数据到达，数据到达后函数解除阻塞，开始接收数据，当发送端断开连接，接收端无法接收到任何数据，但是这时候就不会阻塞了，函数直接返回0。`

---

```c
// 发送数据的函数
ssize_t write(int fd, const void *buf, size_t len);
ssize_t send(int fd, const void *buf, size_t len, int flags);
```

- 参数
  - fd: 通信的文件描述符，accept () 函数的返回值
  - buf: 传入参数，要发送的字符串
  - len: 要发送的字符串的长度
  - flags: 特殊的属性，一般不使用，指定为 0

- 返回值：
  - 大于 0：实际发送的字节数，和参数 len 是相等的
  - -1：发送失败

---

```c
// 成功连接服务器之后, 客户端会自动随机绑定一个端口
// 服务器端调用accept()的函数, 第二个参数存储的就是客户端的IP和端口信息
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

- 参数
  - sockfd: 通信的文件描述符，通过调用 socket () 函数就得到了
  - addr: 存储了要连接的服务器端的地址信息: iP 和 端口，这个 IP 和端口也需要转换为大端然后再赋值
  - addrlen: addr 指针指向的内存的大小 sizeof (struct sockaddr)

- 返回值：
  - 连接成功返回 0
  - 连接失败返回 - 1



# 一些关键的结构

1. 描述与保存网络地址

![img](https://subingwen.cn/linux/socket/sockaddr.png)



```c
// 在写数据的时候不好用
struct sockaddr {
	sa_family_t sa_family;       // 地址族协议, ipv4
	char        sa_data[14];     // 端口(2字节) + IP地址(4字节) + 填充(8字节)
}

typedef unsigned short  uint16_t;
typedef unsigned int    uint32_t;
typedef uint16_t in_port_t;
typedef uint32_t in_addr_t;
typedef unsigned short int sa_family_t;
#define __SOCKADDR_COMMON_SIZE (sizeof (unsigned short int))

struct in_addr
{
    in_addr_t s_addr;
};  

// sizeof(struct sockaddr) == sizeof(struct sockaddr_in)
struct sockaddr_in
{
    sa_family_t sin_family;		/* 地址族协议: AF_INET */
    in_port_t sin_port;         /* 端口, 2字节-> 大端  */
    struct in_addr sin_addr;    /* IP地址, 4字节 -> 大端  */
    /* 填充 8字节 */
    unsigned char sin_zero[sizeof (struct sockaddr) - sizeof(sin_family) -
               sizeof (in_port_t) - sizeof (struct in_addr)];
};  
```

2. 描述close函数关闭方式

```c
#include <arpa/inet.h>

struct linger {
　　int l_onoff;
　　int l_linger;
};

struct linger ling = {0, 0};
setsockopt(socketfd, SOL_SOCKET, SO_LINGER, (void*)&ling, sizeof(ling));
```



- l_onoff = 0; l_linger忽略

  close()立刻返回，底层会将未发送完的数据发送完成后再释放资源，即优雅退出。

- l_onoff != 0; l_linger = 0;

  close()立刻返回，但不会发送未发送完成的数据，而是通过一个REST包强制的关闭socket描述符，即强制退出。

- l_onoff != 0; l_linger > 0;

  close()不会立刻返回，内核会延迟一段时间，这个时间就由l_linger的值来决定。如果超时时间到达之前，发送完未发送的数据(包括FIN包)并得到另一端的确认，close()会返回正确，socket描述符优雅性退出。否则，close()会直接返回错误值，未发送数据丢失，socket描述符被强制性退出。需要注意的时，如果socket描述符被设置为非堵塞型，则close()会直接返回值。



# 参考文献

https://subingwen.cn/linux/socket/#3-1-%E5%AD%97%E8%8A%82%E5%BA%8F