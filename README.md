# server-configs
Nginx HTTP server configs for local vagrant deploy


###PHP-FPM配置
**全局配置**

Ubuntu中，PHP-FPM主配置文件是`/etc/php5/fpm/php-fpm.conf`
Centos中，PHP-FPM的主配置文件是`/etc/php-fpm.conf`

有两个比较重要的全局设置，主要是在指定的一段时间内有指定个子进程失效，让PHP-FPM主进程重启。
这是PHP-FPM进程的基本安全保障，能解决简单的问题。
```
emergency_restart_threshold = 10 
# 在指定的一段时间内，如果失效的PHP-FPM子进程超过这个值，PHP-FPN主进程就会优雅重启。
# 0 表示“关闭该功能”。默认值：0（关闭）。

emergency_restart_interval =1m
# 设定emergency_restart_threshold设置采用的时间跨度。
# 可用单位：s（秒），m（分），h（小时）或者 d（天）。默认单位：s（秒）。默认值：0（关闭）

```
> PHP-FPM全局设置的详细信息：http://php.net/manual/zh/install.fpm.configuration.php

**配置进程池**
Pool Definitions区域，这个区域的配置用于设置每个PHP-FPM进程池。PHP-FPM进程池中是一系列相关的php子进程。
通常一个PHP应用有自己的一个PHP-FPM进程池。

Ubuntu中，Pool Definitions区域只有下面一行内容：
`include=/etc/php5/fpm/pool.d/*.conf`
Centos中，则在PHP-FPM主配置文件的顶部使用下面这行代码引入进程池定义文件：
`include=/etc/php-fpm.d/*.conf`

各个PHP-FPM进程池都已指定的操作系统用户和用户组的身份运行.
建议以单独的非root用户身份运行各个PHP-FPM进程池，这样在命令中使用`top`或`ps aux`命令时便于识别每个PHP应用的PHP-FPM进程池。
这是个好习惯，因为每个PHP-FPM进程池中的进程都受相应的操作系统用户和用户组的权限限制在沙盒中。
打开`www.conf`文件进行配置：
```
user = deploy
# 拥有这个PHP-FPM进程池中子进程的系统用户

group = deploy
# 拥有这个PHP-FPM进程池中子进程的系统用户组

listen = 127.0.0.1：9000
# PHP-FPM进程池监听的IP地址和端口号，让PHP-FPM只接受nginx从这里传入的请求

listen.allowed_clients = 127.0.0.1
# 可以给这个PHP-FPM进程池发送请求的IP地址。

pm.max_chindren = 51
# 这个设置设定任何时间点PHP-FPM进程池中最多能有多少个进程。这个需要测试自己的PHP应用，
# 确定每个PHP进程需要使用多少内存，然后把这个设置设为设备可用内存能容纳的PHP进程总数。
# 对于大多数中小型PHP应用来说，每个PHP进程要使用5~15M内存（具体用量可能有差异）
 
pm.start_servers = 3
# PHP-FPM启动时PHP-FPM进程池中立即可用的进程数。一个中小型PHP应用，建议可设置为2或3
# 这么做是为了先准备好两到三个进程，等待请求进入，不让PHP应用的头几个HTTP请求等待PHP-PFM储池华进程池中的进程

pm.min_spare_servers = 2 
# PHP应用空闲时PHP-FPM进程池中可以存在的进程数最小值。可同上

pm.max_spare_servers = 4
# PHP应用空闲时PHP-FPM进程池中可以存在的进程数最大值。可同上

pm.max_requests = 1000
# 回收进程之前，PHP-FPM进程池中各个进程最多能处理的HTTP请求数量。
# 这个设置有助于避免PHP扩展或库因编写拙劣而导致不断泄露内存。建议设置为1000，也可根据应用需要做调整

slowlog = /path/to/slowlog.log
# 这个日志文件用于记录处理时间超过n秒的HTTP请求信息，以便找出PHP应用的瓶颈，进行调试。
# ** PHP-FPM进程池所属的用户和用户组必须有这个文件的写权限**

request_slowlog_timeout = 5s
# 当前HTTP请求的处理时间超过指定的值，就把请求的回溯信息写入slowlog设置指定的日志文件。
# 一开始可以设为5s
```

编辑并保存PHP-FPM的配置文件后，要执行下述命令重启PHP-FPM主进程

Ubuntu
`sudo service php5-fpm restart`

Centos
`sudo systemctl restart php-fpm.service`

> PHP-FPM进程池的配置详情参见：http://php.net/manual/zh/install.fpm.configuration.php
