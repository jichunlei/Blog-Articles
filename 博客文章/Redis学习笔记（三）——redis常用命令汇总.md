# **一、常用管理命令**

## **1. 启动Redis**

```shell
redis-server [--port 6379]
```

如果参数过多，可以通过配置文件来启动Redis

```shell
redis-server [xx/xx/redis.conf]
```

## **2. 连接Redis**

```shell
./redis-cli [-h 127.0.0.1 -p 6379 -a 你的密码]
```

## **3. 停止Redis**

```shell
redis-cli shutdown
```

```shell
kill redis-pid
```

以上两句效果一样。

## **4. 测试Redis连通性**

```shell
127.0.0.1:6379> PING
PONG
```

返回PONG则表示连接正常

## **5. 查看Redis版本信息**

```shell
redis-server -v
```

```shell
redis-cli --version
```

# **二、key操作命令**

## **1. 获取所有键**

* **语法**：`keys pattern`

* **pattern表示为匹配模式**

* **例**：

  ```shell
  127.0.0.1:6379> keys *
  1) "k3"
  2) "k5"
  3) "k2"
  4) "list01"
  5) "k4"
  ```

  表示**通配符**，表示任意字符，会遍历所有键显示所有的键列表，时间复杂度O(n)，在生产环境不建议使用。

## **2. 获取键总数**

* **语法**：`dbsize`

* 获取**键总数**时不会遍历所有的键，直接获取内部变量，时间复杂度O(1)

* **例**：

  ```shell
  127.0.0.1:6379> dbsize
  (integer) 5
  ```

## 3.查询键是否存在

* **语法**：`exists key [key ...]`

* 查询查询多个，返回存在的个数

* **例**：

  ```shell
  127.0.0.1:6379> EXISTS k1
  (integer) 0
  127.0.0.1:6379> EXISTS k2
  (integer) 1
  127.0.0.1:6379> EXISTS k2 k1
  (integer) 1
  127.0.0.1:6379> EXISTS k2 k1 k2 k3
  (integer) 3
  ```

## 4.删除键

* **语法**：`del key [key ...]`

* 可以**删除多个**，**返回删除成功的个数**。

* 例：

  ```shell
  127.0.0.1:6379> DEL k1
  (integer) 0
  127.0.0.1:6379> DEL k2
  (integer) 1
  127.0.0.1:6379> DEL k1 k2 k3 k4 k5
  (integer) 3
  ```

## 5.查询键类型

* 语法：type key

* 例：

  ```shell
  127.0.0.1:6379> type k1
  string
  127.0.0.1:6379> type list01
  list
  ```

## 6.移动键

* 语法：move key db

* 例：把k2键移动到2号库

  ```shell
  127.0.0.1:6379> get k2
  "v2"
  127.0.0.1:6379> move k2 2
  (integer) 1
  127.0.0.1:6379> select 2
  OK
  127.0.0.1:6379[2]> get k2
  "v2"
  127.0.0.1:6379[2]> select 0
  OK
  127.0.0.1:6379> get k2
  (nil)
  ```

## 7.查询key的生命周期（秒）

* 语法
  * 秒：ttl key
  * 毫秒：pttl key

* 返回说明（Redis2.8+）
  * 如果key不存在或者已过期，返回 -2
  * 如果key存在并且没有设置过期时间（永久有效），返回 -1

* 例：

  ```shell
  127.0.0.1:6379> ttl k1
  (integer) 54
  127.0.0.1:6379> ttl k2
  (integer) -2
  127.0.0.1:6379> ttl k3
  (integer) -1
  127.0.0.1:6379> pttl k1
  (integer) 38747
  127.0.0.1:6379> pttl k2
  (integer) -2
  127.0.0.1:6379> pttl k3
  (integer) -1
  ```

## 8.设置过期时间

* 语法：
  * 秒：expire key seconds
  * 毫秒：pexpire key milliseconds

* 例：

  ```shell
  127.0.0.1:6379> expire k1 60
  (integer) 1
  127.0.0.1:6379> pexpire k1 60000
  (integer) 1
  ```

## 9.设置永不过期

* 语法：persist key

* 例：

  ```shell
  127.0.0.1:6379> PERSIST k3
  (integer) 1
  127.0.0.1:6379> ttl k3
  (integer) -1
  ```

## 10.更改键名称

* 语法：rename key newkey

