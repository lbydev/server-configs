# PHP-FPM一些配置建议

**以下主要根据Josh Lockhart 《Modern PHP》整理**

###配置
PHP-FPM，管理PHP进程池的软件。

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

# PHP.ini

### 内存
php.ini文件中的`memory_limit`，用于设置单个PHP进程可以使用的系统内存最大值。
这个默认值是128M，对于中小型PHP应用来说或许合适。如果是运行微型PHP应用，可以降低这个值，例如设为64M，节省系统资源。
如果运行的是内存集中性PHP应用（例如使用Drupal搭建的网站）,可以增加这个值，例如设为512M，提升性能。
当然这个值还需要根据系统可以分配给PHP多少内存，以及能负担起多少个PHP-FPM进程，单个PHP进程平均需要消耗多少内存。

### Zend OPcache
Modern-PHP 作者在`php.ini`文件中配置和优化Zend OPcache扩展所用的设置：
```

opcache.memory_consumption = 64
# 为操作码分配的内存量
opcache.interned_string_buffer = 16
# 用来存储驻留字符串
opcache.max_accelerated_files = 4000
#操作码缓存中最多能存储多少个PHP脚本。一定要比PHP应用中的文件数量大
opcache.validate_timestaps = 1
# 设置为1，经过一段时间后PHP会检查PHP脚本的内容是否有变化，检查的时间间隔由opcache.revalidate_freq设置指定。
# 设置为0，PHP不会检查PHP脚本的内容是否有变化，我们必须手动清除缓存的操作码。
# 建议开发环境设置为1，在生产环境中设为0
opcache.revalidate_freq = 0
#设置PHP多久（单马为秒）检查一次PHP脚本的内容是否有变化。如果有变化，PHP会重新编译脚本，再次缓存。
# 建议开发环境设置为0，这样opcache.validate_timestaps设置为1的时候，每次请求都会重新验证PHP文件
#（这对于开发环境来说是好事）
# 对于生产环境中opcache.validate_timestaps设置为0，这个选项对于生产环境意义不大
opcache.fast_shutdown=1
# 这个设置能让操作码使用更快的停机步骤，把对象析构和内存释放交给Zend Engine的内存管理器完成。
# 这个设置缺少文档，设置为1即可。

```

### 最长执行时间
php.ini的`max_execution_time`设置单个PHP进程在终止之前可以运行多少时间。默认值是30s。我们不需要让PHP进程执行这么长，
因为我们想让应用运行得特别快。建议设置改为5s
`max_execution_time = 5`

如果一些特殊场景需要PHP脚本运行更长的时间怎么办？答案是，PHP脚本不能长时间运行，PHP运行的时间越长，Web应用的访问者等待响应的时间就会越长。
如果有长时间运行的任务（例如，调整图像尺寸或生成报告），要在单独的进程中运行。

可以使用PHP中的`exec()`函数调用`bash`的`at`命令。这个命令的作用是派生单独的非阻塞进程，不耽误当前的PHP进程。使用PHP中的`exec()`函数要使用
`escapeshellarg()`函数转义shell参数

假设要生成报告，并将结果制作成PDF文件。这个任务可能需要花10分钟才能完成，而我们肯定不想让PHP请求等待10分钟。
可以单独编写一个PHP文件，假如命名为create-report.php，让这个文件运行10分钟，最后生成报告。其实Web应用只需要毫秒就可以派生一个单独的后台进程，
然后返回HTTP相应，如下所示：
```
<?php
exec('echo "create-report.php" | at now');
echo 'Report pending...'

```
create-report.php脚本在单独的后台进程中运行，运行完毕后可以更新数据库，或者通过电子邮件把报告发给收件人。
我们完全没有必要让长时间运行的任务拖延PHP主脚本，影响用户体验。
> 如果发现自己派生了很多后台进程，或许最好使用专门的作业队列。PHPResque（https://github.com/chrisboulton/php-resque）是个非常不错的作业队列管理器，
> 它是基于Github的作业队列管理器Resque（https://github.com/blog/542-introducing-resque）开发的

### 真实路径缓存
PHP会缓存应用使用的文件路径，这样每次包含或导入文件时就无需不断搜索包含路径了。这个缓存就真是路径缓存（realpath cache）。
如果运行的是大型PHP文件（Drupal和Composer组件等），使用了大量文件，增加PHP真实路径缓存的大小都能得到更好的性能。

默认值是16K，不过这个缓存的准确大小不容易确定，这里有个小技巧。首先，增加真实路径缓存的大小，设为特别大的值，例如256k。
然后再一个PHP脚本的末尾加上`print_r(realpath_cache_size());`，输出真实路径缓存真正的大小。最后，将真是路径缓存的大小改为这个真正的值。
例如：`realpath_cache_size = 64k`
