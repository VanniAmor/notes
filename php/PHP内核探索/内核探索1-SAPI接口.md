SAPI, Server Application Programming Interface, 服务端应用编程端口。

PHP内核向外提供SAPI，让外部系统可以通过SAPI调用php提供的编译脚本、执行脚本等服务。

![php运行生命周期](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210122152636971.png)

常见的SAPI共有四种：

- CGI
- Fast-CGI
- CLI
- Apache的模块dll

这也是PHP的常见运行方式，在php编译后，就会看到php.exe, php-cgi, php-fpm，这些都是实现php实现SAPI的程序。

脚本的执行往往都是以SAPI接口的实现开始。只是不同的SAPI接口实现会完成他们特定的工作， 例如Apache的mod_php SAPI实现需要初始化从Apache获取的一些信息，在输出内容是将内容返回给Apache， 其他的SAPI实现也类似。

下文以CGI的SAPI为例，深入理解SAPI的工作原理

## SAPI定义

要定义SAPI，先要定义一个 `sapi_module_struct`， 源码位置在 ` PHP-SRC/sapi/cgi/cgi_main.c` 

在这个结构里包含了这个PHP向外暴露的功能接口（如php模块初始化，接收传入的参数等等，php），这也是SAPI接口的功能

```c
*/
static sapi_module_struct cgi_sapi_module = {
#if PHP_FASTCGI
    "cgi-fcgi",                     /* name */
    "CGI/FastCGI",                  /* pretty name */
#else
    "cgi",                          /* name */
    "CGI",                          /* pretty name */
#endif
 
    php_cgi_startup,                /* startup */
    php_module_shutdown_wrapper,    /* shutdown */
 
    NULL,                           /* activate */
    sapi_cgi_deactivate,            /* deactivate */
 
    sapi_cgibin_ub_write,           /* unbuffered write */
    sapi_cgibin_flush,              /* flush */
    NULL,                           /* get uid */
    sapi_cgibin_getenv,             /* getenv */
 
    php_error,                      /* error handler */
 
    NULL,                           /* header handler */
    sapi_cgi_send_headers,          /* send headers handler */
    NULL,                           /* send header handler */
 
    sapi_cgi_read_post,             /* read POST data */
    sapi_cgi_read_cookies,          /* read Cookies */
 
    sapi_cgi_register_variables,    /* register server variables */
    sapi_cgi_log_message,           /* Log message */
    NULL,                           /* Get request time */
 
    STANDARD_SAPI_MODULE_PROPERTIES
};
```

结构内包含了一些常量，比如name，会在调用php_info()的时候被使用。一些初始化，收尾函数，以及一些函数指针，用于告诉Zend以及如何获取和输出数据。

以下是该SAPI提供的函数功能

（1）php_cgi_startup, 当一个应用要调用PHP的时候，这个函数会被调用。对于CGI来说，它只是简单的调用了PHP的初始化函数

```c
static int php_cgi_startup(sapi_module_struct *sapi_module)
{
    if (php_module_startup(sapi_module, NULL, 0) == FAILURE) {
        return FAILURE;
    }
    return SUCCESS;
} 
```

（2）php_module_shutdown_wrapper，一个对PHP关闭函数的简单包装。只是简单的调用php_module_shutdown;

（3） PHP会在每个request的时候，处理一些初始化，资源分配的事务。这部分就是activate字段要定义的，从上面的结构我们可以看出，对于CGI来说，它并没有提供初始化处理句柄。对于mod_php来说，那就不同了，他要在apache的pool中注册资源析构函数， 申请空间， 初始化环境变量，等等。

（4） sapi_cgi_deactivate, 这个是对应与activate的函数，顾名思义，它会提供一个handler, 用来处理收尾工作，对于CGI来说，他只是简单的刷新缓冲区，用以保证用户在Zend关闭前得到所有的输出数据：

```c
static int sapi_cgi_deactivate(TSRMLS_D)
{
    /* flush only when SAPI was started. The reasons are:
        1. SAPI Deactivate is called from two places: module init and request shutdown
        2. When the first call occurs and the request is not set up, flush fails on
            FastCGI.
    */
    if (SG(sapi_started)) {
        sapi_cgibin_flush(SG(server_context));
    }
    return SUCCESS;
}
```