* 例：

  ```shell
  127.0.0.1:6379> RENAME k3 kk3
  OK
  127.0.0.1:6379> get kk3
  "v3"
  ```


# 三、字符串操作命令

## 1.存放键值

* 语法：

  * set key value [EX seconds] [PX milliseconds] [NX|XX]
  * setex key seconds value：设置带过期时间的key
  * setnx key value：只有在 key 不存在时设置 key 的值

* 单个数据能存储的最大空间为512M

* 可选参数不填写表示永久不失效

* 例：设置k1键值为v1，有效时间为20s

  ```shell
  127.0.0.1:6379> set k1 v1 EX 20
  OK
  127.0.0.1:6379> setex k1 10 v1
  OK
  127.0.0.1:6379> setnx k1 v1
  (integer) 1
  127.0.0.1:6379> setnx k1 v1
  (integer) 0
  ```

## 2.获取键值

* 语法：

  * get key
  * getset key value：先get后set，返回旧值

* 例：

  ```shell
  127.0.0.1:6379> get k1
  "v1"
  127.0.0.1:6379> getset k1 v2
  "v1"
  127.0.0.1:6379> get k1
  "v2"
  ```

## 3.值递增/递减

* 语法：
  * 递增1：incr key
  * 递减1：decr key
  * 递增n：incrby key n
  * 递减n：decrby key n

* 字符串中的值必须是数字类型，否则会报错

* 例：

  ```shell
  127.0.0.1:6379> set k2 1
  OK
  127.0.0.1:6379> INCR k2 
  (integer) 2
  127.0.0.1:6379> INCRBY k2 3
  (integer) 5
  127.0.0.1:6379> DECR k2
  (integer) 4
  127.0.0.1:6379> DECRBY k2 3
  (integer) 1
  127.0.0.1:6379> set k1 v1
  OK
  127.0.0.1:6379> INCR k1
  (error) ERR value is not an integer or out of range
  ```

## 4.批量存放键值

* 语法：

  * mset key value [key value ...]
  * msetnx key value [key value ...] ：仅当所有的key都不存在才能存放成功，否则全部失败

* 例：

  ```powershell
  127.0.0.1:6379> mset k1 v1 k2 v2 k3 v3
  OK
  127.0.0.1:6379> msetnx k4 v4 k1 v2
  (integer) 0
  ```

## 5.批量获取键值

* 语法：mget key [key ...]

* 例：

  ```shell
  127.0.0.1:6379> mget k1 k2 k3
  1) "v1"
  2) "v2"
  3) "v3"
  ```

## 6.获取键长度

* 语法：strlen key

* Redis接收的是UTF-8的编码，如果是中文一个汉字将占3位返回

* 例

  ```shell
  127.0.0.1:6379> set k4 哈哈
  OK
  127.0.0.1:6379> get k4
  "\xe5\x93\x88\xe5\x93\x88"
  127.0.0.1:6379> STRLEN k4
  (integer) 6
  ```

## 7.追加值的内容

* 语法：append key value

* 键值尾部添加

* 例：

  ```shell
  127.0.0.1:6379> get k1
  "v1"
  127.0.0.1:6379> APPEND k1 23456
  (integer) 7
  127.0.0.1:6379> get k1
  "v123456"
  ```

## 8.获取部分字符

* 语法：getrange key start end

* 类似between......and

* getrange key 0 -1：表示获取全部字符

* 例：

  ```shell
  127.0.0.1:6379> get k1
  "v123456"
  127.0.0.1:6379> GETRANGE k1 2 4
  "234"
  127.0.0.1:6379> GETRANGE k1 0 -1
  "v123456"
  ```

## 9.设置指定位置的值

* 语法：getrange key offset value

* 会覆盖后面的值

* 举例

  ```shell
  127.0.0.1:6379> get k1
  "v123456"
  127.0.0.1:6379> SETRANGE k1 1 aaa
  (integer) 7
  127.0.0.1:6379> get k1
  "vaaa456"
  ```

# 四、列表操作命令

## 1.插入值

* 语法：

  * 左插入：lpush key value [value ...]
  * 右插入：rpush key value [value ...]

