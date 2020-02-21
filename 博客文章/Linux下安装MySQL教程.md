# 一、下载

* **下载地址**：[官网下载地址](https://downloads.mysql.com/archives/community/)

* 以**5.7.28**版本为例

  ![](http://img.xianzilei.cn/Linux%E5%AE%89%E8%A3%85MySQL%E7%94%A8%E4%BE%8B%E5%9B%BE-01.png)

# 二、环境准备

* **检测系统是否已经安装了MySQL**

  ```shell
  rpm -qa|grep mysql
  ```

  执行上述命令如果出现类似如下情况

  ```shell
  mysql-libs-5.1.52-1.el6_0.1.x86_64
  ```

  则说明系统已经安装了MySQL，可以选择进行卸载

  ```shell
  # 普通删除模式
  rpm -e mysql-libs-5.1.52-1.el6_0.1.x86_64
  
  # 强力删除模式，如果使用上面普通删除命令删除时，提示有依赖的其它文件，则用该命令可以对其进行强力删除
  rpm -e --nodeps mysql-libs-5.1.52-1.el6_0.1.x86_64　　
  ```

* **检测系统是否存在`mariadb` 数据库**

  ```shell
  rpm -qa | grep mariadb
  ```

  执行上述命令如果出现类似如下情况

  ```shell
  mariadb-libs-5.5.56-2.el7.x86_64
  ```

  则说明系统存在`mariadb` 数据库，可以选择进行卸载

  ```shell
  rpm -e --nodeps mariadb-libs-5.5.56-2.el7.x86_64
  ```

# 三、安装启动

* **1）将下载的安装包上传至服务器上**（我是放在`/opt/mysql/`目录下）

  ```shell
  [root@localhost mysql]# cd /opt/mysql/
  [root@localhost mysql]# ll
  总用量 707688
  -rw-r--r--. 1 root root 724672294 2月  21 09:38 mysql-5.7.28-linux-glibc2.12-x86_64.tar.gz
  ```

* **2）解压至当前文件夹**

  ```shell
  [root@localhost mysql]# tar -zxvf mysql-5.7.28-linux-glibc2.12-x86_64.tar.gz 
  mysql-5.7.28-linux-glibc2.12-x86_64/bin/myisam_ftdump
  mysql-5.7.28-linux-glibc2.12-x86_64/bin/myisamchk
  mysql-5.7.28-linux-glibc2.12-x86_64/bin/myisamlog
  mysql-5.7.28-linux-glibc2.12-x86_64/bin/myisampack
  mysql-5.7.28-linux-glibc2.12-x86_64/bin/mysql
  mysql-5.7.28-linux-glibc2.12-x86_64/bin/mysql_client_test_embedded
  ......
  ```

* **3）重命名解压后的文件夹**

  ```shell
  [root@localhost mysql]# mv mysql-5.7.28-linux-glibc2.12-x86_64 mysql-5.7.28
  ```

* **4）添加系统`mysql`组和`mysql`用户**

  * 先检查是否存在

    ```shell
    # 检查是否存在mysql组
    cat /etc/group | grep mysql
    
    #检查是否存在mysql用户
    cat /etc/passwd | grep mysql
    ```

  * 如果分别出现类似下面情况

    ```shell
    #类似（检查是否存在mysql组）
    mysql:x:490:
    
    #类似（检查是否存在mysql用户）
    mysql:x:496:490::/home/mysql:/bin/bash
    ```

    则说明已经存在，无须执行添加命令

  * 否则执行添加命令

    ```shell
    # 添加mysql组
    groupadd mysql
    # 添加mysql用户，-r参数表示mysql用户是系统用户，不可用于登录系统
    useradd -r -g mysql mysql
    ```

  * 检查是否添加成功

    ```shell
    [root@localhost mysql]# cat /etc/group | grep mysql
    mysql:x:1001:
    [root@localhost mysql]# cat /etc/passwd | grep mysql
    mysql:x:998:1001::/home/mysql:/bin/bash
    ```

    **添加成功！**

* **5）mysql安装目录下创建data目录**

  ```shell
  [root@localhost mysql]# cd mysql-5.7.28
  [root@localhost mysql-5.7.28]# mkdir data
  ```

* **6）将`/opt/mysql/mysql-5.7.28`的所有者及所属组改为mysql**

  ```shell
  [root@localhost mysql-5.7.28]# chown -R mysql.mysql /opt/mysql/mysql-5.7.28
  ```

* **7）修改/etc/my.cnf文件**

  ```pro
  [mysql]
  # 设置mysql客户端默认字符集
  default-character-set=utf8 
  [mysqld]
  skip-name-resolve
  #设置3306端口
  port = 3306 
  # 设置mysql的安装目录
  basedir=/opt/mysql/mysql-5.7.28
  # 设置mysql数据库的数据的存放目录
  datadir=/opt/mysql/mysql-5.7.28/data
  # 允许最大连接数
  max_connections=200
  # 服务端使用的字符集默认为8比特编码的latin1字符集
  character-set-server=utf8
  # 创建新表时将使用的默认存储引擎
  default-storage-engine=INNODB 
  lower_case_table_names=1
  max_allowed_packet=16M
  ```

* **8）初始化mysqld**

  ```shell
  [root@localhost etc]# cd /opt/mysql/mysql-5.7.28/bin/
  [root@localhost bin]# ./mysqld --initialize --user=mysql --basedir=/opt/mysql/mysql-5.7.28/ --datadir=/opt/mysql/mysql-5.7.28/data/ 
  2020-02-21T10:27:02.302935Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
  2020-02-21T10:27:02.598899Z 0 [Warning] InnoDB: New log files created, LSN=45790
  2020-02-21T10:27:02.728734Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
  2020-02-21T10:27:02.824855Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: b34c7b66-5494-11ea-a2f2-000c293e6164.
  2020-02-21T10:27:02.825963Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
  2020-02-21T10:27:03.625258Z 0 [Warning] CA certificate ca.pem is self signed.
  2020-02-21T10:27:03.961315Z 1 [Note] A temporary password is generated for root@localhost: (vjwMKe+q8.r
  ```

  **其中初始密码为(vjwMKe+q8.r**

* **9）把启动脚本放到开机初始化目录**

  ```shell
  [root@localhost mysql-5.7.28]# cd /opt/mysql/mysql-5.7.28
  [root@localhost mysql-5.7.28]# cp support-files/mysql.server /etc/init.d/mysql
  ```

* **10）启动mysql服务**

  ```shell
  [root@localhost bin]# service mysql start
  Starting MySQL.Logging to '/opt/mysql/mysql-5.7.28/data/localhost.localdomain.err'.
   SUCCESS! 
  ```

  **启动成功！**

* **11）登录mysql，密码为上述的初始密码**

  ```shell
  [root@localhost bin]# cd /opt/mysql/mysql-5.7.28/bin/
  [root@localhost bin]# ./mysql -u root -p
  Enter password: 
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 3
  Server version: 5.7.28
  
  Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.
  
  Oracle is a registered trademark of Oracle Corporation and/or its
  affiliates. Other names may be trademarks of their respective
  owners.
  
  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
  
  mysql> 
  ```

* **12）修改密码**

  ```shell
  mysql> set password=password('root');
  Query OK, 0 rows affected, 1 warning (0.01 sec)
  
  mysql> flush privileges;
  Query OK, 0 rows affected (0.01 sec)
  ```
  
* **13）添加远程访问权限**

  ```shell
  mysql> use mysql
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A
  
  Database changed
  mysql> update user set host='%' where user = 'root';
  Query OK, 1 row affected (0.00 sec)
  Rows matched: 1  Changed: 1  Warnings: 0
  
  mysql> flush privileges;
  Query OK, 0 rows affected (0.00 sec)
  ```

* **14）重启mysql**

  ```shell
  [root@localhost bin]# service mysql stop
  Shutting down MySQL.. SUCCESS! 
  [root@localhost bin]# service mysql start
  Starting MySQL. SUCCESS!
  ```

* **15）Navicat连接测试**

  ![](http://img.xianzilei.cn/Linux%E5%AE%89%E8%A3%85MySQL%E7%94%A8%E4%BE%8B%E5%9B%BE-02.png)

  **连接成功！**