（5）sapi_cgibin_ub_write, 这个hanlder告诉了Zend，如何输出数据，对于mod_php来说，这个函数提供了一个向response数据写的接口，而对于CGI来说，只是简单的写到stdout：

```c
static inline size_t sapi_cgibin_single_write(const char *str, uint str_length TSRMLS_DC)
{
#ifdef PHP_WRITE_STDOUT
    long ret;
#else
    size_t ret;
#endif
 
#if PHP_FASTCGI
    if (fcgi_is_fastcgi()) {
        fcgi_request *request = (fcgi_request*) SG(server_context);
        long ret = fcgi_write(request, FCGI_STDOUT, str, str_length);
        if (ret <= 0) {
            return 0;
        }
        return ret;
    }
#endif
#ifdef PHP_WRITE_STDOUT
    ret = write(STDOUT_FILENO, str, str_length);
    if (ret <= 0) return 0;
    return ret;
#else
    ret = fwrite(str, 1, MIN(str_length, 16384), stdout);
    return ret;
#endif
}
 
static int sapi_cgibin_ub_write(const char *str, uint str_length TSRMLS_DC)
{
    const char *ptr = str;
    uint remaining = str_length;
    size_t ret;
 
    while (remaining > 0) {
        ret = sapi_cgibin_single_write(ptr, remaining TSRMLS_CC);
        if (!ret) {
            php_handle_aborted_connection();
            return str_length - remaining;
        }
        ptr += ret;
        remaining -= ret;
    }
 
    return str_length;
}
```

（6）sapi_cgibin_flush, 这个是提供给zend的刷新缓存的函数句柄，对于CGI来说，只是简单的调用系统提供的flush;

（7）NULL， 这部分用来让Zend可以验证一个要执行脚本文件的state，从而判断文件是否据有执行权限等等，CGI没有提供。

（8）sapi_cgibin_getenv, 为Zend提供了一个根据name来查找环境变量的接口，对于mod_php5来说，当我们在脚本中调用getenv的时候，就会间接的调用这个句柄。而对于CGI来说，因为他的运行机制和CLI很类似，直接调用父级是Shell， 所以，只是简单的调用了系统提供的genenv:

```c
static char *sapi_cgibin_getenv(char *name, size_t name_len TSRMLS_DC)
{
#if PHP_FASTCGI
    /* when php is started by mod_fastcgi, no regular environment
       is provided to PHP.  It is always sent to PHP at the start
       of a request.  So we have to do our own lookup to get env
       vars.  This could probably be faster somehow.  */
    if (fcgi_is_fastcgi()) {
        fcgi_request *request = (fcgi_request*) SG(server_context);
        return fcgi_getenv(request, name, name_len);
    }
#endif
    /*  if cgi, or fastcgi and not found in fcgi env
        check the regular environment */
    return getenv(name);
}
```

（9）php_error, 错误处理函数, 到这里，说几句题外话，上次看到php maillist 提到的使得PHP的错误处理机制完全OO化， 也就是，改写这个函数句柄，使得每当有错误发生的时候，都throw一个异常。而CGI只是简单的调用了PHP提供的错误处理函数。

（10） 这个函数会在我们调用PHP的header()函数的时候被调用，对于CGI来说，不提供。

（11） sapi_cgi_send_headers， 这个函数会在要真正发送header的时候被调用，一般来说，就是当有任何的输出要发送之前：

```c
static int sapi_cgi_send_headers(sapi_headers_struct *sapi_headers TSRMLS_DC)
{
    char buf[SAPI_CGI_MAX_HEADER_LENGTH];
    sapi_header_struct *h;
    zend_llist_position pos;
 
    if (SG(request_info).no_headers == 1) {
        return  SAPI_HEADER_SENT_SUCCESSFULLY;
    }
 
    if (cgi_nph || SG(sapi_headers).http_response_code != 200)
    {
        int len;
 
        if (rfc2616_headers && SG(sapi_headers).http_status_line) {
            len = snprintf(buf, SAPI_CGI_MAX_HEADER_LENGTH,
                           "%s\r\n", SG(sapi_headers).http_status_line);
 
            if (len > SAPI_CGI_MAX_HEADER_LENGTH) {
                len = SAPI_CGI_MAX_HEADER_LENGTH;
            }
 
        } else {
            len = sprintf(buf, "Status: %d\r\n", SG(sapi_headers).http_response_code);
        }
 
        PHPWRITE_H(buf, len);
    }
 
    h = (sapi_header_struct*)zend_llist_get_first_ex(&sapi_headers->headers, &pos);
    while (h) {
        /* prevent CRLFCRLF */
        if (h->header_len) {
            PHPWRITE_H(h->header, h->header_len);
            PHPWRITE_H("\r\n", 2);
        }
        h = (sapi_header_struct*)zend_llist_get_next_ex(&sapi_headers->headers, &pos);
    }
    PHPWRITE_H("\r\n", 2);
 
    return SAPI_HEADER_SENT_SUCCESSFULLY;
   }
```