* 例：

  ```shell
  127.0.0.1:6379> LPUSH k1 1 2 3 4 5 6 7 x x s f s w f s w
  (integer) 16
  127.0.0.1:6379> LRANGE k1 0 -1
  1) "9"
  2) "8"
  3) "7"
  4) "6"
  5) "5"
  6) "4"
  7) "3"
  8) "2"
  9) "1"
  127.0.0.1:6379> RPUSH k2 1 2 3 4 5 6 7 8 9
  (integer) 9
  127.0.0.1:6379> LRANGE k2 0 -1
  1) "1"
  2) "2"
  3) "3"
  4) "4"
  5) "5"
  6) "6"
  7) "7"
  8) "8"
  9) "9"
  ```

## 2.获取指定范围的元素

* 语法：LRANGE key start stop

* start 和 end 偏移量都是基于0的下标，即list的第一个元素下标是0（list的表头），第二个元素下标是1，以此类推。

* 偏移量也可以是负数，表示偏移量是从list尾部开始计数。 例如， -1 表示列表的最后一个元素，-2 是倒数第二个，以此类推

* 当下标超过list范围的时候不会产生error。 如果start比list的尾部下标大的时候，会返回一个空列表。 如果stop比list的实际尾部大的时候，Redis会当它是最后一个元素的下标

* 例：

  ```shell
  127.0.0.1:6379> LRANGE k3 0 -1
  1) "9"
  2) "8"
  3) "7"
  4) "6"
  5) "5"
  6) "4"
  7) "3"
  8) "2"
  9) "1"
  ```

## 3.移除元素

* 语法
  * 左移除key对应的list的第一个元素：LPOP key
  * 右移除key对应的list的第一个元素：RPOP key
* 

* key 不存在时返回 nil

* 例：

  ```shell
  127.0.0.1:6379> LPOP k1
  "1"
  127.0.0.1:6379> RPOP k1
  "9"
  127.0.0.1:6379> LPOP kk
  (nil)
  ```

## 4.根据索引获取元素

* 语法：LINDEX key index

* 下标是从0开始索引的，所以 0 是表示第一个元素， 1 表示第二个元素，并以此类推。 

* 负数索引用于指定从列表尾部开始索引的元素。在这种方法下，-1 表示最后一个元素，-2 表示倒数第二个元素，并以此往前推。

* key不存在时返回nil

* 当 key 位置的值不是一个列表的时候，会返回一个error

* 例：

  ```shell
  127.0.0.1:6379> LINDEX k1 0
  "2"
  127.0.0.1:6379> LINDEX k1 1
  "3"
  127.0.0.1:6379> LINDEX k1 -1
  "8"
  127.0.0.1:6379> LINDEX k1 -2
  "7"
  127.0.0.1:6379> LINDEX kk 0
  (nil)
  127.0.0.1:6379> set k4 v4
  OK
  127.0.0.1:6379> LINDEX k4 0
  (error) WRONGTYPE Operation against a key holding the wrong kind of value
  ```

## 5.获取列表长度

* 语法：LLEN key

* 如果 key 不存在，那么就被看作是空list，并且返回长度为 0。 

* 当存储在 key 里的值不是一个list的话，会返回error

* 例：

  ```shell
  127.0.0.1:6379> LLEN k1
  (integer) 7
  127.0.0.1:6379> LLEN kk
  (integer) 0
  127.0.0.1:6379> LLEN k4
  (error) WRONGTYPE Operation against a key holding the wrong kind of value
  ```

## 6.删除列表元素

* 语法：LREM key count value

* 从存于 key 的列表里移除前 count 次出现的值为 value 的元素

* count > 0: 从头往尾移除值为 value 的元素。

* count < 0: 从尾往头移除值为 value 的元素。

* count = 0: 移除所有值为 value 的元素。

* 例：

  ```shell
  127.0.0.1:6379> LRANGE k1 0 -1
  1) "2"
  2) "2"
  3) "2"
  4) "2"
  5) "3"
  6) "4"
  7) "5"
  8) "7"
  9) "8"
  127.0.0.1:6379> LREM k1 -3 2
  (integer) 3
  127.0.0.1:6379> LRANGE k1 0 -1
  1) "2"
  2) "3"
  3) "4"
  4) "5"
  5) "7"
  6) "8"
  ```

## 7.修剪列表

* 语法：LTRIM key start stop
* 修剪(trim)一个已存在的 list，这样 list 就会只包含指定范围的指定元素
* start 和 end 也可以用负数来表示与表尾的偏移量，比如 -1 表示列表里的最后一个元素， -2 表示倒数第二个
* 超过范围的下标并不会产生错误：
  * 如果 start 超过列表尾部，或者 start > end，结果会是列表变成空表（即该 key 会被移除）。 
  * 如果 end 超过列表尾部，Redis 会将其当作列表的最后一个元素。

