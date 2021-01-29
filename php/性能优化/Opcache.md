目前为php提供opcode缓存的扩展有很多，比如：Zend Opcache,xcache,eAccelerator,apc等等。

但是目前其他产品都逐渐被淘汰

- Zend Opcache为官方产品，保证版本的兼容和持续更新
- Zend Opcache适配php5，且在5.5后都自带了opcache，无须额外安装
- Zend Opcache性能最为优越

下文讨论的均是Zend自带的opcache拓展

## Opcache原理

编译器把源程序的每一条语句都编译成机器语言，并保存成二进制文件，这样运行的时候直接以机器语言来运行，速度很快。

解释型语言的实现里，翻译器并不产生目标机器代码，而是产生易于执行的中间代码。这种中间代码，并不能直接执行，需要在虚拟机中进一步解析为可执行的机器指令。

**对PHP来说，PHP并不产生机器码，解析后产生中间码opCode，然后Zend引擎解析执行opCode**

php执行流程大概如下：

![image-20210128112718408](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210128112718408.png)

1. request请求（nginx，cli，apache等）
2. zend引擎读取.php文件
3. 词法分析
4. 语法分析
5. 解析文件，得到opCode
6. zend引擎执行opCode
7. 得到结果并返回

显然，如果PHP源码没有发生变化，那么最终执行的opCode也不会变化，没有必要每次都重新生成opCode。我们可以把opCode缓存下来，以后直接访问

**Opcode cache 的目地是避免重复编译，减少 CPU 和内存开销。**



## opCache安装

### php7

1. 编译安装的时候，设置参数 `--enable-opcache`

2. PHP7之后，已经内置了opcache拓展，但是没有开启，需要在php.ini中添加 ` zend_extension="opcache.so"` 后，把`opcache.enable = 1`

![image-20210128141701035](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210128141701035.png)

通过打印phpinfo()可以查看opCache拓展的状态

**【注意】php7后的版本，无法通过php -m来查看是否安装了opcache**

**【注意】当在编译安装php时，没有打上 --enable-opcache参数，需要编译安装opcache拓展**

### php5

1. 下载拓展，编译安装

```shell
https://pecl.php.net/get/zendopcache-7.0.5.tgz
tar xzf zendopcache-7.0.5.tgz
cd zendopcache-7.0.5
/usr/local/php/bin/phpize

./configure --with-php-config=/usr/local/php/bin/php-config
make
make install
```

2. 修改php.ini

```ini
zend_extension=/usr/local/php/lib/php/extensions/no-debug-zts-20100525/opcache.so
opcache.memory_consumption=128    //共享内存的大小, 总共能够存储多少预编译的 PHP 代码(单位:MB) --- 推荐 128
opcache.interned_strings_buffer=8     //最大缓存的文件数目 200  到 100000 之间---  推荐 4000
opcache.max_accelerated_files=4000    //内存“浪费”达到此值对应的百分比,就会发起一个重启调度
opcache.revalidate_freq=60            //允许或禁止在 include_path 中进行文件搜索的优化   单位 秒
opcache.fast_shutdown=1                //允许覆盖文件存在（file_exists等）的优化特性
opcache.enable_cli=1                   //关闭时代码不再优化.
```



## opCache常用配置介绍

