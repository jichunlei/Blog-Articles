# **一、介绍**

## **1. 简介**

* `ab`(`apache bench`的缩写)：是`Apache`超文本传输协议(HTTP)的性能测试工具。
* 其设计意图是描绘当前所安装的`Apache`的执行性能，主要是显示你安装的`Apache`每秒可以处理多少个请求。
* 不仅仅能进行基于`apache`服务器的压力测试，也能够对`nginx`、`tomacat`以及`IIS`等服务器进行访问压力测试

## **2. 原理**

* ab命令会创建**多个并发访问线程**，模拟多个访问者同时对**某一URL地址**进行访问。
* 它的测试基于URL的，因此，它既可以用来测试apache的负载压力，也可以测试`nginx`、`lighthttp`、`omcat`、`IIS`等其它Web服务器的压力。
* ab命令对发出负载的计算机要求很低，它既不会占用很高CPU，也不会占用很多内存。但却会给目标服务器造成巨大的负载。因此自己测试使用需要注意命令参数的设置。

# **二、安装**

* 先查看是否安装了ab，命令`ab -V`（注意是**大写的V**）

  ```shell
  [root@iz2zehcv8qx0688zmjq8wnz ~]# ab -V
  -bash: ab: command not found
  ```

  表示未安装

* 执行安装命令`yum -y install httpd-tools`

  ```shell
  [root@iz2zehcv8qx0688zmjq8wnz ~]# yum -y install httpd-tools
  Loaded plugins: fastestmirror
  base                                                                                                                              | 3.6 kB  00:00:00     
  epel                                                                                                                              | 5.3 kB  00:00:00     
  extras                                                                                                                            | 2.9 kB  00:00:00     
  updates                                                                                                                           | 2.9 kB  00:00:00     
  (1/4): extras/7/x86_64/primary_db                                                                                                 | 159 kB  00:00:00     
  (2/4): epel/x86_64/updateinfo                                                                                                     | 1.0 MB  00:00:00     
  (3/4): epel/x86_64/primary_db                                                                                                     | 6.9 MB  00:00:00     
  (4/4): updates/7/x86_64/primary_db                                                                                                | 6.7 MB  00:00:00     
  Determining fastest mirrors
  Resolving Dependencies
  --> Running transaction check
  ---> Package httpd-tools.x86_64 0:2.4.6-90.el7.centos will be installed
  --> Processing Dependency: libaprutil-1.so.0()(64bit) for package: httpd-tools-2.4.6-90.el7.centos.x86_64
  --> Processing Dependency: libapr-1.so.0()(64bit) for package: httpd-tools-2.4.6-90.el7.centos.x86_64
  --> Running transaction check
  ---> Package apr.x86_64 0:1.4.8-5.el7 will be installed
  ---> Package apr-util.x86_64 0:1.5.2-6.el7 will be installed
  --> Finished Dependency Resolution
  
  Dependencies Resolved
  
  =========================================================================================================================================================
   Package                              Arch                            Version                                        Repository                     Size
  =========================================================================================================================================================
  Installing:
   httpd-tools                          x86_64                          2.4.6-90.el7.centos                            base                           91 k
  Installing for dependencies:
   apr                                  x86_64                          1.4.8-5.el7                                    base                          103 k
   apr-util                             x86_64                          1.5.2-6.el7                                    base                           92 k
  
  Transaction Summary
  =========================================================================================================================================================
  Install  1 Package (+2 Dependent packages)
  
  Total download size: 286 k
  Installed size: 584 k
  Downloading packages:
  (1/3): apr-1.4.8-5.el7.x86_64.rpm                                                                                                 | 103 kB  00:00:00     
  (2/3): httpd-tools-2.4.6-90.el7.centos.x86_64.rpm                                                                                 |  91 kB  00:00:00     
  (3/3): apr-util-1.5.2-6.el7.x86_64.rpm                                                                                            |  92 kB  00:00:00     
  ---------------------------------------------------------------------------------------------------------------------------------------------------------
  Total                                                                                                                    537 kB/s | 286 kB  00:00:00     
  Running transaction check
  Running transaction test
  Transaction test succeeded
  Running transaction
  Warning: RPMDB altered outside of yum.
    Installing : apr-1.4.8-5.el7.x86_64                                                                                                                1/3 
    Installing : apr-util-1.5.2-6.el7.x86_64                                                                                                           2/3 
    Installing : httpd-tools-2.4.6-90.el7.centos.x86_64                                                                                                3/3 
    Verifying  : apr-1.4.8-5.el7.x86_64                                                                                                                1/3 
    Verifying  : httpd-tools-2.4.6-90.el7.centos.x86_64                                                                                                2/3 
    Verifying  : apr-util-1.5.2-6.el7.x86_64                                                                                                           3/3 
  
  Installed:
    httpd-tools.x86_64 0:2.4.6-90.el7.centos                                                                                                               
  
  Dependency Installed:
    apr.x86_64 0:1.4.8-5.el7                                                 apr-util.x86_64 0:1.5.2-6.el7                                                
  
  Complete!
  ```

  出现Complete!代表安装结束