* 例：

  ```shell
  127.0.0.1:6379> LRANGE k1 0 -1
  1) "2"
  2) "3"
  3) "4"
  4) "5"
  5) "7"
  6) "8"
  127.0.0.1:6379> LTRIM k1 0 -1
  OK
  127.0.0.1:6379> LRANGE k1 0 -2
  1) "2"
  2) "3"
  3) "4"
  4) "5"
  5) "7"
  ```

## 8.双列表操作

* 语法：RPOPLPUSH source destination

* **原子性**地返回并移除存储在 source 的列表的最后一个元素（列表尾部元素）， 并把该元素放入存储在 destination 的列表的第一个元素位置（列表头部）。

* 如果 source 不存在，那么会返回 nil 值，并且不会执行任何操作。 如果 source 和 destination 是同样的，那么这个操作等同于移除列表最后一个元素并且把该元素放在列表头部， 所以这个命令也可以当作是一个**旋转**列表的命令。

* 例：

  ```shell
  127.0.0.1:6379> LRANGE k1 0 -1
  1) "1"
  2) "2"
  3) "3"
  4) "4"
  5) "5"
  127.0.0.1:6379> LRANGE k2 0 -1
  1) "a"
  2) "b"
  3) "c"
  4) "d"
  5) "e"
  127.0.0.1:6379> RPOPLPUSH k1 k2
  "5"
  127.0.0.1:6379> LRANGE k1 0 -1
  1) "1"
  2) "2"
  3) "3"
  4) "4"
  127.0.0.1:6379> LRANGE k2 0 -1
  1) "5"
  2) "a"
  3) "b"
  4) "c"
  5) "d"
  6) "e"
  ```

## 9.修改列表指定位置值

* 语法：LSET key index value

* 设置 index 位置的list元素的值为 value

* 当index超出范围时会返回一个error

* 例：

  ```shell
  127.0.0.1:6379> LRANGE k1 0 -1
  1) "1"
  2) "2"
  3) "3"
  4) "4"
  127.0.0.1:6379> LSET k1 0 a
  OK
  127.0.0.1:6379> LRANGE k1 0 -1
  1) "a"
  2) "2"
  3) "3"
  4) "4"
  127.0.0.1:6379> LSET k1 5 b
  (error) ERR index out of range
  ```

## 10.列表插入元素

* 语法：LINSERT key BEFORE|AFTER pivot value

* 把 value 插入存于 key 的列表中在基准值 pivot 的前面或后面。

* 当 key 不存在时，这个list会被看作是空list，任何操作都不会发生。

* 当 key 存在，但保存的不是一个list的时候，会返回error。

* 例：

  ```shell
  127.0.0.1:6379> LRANGE k1 0 -1
  1) "a"
  2) "2"
  3) "3"
  4) "4"
  5) "3"
  6) "4"
  127.0.0.1:6379> LINSERT k1 before 3 x
  (integer) 7
  127.0.0.1:6379> LRANGE k1 0 -1
  1) "a"
  2) "2"
  3) "x"
  4) "3"
  5) "4"
  6) "3"
  7) "4"
  127.0.0.1:6379> LINSERT k1 after 3 x
  (integer) 8
  127.0.0.1:6379> LRANGE k1 0 -1
  1) "a"
  2) "2"
  3) "x"
  4) "3"
  5) "x"
  6) "4"
  7) "3"
  8) "4"
  ```

# 五、集合操作命令

## 1.添加元素

* 语法：SADD key member [member ...]

* 添加一个或多个指定的member元素到集合的 key中.指定的一个或者多个元素member 

* 如果已经在集合key中存在则忽略.如果集合key 不存在，则新建集合key,并添加member元素到集合key中.

* 如果key 的类型不是集合则返回错误.

* 例：

  ```shell
  127.0.0.1:6379> SADD k1 a b c c d d e e f g g
  (integer) 7
  127.0.0.1:6379> SMEMBERS k1
  1) "e"
  2) "f"
  3) "b"
  4) "d"
  5) "g"
  6) "a"
  7) "c"
  ```

## 2.返回集合所有元素