（12）NULL, 这个用来单独发送每一个header, CGI没有提供

（13）sapi_cgi_read_post, 这个句柄指明了如何获取POST的数据，如果做过CGI编程的话，我们就知道CGI是从stdin中读取POST DATA的：

```C
static int sapi_cgi_read_post(char *buffer, uint count_bytes TSRMLS_DC)
{
    uint read_bytes=0, tmp_read_bytes;
#if PHP_FASTCGI
    char *pos = buffer;
#endif
 
    count_bytes = MIN(count_bytes, (uint) SG(request_info).content_length - SG(read_post_bytes));
    while (read_bytes < count_bytes) {
#if PHP_FASTCGI
        if (fcgi_is_fastcgi()) {
            fcgi_request *request = (fcgi_request*) SG(server_context);
            tmp_read_bytes = fcgi_read(request, pos, count_bytes - read_bytes);
            pos += tmp_read_bytes;
        } else {
            tmp_read_bytes = read(0, buffer + read_bytes, count_bytes - read_bytes);
        }
#else
        tmp_read_bytes = read(0, buffer + read_bytes, count_bytes - read_bytes);
#endif
 
        if (tmp_read_bytes <= 0) {
            break;
        }
        read_bytes += tmp_read_bytes;
    }
    return read_bytes;
}
```

（14）sapi_cgi_read_cookies,获取cookies值

```c
static char *sapi_cgi_read_cookies(TSRMLS_D)
{
    return sapi_cgibin_getenv((char *) "HTTP_COOKIE", sizeof("HTTP_COOKIE")-1 TSRMLS_CC);
}
```

（15） sapi_cgi_register_variables, 这个函数给了一个接口，用以给`$_SERVER`变量中添加变量，对于CGI来说，注册了一个PHP_SELF,这样我们就可以在脚本中访问`$_SERVER['PHP_SELF']`来获取本次的request_uri：

```c
static void sapi_cgi_register_variables(zval *track_vars_array TSRMLS_DC)
{
    /* In CGI mode, we consider the environment to be a part of the server
     * variables
     */
    php_import_environment_variables(track_vars_array TSRMLS_CC);
    /* Build the special-case PHP_SELF variable for the CGI version */
    php_register_variable("PHP_SELF", (SG(request_info).request_uri ? SG(request_info).request_uri : ""), track_vars_array TSRMLS_CC);
}
```

（16）sapi_cgi_log_message ，用来输出错误信息，对于CGI来说，只是简单的输出到stderr：

```c
static void sapi_cgi_log_message(char *message)
{
#if PHP_FASTCGI
    if (fcgi_is_fastcgi() && fcgi_logging) {
        fcgi_request *request;
        TSRMLS_FETCH();
 
        request = (fcgi_request*) SG(server_context);
        if (request) {
            int len = strlen(message);
            char *buf = malloc(len+2);
 
            memcpy(buf, message, len);
            memcpy(buf + len, "\n", sizeof("\n"));
            fcgi_write(request, FCGI_STDERR, buf, len+1);
            free(buf);
        } else {
            fprintf(stderr, "%s\n", message);
        }
        /* ignore return code */
    } else
#endif /* PHP_FASTCGI */
    fprintf(stderr, "%s\n", message);
}
```



综上就是一个SAPI接口中的定义，webServer等通过SAPI接口来调用Zend引擎提供的php服务，从而解析php脚本，执行php程序等等。

## 参考资料

https://www.php.cn/php-weizijiaocheng-410435.html

http://www.nowamagic.net/librarys/veda/detail/1285