* 再次执行查看命令`ab -V`

  ```shell
  [root@iz2zehcv8qx0688zmjq8wnz ~]# ab -V
  This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
  Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
  Licensed to The Apache Software Foundation, http://www.apache.org/
  ```

  安装成功

# **三、基本命令**

## **1. 参数总览**

执行命令`ab -help`可以查看参数

```tex
[root@iz2zehcv8qx0677zmjq8wnz ~]# ab -help
Usage: ab [options] [http[s]://]hostname[:port]/path
Options are:
    -n requests     Number of requests to perform
    -c concurrency  Number of multiple requests to make at a time
    -t timelimit    Seconds to max. to spend on benchmarking
                    This implies -n 50000
    -s timeout      Seconds to max. wait for each response
                    Default is 30 seconds
    -b windowsize   Size of TCP send/receive buffer, in bytes
    -B address      Address to bind to when making outgoing connections
    -p postfile     File containing data to POST. Remember also to set -T
    -u putfile      File containing data to PUT. Remember also to set -T
    -T content-type Content-type header to use for POST/PUT data, eg.
                    'application/x-www-form-urlencoded'
                    Default is 'text/plain'
    -v verbosity    How much troubleshooting info to print
    -w              Print out results in HTML tables
    -i              Use HEAD instead of GET
    -x attributes   String to insert as table attributes
    -y attributes   String to insert as tr attributes
    -z attributes   String to insert as td or th attributes
    -C attribute    Add cookie, eg. 'Apache=1234'. (repeatable)
    -H attribute    Add Arbitrary header line, eg. 'Accept-Encoding: gzip'
                    Inserted after all normal header lines. (repeatable)
    -A attribute    Add Basic WWW Authentication, the attributes
                    are a colon separated username and password.
    -P attribute    Add Basic Proxy Authentication, the attributes
                    are a colon separated username and password.
    -X proxy:port   Proxyserver and port number to use
    -V              Print version number and exit
    -k              Use HTTP KeepAlive feature
    -d              Do not show percentiles served table.
    -S              Do not show confidence estimators and warnings.
    -q              Do not show progress when doing more than 150 requests
    -g filename     Output collected data to gnuplot format file.
    -e filename     Output CSV file with percentages served
    -r              Don't exit on socket receive errors.
    -h              Display usage information (this message)
    -Z ciphersuite  Specify SSL/TLS cipher suite (See openssl ciphers)
    -f protocol     Specify SSL/TLS protocol
                    (SSL3, TLS1, TLS1.1, TLS1.2 or ALL)
```

## **2. 常用参数**

可见参数非常之多，但是绝大多数情况下我们只需了解以下几个即可

* `-n`：本次测试发起的总请求数
* `-c`：并发数
* `-p`：模拟post请求，参数为请求参数文件地址，文件格式为p1=1&p2=2，需要配合`-T`使用
* `-T`：`post`数据所使用的Content-Type头信息，例如`-T 'application/x-www-form-urlencoded'`

# **四、使用**

## **1. GET请求**