* 语法：SMEMBERS key

* 例：

  ```shell
  127.0.0.1:6379> SMEMBERS k1
  1) "e"
  2) "f"
  3) "b"
  4) "d"
  5) "g"
  6) "a"
  7) "c"
  ```

## 3.判断元素是否存在

* 语法：SISMEMBER key member
* 返回值
  * 如果member元素是集合key的成员，则返回1
  * 如果member元素不是key的成员，或者集合key不存在，则返回0

* 例：

  ```shell
  127.0.0.1:6379> SISMEMBER k1 e
  (integer) 1
  127.0.0.1:6379> SISMEMBER k1 aaa
  (integer) 0
  127.0.0.1:6379> SISMEMBER kk aaa
  (integer) 0
  ```

## 4.获取集合元素数量

* 语法：SCARD key

* 例：

  ```shell
  127.0.0.1:6379> SCARD k1
  (integer) 7
  127.0.0.1:6379> SCARD kk
  (integer) 0
  ```

## 5.删除集合元素

* 语法：SREM key member [member ...]

* 在key集合中移除指定的元素. 如果指定的元素不是key集合中的元素则忽略

*  如果key集合不存在则被视为一个空的集合，该命令返回0.

* 如果key的类型不是一个集合,则返回错误.

* 例：

  ```shell
  127.0.0.1:6379> SREM k1 a
  (integer) 1
  127.0.0.1:6379> SREM k1 aa
  (integer) 0
  127.0.0.1:6379> SREM kk aa
  (integer) 0
  ```

## 6.随机返回集合中一个或多个元素

* 语法：SRANDMEMBER key [count]

* Redis 2.6开始，可以接受 count 参数，

  * 如果count是整数且小于元素的个数，返回含有 count 个不同的元素的数组，
  * 如果count是个整数且大于集合中元素的个数时，仅返回整个集合的所有元素，
  * 当count是负数，则会返回一个包含count的绝对值的个数元素的数组，如果count的绝对值大于元素的个数，则返回的结果集里会出现一个元素出现多次的情况.

* 仅提供key参数时，该命令作用类似于SPOP命令，不同的是SPOP命令会将被选择的随机元素从集合中移除，而SRANDMEMBER仅仅是返回该随记元素，而不做任何操作.

* 例：

  ```shell
  127.0.0.1:6379> SRANDMEMBER k1
  "c"
  127.0.0.1:6379> SRANDMEMBER k1
  "g"
  127.0.0.1:6379> SRANDMEMBER k1
  "b"
  127.0.0.1:6379> SRANDMEMBER k1 3
  1) "f"
  2) "g"
  3) "c"
  127.0.0.1:6379> SRANDMEMBER k1 3
  1) "f"
  2) "d"
  3) "g"
  127.0.0.1:6379> SRANDMEMBER k1 -2
  1) "c"
  2) "c"
  127.0.0.1:6379> SRANDMEMBER k1 -2
  1) "c"
  2) "b"
  127.0.0.1:6379> SRANDMEMBER k1 9
  1) "d"
  2) "b"
  3) "f"
  4) "e"
  5) "g"
  6) "c"
  127.0.0.1:6379> SRANDMEMBER k1 -9
  1) "b"
  2) "c"
  3) "d"
  4) "g"
  5) "g"
  6) "c"
  7) "c"
  8) "c"
  9) "e"
  ```

## 7.随机移除集合中一个或多个元素

* 语法：SPOP key [count]
* 从存储在`key`的集合中移除并返回一个或多个随机元素。
* 此操作与`SRANDMEMBER`类似，它从一个集合中返回一个或多个随机元素，但不删除元素。

* 例：

  ```shell
  127.0.0.1:6379> SPOP k1
  "d"
  127.0.0.1:6379> SPOP k1 2
  1) "g"
  2) "c"
  ```

## 8.移动元素至另一集合

* 语法：SMOVE source destination member

* 如果source 集合不存在或者不包含指定的元素,这smove命令不执行任何操作并且返回0.

* 否则对象将会从source集合中移除，并添加到destination集合中去，

* 如果destination集合已经存在该元素，则smove命令仅将该元素充source集合中移除.

* 如果destination集合不存在，则创建destination集合，并将元素插入

*  如果source 和destination不是集合类型,则返回错误.