```config
opcache.enable boolean
    启用操作码缓存。如果禁用此选项，则不会优化和缓存代码。 在运行期使用 ini_set() 函数只能禁用 opcache.enable 设置，不可以启用此设置。 如果在脚本中尝试启用此设置项会产生警告。

opcache.enable_cli boolean
    仅针对 CLI 版本的 PHP 启用操作码缓存。 通常被用来测试和调试。

opcache.memory_consumption integer
    OPcache 的共享内存大小，以兆字节为单位。

opcache.interned_strings_buffer integer
    用来存储临时字符串的内存大小，以兆字节为单位。 PHP 5.3.0 之前的版本会忽略此配置指令。

opcache.max_accelerated_files integer
    OPcache 哈希表中可存储的脚本文件数量上限。 真实的取值是在质数集合 { 223, 463, 983, 1979, 3907, 7963, 16229, 32531, 65407, 130987 } 中找到的第一个比设置值大的质数。 设置值取值范围最小值是 200，最大值在 PHP 5.5.6 之前是 100000，PHP 5.5.6 及之后是 1000000。

opcache.max_wasted_percentage integer
    浪费内存的上限，以百分比计。 如果达到此上限，那么 OPcache 将产生重新启动续发事件。

opcache.use_cwd boolean
    如果启用，OPcache 将在哈希表的脚本键之后附加改脚本的工作目录， 以避免同名脚本冲突的问题。 禁用此选项可以提高性能，但是可能会导致应用崩溃。

opcache.validate_timestamps boolean
    如果启用，那么 OPcache 会每隔 opcache.revalidate_freq 设定的秒数 检查脚本是否更新。 如果禁用此选项，你必须使用 opcache_reset() 或者 opcache_invalidate() 函数来手动重置 OPcache，也可以 通过重启 Web 服务器来使文件系统更改生效。

opcache.revalidate_freq integer
    检查脚本时间戳是否有更新的周期，以秒为单位。 设置为 0 会导致针对每个请求， OPcache 都会检查脚本更新。
    如果 opcache.validate_timestamps 配置指令设置为禁用，那么此设置项将会被忽略。

opcache.revalidate_path boolean
    如果禁用此选项，在同一个 include_path 已存在的缓存文件会被重用。 因此，将无法找到不在包含路径下的同名文件。

opcache.save_comments boolean
    如果禁用，脚本文件中的注释内容将不会被包含到操作码缓存文件， 这样可以有效减小优化后的文件体积。 禁用此配置指令可能会导致一些依赖注释或注解的 应用或框架无法正常工作， 比如： Doctrine， Zend Framework 2 以及 PHPUnit。

opcache.load_comments boolean
    如果禁用，则即使文件中包含注释，也不会加载这些注释内容。 本选项可以和 opcache.save_comments 一起使用，以实现按需加载注释内容。

opcache.fast_shutdown boolean
    如果启用，则会使用快速停止续发事件。 所谓快速停止续发事件是指依赖 Zend 引擎的内存管理模块 一次释放全部请求变量的内存，而不是依次释放每一个已分配的内存块。

opcache.enable_file_override boolean
    如果启用，则在调用函数 file_exists()， is_file() 以及 is_readable() 的时候， 都会检查操作码缓存，无论文件是否已经被缓存。 如果应用中包含检查 PHP 脚本存在性和可读性的功能，这样可以提升性能。 但是如果禁用了 opcache.validate_timestamps 选项， 可能存在返回过时数据的风险。

opcache.optimization_level integer
    控制优化级别的二进制位掩码。

opcache.inherited_hack boolean
    在 PHP 5.3 之前的版本，OPcache 会存储代码中使用 DECLARE_CLASS 操作码 来实现继承的位置。当文件被加载之后，OPcache 会尝试使用当前环境来绑定被继承的类。 由于当前脚本中可能并不需要 DECLARE_CLASS 操作码，如果这样的脚本需要对应的操作码被定义时， 可能无法运行。
    在 PHP 5.3 及后续版本中，此配置指令会被忽略。

opcache.dups_fix boolean
    仅作为针对 “不可重定义类”错误的一种解决方案。

opcache.blacklist_filename string
    OPcache 黑名单文件位置。 黑名单文件为文本文件，包含了不进行预编译优化的文件名，每行一个文件名。 黑名单中的文件名可以使用通配符，也可以使用前缀。 此文件中以分号（;）开头的行将被视为注释。
	; 将特定文件加入到黑名单
    /var/www/broken.php
    ; 以字符 x 文件打头的文件
    /var/www/x
    ; 通配符匹配
    /var/www/*-broken.php
    
opcache.max_file_size integer
    以字节为单位的缓存的文件大小上限。设置为 0 表示缓存全部文件。

opcache.consistency_checks integer
    如果是非 0 值，OPcache 将会每隔 N 次请求检查缓存校验和。 N 即为此配置指令的设置值。 由于此选项对于性能有较大影响，请尽在调试环境使用。

opcache.force_restart_timeout integer
    如果缓存处于非激活状态，等待多少秒之后计划重启。 如果超出了设定时间，则 OPcache 模块将杀除持有缓存锁的进程， 并进行重启。
    如果选项 opcache.log_verbosity_level 设置为 3 或者 3 以上的数值，当发生重启时将在日志中记录一条错误信息。

opcache.error_log string
    OPcache 模块的错误日志文件。 如果留空，则视为 stderr， 错误日志将被送往标准错误输出 （通常情况下是 Web 服务器的错误日志文件）。

opcache.log_verbosity_level integer
    OPcache 模块的日志级别。 默认情况下，仅有致命级别（0）及错误级别（1）的日志会被记录。 其他可用的级别有：警告（2），信息（3）和调试（4）。

opcache.preferred_memory_model string
    OPcache 首选的内存模块。 如果留空，OPcache 会选择适用的模块， 通常情况下，自动选择就可以满足需求。
    可选值包括： mmap，shm, posix 以及 win32。

opcache.protect_memory boolean
    保护共享内存，以避免执行脚本时发生非预期的写入。 仅用于内部调试。

opcache.mmap_base string
    在 Windows 平台上共享内存段的基地址。 所有的 PHP 进程都将共享内存映射到同样的地址空间。 使用此配置指令避免“无法重新附加到基地址”的错误。

opcache.restrict_api string
    仅允许路径是以指定字符串开始的 PHP 脚本调用 OPcache API 函数。 默认值为空字符串 ""，表示不做限
```

## 效果比较

可以通过 https://github.com/rlerdorf/opcache-status，来查看opCache的加速效果

![image-20210128143447455](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210128143447455.png)

说白了就是opCache可视化工具，将下载下来的项目放入到当前的Web服务器根目录下，直接访问即可

## 其他事项

1. 不建议Xcache和Opcache同时启用PHP优化；

   因为PHP 5.5.0及后续版本已经内嵌对Opcache的支持，所以PHP意识到其重要性，相对于Xcache等第三方的PHP优化器来说，使用Opcache会是更好的选择。另外，两者同时存在的话，会使Opcache的缓存命中数大大降低，而且增加不必要的开销。

2. 不建议在开发过程中开启Opcache

   原因很明显，开启了Opcache之后，开发人员修改的内容不会立即显示和生效，因为受到opcache.revalidate_freq=60的影响，所以建议在开发并测试之后，测试性能时再行打开测试，当然，生产环境一直都要开着Opcache了哦。

3. 不建议将Opcache指标设置太大

   Opcache各项指标配置大小或是否开启，需要结合项目实际情况需求及Opcache官方建议的配置，项目的实际情况分析，可结合上面第四部分的可视化缓存信息分析调整。

4. 不建议长期使用老版本的Opcache

   建议及时关注Opcache官网动态，实时了解其的bugs修复，功能优化及新增功能，以便更好的将其应用在自己的项目中。

5. 不建议在生产环境中，将opCache可视化工具放入Web服务根目录

   原因很简单，因为这个开源项目并未做访问的限制和安全处理，也就是说凡是可以访问外网的用户，只要知道了访问地址就可以直接访问，所以不安全。一般下，这个开源工具只是帮助可视化分析PHP的性能，通常在开发调试阶段使用。如果就是想在生产环境开启使用，那么就必须做好安全限制工作。