* 格式：`ab -n <总请求数> -c <并发数> <请求的url（如果有参数直接拼接在url后面即可）>`

* 例：以并发数10访问百度100次

  `ab -n 100 -c 10 https://www.baidu.com/`

## **2. POST请求**

* 格式：`ab -n <总请求数> -c <并发数> -p <post数据文件地址> -T <post数据的Content-Type头信息> <请求的url>`

* 例：同样以`post`方式访问百度（需自己创建`post.txt文件`）

  `ab -c 10 -n 100 -p '/tmp/post.txt' -T 'application/x-www-form-urlencoded' https://www.baidu.com/`

## **3. 返回信息详解**

* **以上述的GET请求为例：**

  ```
  [root@iz2zehcv8qx0688zmjq8wnz ~]# ab -n 100 -c 10 https://www.baidu.com/
  This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
  Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
  Licensed to The Apache Software Foundation, http://www.apache.org/
  
  Benchmarking www.baidu.com (be patient).....done
  
  
  Server Software:        BWS/1.1
  Server Hostname:        www.baidu.com
  Server Port:            443
  
  Document Path:          /
  Document Length:        227 bytes
  
  Concurrency Level:      10
  Time taken for tests:   0.627 seconds
  Complete requests:      100
  Failed requests:        0
  Write errors:           0
  Total transferred:      108179 bytes
  HTML transferred:       22700 bytes
  Requests per second:    159.51 [#/sec] (mean)
  Time per request:       62.692 [ms] (mean)
  Time per request:       6.269 [ms] (mean, across all concurrent requests)
  Transfer rate:          168.51 [Kbytes/sec] received
  
  Connection Times (ms)
                min  mean[+/-sd] median   max
  Connect:       18   38  20.1     39     154
  Processing:     6   15  16.6     15     171
  Waiting:        6   14  16.6     14     171
  Total:         24   53  33.8     53     325
  
  Percentage of the requests served within a certain time (ms)
    50%     53
    66%     60
    75%     62
    80%     63
    90%     67
    95%     73
    98%    123
    99%    325
   100%    325 (longest request)
  ```

* **主要返回结果说明：**
  * `Server Software`：测试服务器的名称
  * `Server Hostname`：请求的URL主机名
  * `Server Port`：服务器监听的端口
  * `Document Path`：请求的URL中的根路径
  * `Document Length`：HTTP响应数据的正文长度
  * `Concurrency Level`：并发数（我们设置的参数）
  * `Time taken for tests`：所有请求的耗时，单位秒
  * `Complete requests`：完成的总请求数（我们设置的参数）
  * `Failed requests`：失败的请求数，这里的失败是指请求在连接服务器、发送数据等环节发生异常，以及无响应后超时的情况
  * `Total transferred`：所有请求的响应数据长度总和。包括每个HTTP响应数据的头信息和正文数据的长度
  * `HTML transferred`：所有请求的响应数据中正文数据的总和，也就是减去了`Total transferred`中HTTP响应数据中的头信息的长度
  * **`Requests per second`：吞吐率**，计算公式：`Complete requests/Time taken for tests`（总请求数/处理完成这些请求数所花费的时间）（重要的指标一）
  * **第一个`Time per request`：用户平均请求等待时间**，计算公式：`Time token for tests/(Complete requests/Concurrency Level)`（处理完成所有请求数所花费的时间/（总请求数/并发用户数））（重要的指标）
  * 第二个`Time per request`：服务器平均请求等待时间，计算公式：`Time taken for tests/Complete requests`（吞吐率的倒数）；也可以这么统计：`Time per request/Concurrency Level`
  * `Transfer rate`：平均每秒网络上的流量，可以帮助排除是否存在网络流量过大导致响应时间延长的问题
  * `Percentage of the requests served within a certain time (ms)`：百分之多少的用户请求在多少毫秒内返回

# **参考**

* [Linux下ab压力测试](https://blog.csdn.net/wangqingchuan92/article/details/79740060)
* [Linux下 安装ab测试工具以及使用](https://blog.csdn.net/feiwutudou/article/details/80334099)