* 例：

  ```shell
  127.0.0.1:6379> SMEMBERS k1
  1) "e"
  2) "f"
  3) "b"
  127.0.0.1:6379> SMEMBERS k2
  1) "1"
  2) "2"
  3) "3"
  4) "4"
  5) "5"
  127.0.0.1:6379> SMOVE k1 k2 e
  (integer) 1
  127.0.0.1:6379> SMEMBERS k1
  1) "f"
  2) "b"
  127.0.0.1:6379> SMEMBERS k2
  1) "e"
  2) "5"
  3) "2"
  4) "3"
  5) "1"
  6) "4"
  ```

## 9.数学集合操作

* 语法：

  * SDIFF key [key ...]
    * 返回一个集合与给定集合的差集的元素

  * SUNION key [key ...]
    * 返回给定的多个集合的并集中的所有成员.
  * SINTER key [key ...]
    * 返回指定所有的集合的成员的交集.

* 例：

  ```shell
  127.0.0.1:6379> SMEMBERS k1
  1) "c"
  2) "a"
  3) "3"
  4) "1"
  127.0.0.1:6379> SMEMBERS k2
  1) "2"
  2) "a"
  3) "d"
  4) "4"
  127.0.0.1:6379> SMEMBERS k3
  1) "5"
  2) "e"
  3) "a"
  4) "3"
  127.0.0.1:6379> SDIFF k1 k2 k3
  1) "c"
  2) "1"
  127.0.0.1:6379> SUNION k1 k2 k3
  1) "3"
  2) "1"
  3) "c"
  4) "a"
  5) "4"
  6) "2"
  7) "d"
  8) "5"
  9) "e"
  127.0.0.1:6379> SINTER k1 k2 k3
  1) "a"
  ```


# 六、哈希操作命令

## 1.插入元素

* 语法
  * HSET key field value
  * HMSET key field value [field value ...]（批量插入）
  * HSETNX key field value：只在 `key` 指定的哈希集中不存在指定的字段时，设置字段的值

* 设置 key 指定的哈希集中指定字段的值。

* 如果 key 指定的哈希集不存在，会创建一个新的哈希集并与 key 关联。

* 如果字段在哈希集中存在，它将被重写（HSETNX除外）

* 例：

  ```shell
  127.0.0.1:6379> HSET h1 k1 v1 
  (integer) 1
  127.0.0.1:6379> HSET h1 k2 v2
  (integer) 0
  127.0.0.1:6379> HMSET h1 k1 v1 k2 v2 k3 v3 k4 v4
  OK
  127.0.0.1:6379> HKEYS h1
  1) "k1"
  2) "k2"
  3) "k3"
  4) "k4"
  127.0.0.1:6379> HVALS h1
  1) "v1"
  2) "v2"
  3) "v3"
  4) "v4"
  127.0.0.1:6379> HSETNX h1 k3 xx
  (integer) 0
  127.0.0.1:6379> HVALS h1
  1) "v1"
  2) "v2"
  3) "v3"
  4) "v4"
  127.0.0.1:6379> HSETNX h1 k5 xx
  (integer) 0
  127.0.0.1:6379> HVALS h1
  1) "v1"
  2) "v2"
  3) "v3"
  4) "v4"
  5) "xx"
  ```

## 2.获取元素

* 语法：
  * HGET key field
  * HMGET key field [field ...]（批量获取）

* 对于哈希集中不存在的每个字段，返回 `nil` 值。因为不存在的keys被认为是一个空的哈希集

* 对一个不存在的 `key` 执行将返回一个只含有 `nil` 值的列表

* 例：

  ```shell
  127.0.0.1:6379> HGET h1 k1
  "v1"
  127.0.0.1:6379> HGET h1 k5
  (nil)
  127.0.0.1:6379> HMGET h1 k1 k2 k5 k3 k4
  1) "v1"
  2) "v2"
  3) (nil)
  4) "v3"
  5) "v4"
  ```

## 3.获取所有元素

* 语法：HGETALL key

* 返回 key 指定的哈希集中所有的字段和值。

* 返回值中，每个字段名的下一个是它的值，所以返回值的长度是哈希集大小的两倍

* 例：

  ```shell
  127.0.0.1:6379> HGETALL h1
  1) "k1"
  2) "v1"
  3) "k2"
  4) "v2"
  5) "k3"
  6) "v3"
  7) "k4"
  8) "v4"
  ```

## 4.删除一个或多个元素

* 语法：HDEL key field [field ...]

* 从 key 指定的哈希集中移除指定的域。

* 在哈希集中不存在的域将被忽略。

* 如果 key 指定的哈希集不存在，它将被认为是一个空的哈希集，该命令将返回0。

* 例：

  ```shell
  127.0.0.1:6379> HGETALL h1
  1) "k1"
  2) "v1"
  3) "k2"
  4) "v2"
  5) "k3"
  6) "v3"
  7) "k4"
  8) "v4"
  127.0.0.1:6379> HDEL h1 k1 k2 k5
  (integer) 2
  127.0.0.1:6379> HGETALL h1
  1) "k3"
  2) "v3"
  3) "k4"
  4) "v4"
  127.0.0.1:6379> HDEL h2 k1 k2 k3
  (integer) 0
  ```

## 5.获取长度

* 语法：HLEN key

* 例：

  ```shell
  127.0.0.1:6379> HLEN h1
  (integer) 2
  127.0.0.1:6379> HLEN h2
  (integer) 0
  ```

## 6.判断字段是否存在

* 语法：HEXISTS key field

* 例：

  ```shell
  127.0.0.1:6379> HEXISTS h1 k1
  (integer) 0
  127.0.0.1:6379> HEXISTS h1 k3
  (integer) 1
  ```

## 7.获取指定key下的所有field/value

* 语法：
  * HKEYS key：返回 key 指定的哈希集中所有字段的名字
  * HVALS key：返回 key 指定的哈希集中所有字段的值

* 例：

  ```shell
  127.0.0.1:6379> HKEYS h1
  1) "k3"
  2) "k4"
  127.0.0.1:6379> HVALS h1
  1) "v3"
  2) "v4"
  ```

## 8.增值操作

* 语法：

  * HINCRBY key field increment
    * 增加 `key` 指定的哈希集中指定字段的数值。
    * 如果 `key` 不存在，会创建一个新的哈希集并与 `key` 关联。
    * 如果字段不存在，则字段的值在该操作执行前被设置为 0
    * `HINCRBY` 支持的值的范围限定在 64位 有符号整数
    * 如果value不是integer类型，则返回error

  * HINCRBYFLOAT key field increment
    * 为指定`key`的hash的`field`字段值执行float类型的`increment`加
    * 如果value不是float类型，则返回error
    * 其余属性同上

* 例：

  ```shell
  127.0.0.1:6379> HINCRBY h1 k1 1
  (integer) 1
  127.0.0.1:6379> HINCRBY h1 k3 1
  (error) ERR hash value is not an integer
  127.0.0.1:6379> HINCRBYFLOAT h1 k2 2.1
  "2.1"
  127.0.0.1:6379> HINCRBYFLOAT h1 k3 2.1
  (error) ERR hash value is not a float
  127.0.0.1:6379> HGETALL h1
  1) "k3"
  2) "v3"
  3) "k4"
  4) "v4"
  5) "k1"
  6) "1"
  7) "k2"
  8) "2.1"
  ```

# 七、有序集合操作命令

## 1.插入元素

* 语法：ZADD key [NX|XX] [CH] [INCR] score member [score member ...]

* 用法：

  * 将所有指定成员添加到键为`key`有序集合（sorted set）里面。 

  * 添加时可以指定多个分数/成员（score/member）对。 

  * 如果指定添加的成员已经是有序集合里面的成员，则会更新改成员的分数（scrore）并更新到正确的排序位置。

  * 如果`key`不存在，将会创建一个新的有序集合（sorted set）并将分数/成员（score/member）对添加到有序集合，就像原来存在一个空的有序集合一样。

  * 如果`key`存在，但是类型不是有序集合，将会返回一个错误应答。

  * 分数值是一个双精度的浮点型数字字符串。`+inf`和`-inf`都是有效值。

* 参数分析：
  * **XX**: 仅仅更新存在的成员，不添加新成员。
  * **NX**: 不更新存在的成员。只添加新成员。
  * **CH**: 修改返回值为发生变化的成员总数，原始是返回新添加成员的总数 (CH 是 *changed* 的意思)。更改的元素是**新添加的成员**，已经存在的成员**更新分数**。 所以在命令中指定的成员有相同的分数将不被计算在内。注：在通常情况下，`ZADD`返回值只计算新添加成员的数量。
  * **INCR**: 当`ZADD`指定这个选项时，成员的操作就等同`ZINCRBY`命令，对成员的分数进行递增操作。

* 例：

  ```shell
  127.0.0.1:6379> ZADD k1 5 a 2 b 7 c 3 d 2 e 3 f 1 g
  (integer) 7
  127.0.0.1:6379> ZRANGE k1 0 -1
  1) "g"
  2) "b"
  3) "e"
  4) "d"
  5) "f"
  6) "a"
  7) "c"
  ```

## 2.获取指定范围的元素

* 语法：
  * ZRANGE key start stop [WITHSCORES]：返回指定位置的成员（从低到高排列）
  * ZREVRANGE key start stop [WITHSCORES]：返回指定位置的成员（从高到低排列）
  * ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]：返回指定分数区间的成员（从低到高排列）
  * ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]：返回指定分数区间的成员（从高到低排列）
* 用法
  * 返回存储在有序集合`key`中的指定范围的元素。
  * 如果得分相同，将按字典排序
  * `start`（`min`）和`stop`（`max`）是**全包含的区间**
  * 可以传递`WITHSCORES`选项，以便将元素的分数与元素一起返回
  * 可以使用 `"+inf"` 和 `"-inf"` 表示得分最大值和最小值
  * 可以通过 `LIMIT` 对满足条件的成员列表进行分页。一般会配合 `"+inf"` 或者 `"-inf"` 来表示最大值和最小值

* 例：

  ```shell
  127.0.0.1:6379> ZRANGE k1 0 -1
  1) "g"
  2) "b"
  3) "e"
  4) "d"
  5) "f"
  6) "a"
  7) "c"
  127.0.0.1:6379> ZRANGE k1 0 -3
  1) "g"
  2) "b"
  3) "e"
  4) "d"
  5) "f"
  127.0.0.1:6379> ZREVRANGE k1 0 -1
  1) "c"
  2) "a"
  3) "f"
  4) "d"
  5) "e"
  6) "b"
  7) "g"
  127.0.0.1:6379> ZREVRANGE k1 0 -1 WITHSCORES
   1) "c"
   2) "7"
   3) "a"
   4) "5"
   5) "f"
   6) "3"
   7) "d"
   8) "3"
   9) "e"
  10) "2"
  11) "b"
  12) "2"
  13) "g"
  14) "1"
  127.0.0.1:6379> ZRANGEBYSCORE k1 1 6 withscores limit 1 2
  1) "b"
  2) "2"
  3) "e"
  4) "2"
  ```

## 3.获取有序集合长度

* 语法：

  * ZCARD key：返回key的有序集元素个数
  * ZCOUNT key min max：返回有序集key中，score值在min和max之间(默认包括score值等于min或max)的成员个数

* 例：

  ```shell
  127.0.0.1:6379> ZCARD k1
  (integer) 7
  127.0.0.1:6379> ZCARD k2
  (integer) 0
  127.0.0.1:6379> ZCOUNT k1 0 2
  (integer) 3
  ```

## 4.获取成员排名

* 语法：

  * ZRANK key member：有序集成员递增排列
  * ZREVRANK key member：有序集成员递减排列

* 返回有序集key中成员member的排名。其中有序集成员按score值递增(或递减)顺序排列。排名以0为底，也就是说，score值最小(或最大)的成员排名为0

* 如果member不是有序集key的成员，返回`nil`

* 例：

  ```shell
  127.0.0.1:6379> ZRANK k1 e
  (integer) 2
  127.0.0.1:6379> ZREVRANK k1 e
  (integer) 4
  127.0.0.1:6379> ZRANK k1 cc
  (nil)
  ```

## 5.获取成员分数

* 语法：ZSCORE key member

* 如果member不是有序集key的成员，返回`nil`

* 例：

  ```shell
  127.0.0.1:6379> ZSCORE k1 e
  "2"
  127.0.0.1:6379> ZSCORE k1 cc
  (nil)
  ```

## 6.删除成员

* 语法：ZREM key member [member ...]

* 例：

  ```shell
  127.0.0.1:6379> ZREM k1 e
  (integer) 1
  127.0.0.1:6379> ZREM k1 e a c
  (integer) 2
  ```


# 八、参考

* [Command reference – Redis](https://redis.io/commands)
* [Redis命令中心（Redis commands） -- Redis中国用户组（CRUG）](http://www.redis.cn/commands.html)