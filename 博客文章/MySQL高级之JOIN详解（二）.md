# **一、SQL执行顺序**

## 1. SQL手写结构

```sql
SELECT DISTINCT
	<select_list>
FROM 
	<left_talbe>
	<join_type> JOIN <right_table> 
	ON <join_condition>
WHERE 
	<where_condition>
GROUP BY
	<group_by_list>
HAVING
	<having_condition>
ORDER BY
	<order_by_condition>
LIMIT <limit_number>	
```

## 2. SQL执行顺序

```sql
(1)  FROM <left_table>
(2)  ON <join_condition>
(3)  <join_type> JOIN <right_table>
(4)  WHERE <where_condition>
(5)  GROUP BY <group_by_list>
(6)  HAVING <having_condition>
(7)  SELECT
(8)  DISTINCT <select_list>
(9)  ORDER BY <order_by_condition>
(10) LIMIT <limit_number>
```

**具体执行细节如下所示**：

- **1）FORM**：对FROM的左边的表和右边的表计算笛卡尔积。产生虚表VT1
- **2）ON**：对虚表VT1进行ON筛选，只有那些符合`<join-condition>`的行才会被记录在虚表VT2中。
- **3）JOIN**: 如果指定了OUTER JOIN（比如left join、 right join），那么保留表中未匹配的行就会作为外部行添加到虚拟表VT2中，产生虚拟表VT3, 如果 from子句中包含两个以上的表的话，那么就会对上一个join连接产生的结果VT3和下一个表重复执行步骤1~3这三个步骤，一直到处理完所有的表为止。
- **4）WHERE**: 对虚拟表VT3进行WHERE条件过滤。只有符合`<where-condition>`的记录才会被插入到虚拟表VT4中。
- **5）GROUP BY**: 根据group by子句中的列，对VT4中的记录进行分组操作，产生VT5.
- **6）HAVING**: 对虚拟表VT5应用having过滤，只有符合`<having-condition>`的记录才会被 插入到虚拟表VT6中。
- **7）SELECT**: 执行select操作，选择指定的列，插入到虚拟表VT7中。
- **8）DISTINCT**: 对VT77中的记录进行去重。产生虚拟表VT8.
- **9）ORDER BY**: 将虚拟表VT8中的记录按照`<order_by_condition>`进行排序操作，产生虚拟表VT9.
- **10）LIMIT**: 取出指定行的记录，产生虚拟表VT10, 并将结果返回。

# **二、JOIN的执行顺序**

## **1. JOIN的通用结构**

```sql
SELECT <row_list> 
  FROM <left_table> 
    <inner|left|right> JOIN <right_table> 
      ON <join condition> 
        WHERE <where_condition>
```

## **2. JOIN的执行顺序**

- **1）FROM**：对左右两张表执行笛卡尔积，产生第一张表vt1。行数为n*m（n为左表的行数，m为右表的行数）
- **2）ON**：根据ON的条件逐行筛选vt1，将结果插入vt2中
- **3）JOIN**：添加外部行，如果指定了**LEFT JOIN**(**LEFT OUTER JOIN**)，则先遍历一遍**左表**的每一行，其中不在vt2的行会被插入到vt2，该行的剩余字段将被填充为**NULL**，形成vt3；如果指定了**RIGHT JOIN**也是同理。但如果指定的是**INNER JOIN**，则不会添加外部行，上述插入过程被忽略。
- **4）WHERE**：对vt3进行条件过滤，满足条件的行被输出到vt4
- **5）SELECT**：取出vt4的指定字段到vt5

# **三、举例说明**

- **创建两张表**

  ```sql
  --用户信息表
  CREATE TABLE `user_info` (
    `userid` int(11) NOT NULL,
    `name` varchar(255) NOT NULL,
    UNIQUE KEY `userid` (`userid`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
  
  --用户余额表
  CREATE TABLE `user_account` (
    `userid` int(11) NOT NULL,
    `money` bigint(20) NOT NULL,
    UNIQUE KEY `userid` (`userid`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
  ```

- **插入一些初始数据**

  ```shell
  mysql> SELECT * FROM `user_info`;
  +--------+------+
  | userid | name |
  +--------+------+
  |      1 | z3   |
  |      2 | l4   |
  |      3 | w5   |
  |      6 | s6   |
  +--------+------+
  
  mysql> SELECT * FROM `user_account`;
  +--------+-------+
  | userid | money |
  +--------+-------+
  |      1 |  1000 |
  |      2 |  2000 |
  |      4 |  4000 |
  |      5 |  5000 |
  +--------+-------+
  ```

- **查询出用户编号为1的用户的姓名和余额，SQL如下**

  ```sql
  SELECT
  	a.NAME,
  	b.money 
  FROM
  	`user_info` a
  	LEFT JOIN `user_account` b ON a.userid = b.userid 
  WHERE
  	a.userid =1
  ```

  **上述SQL执行流程如下**

  - **第一步：执行FROM子句对两张表进行笛卡尔积操作**

    左表user_info有4行，右表user_account有4行，一共有4*4=16条数据

    ```shell
    mysql> select * from user_info left join user_account on 1;
    +--------+------+--------+-------+
    | userid | name | userid | money |
    +--------+------+--------+-------+
    |      1 | z3   |      1 |  1000 |
    |      1 | z3   |      2 |  2000 |
    |      1 | z3   |      4 |  4000 |
    |      1 | z3   |      5 |  5000 |
    |      2 | l4   |      1 |  1000 |
    |      2 | l4   |      2 |  2000 |
    |      2 | l4   |      4 |  4000 |
    |      2 | l4   |      5 |  5000 |
    |      3 | w5   |      1 |  1000 |
    |      3 | w5   |      2 |  2000 |
    |      3 | w5   |      4 |  4000 |
    |      3 | w5   |      5 |  5000 |
    |      6 | s6   |      1 |  1000 |
    |      6 | s6   |      2 |  2000 |
    |      6 | s6   |      4 |  4000 |
    |      6 | s6   |      5 |  5000 |
    +--------+------+--------+-------+
    16 rows in set (0.00 sec)
    ```

  - **第二步：根据ON的条件逐行筛选**

    过滤掉不满足条件`a.userid = b.userid`的数据，还剩两条

    ```shell
    +--------+------+--------+-------+
    | userid | name | userid | money |
    +--------+------+--------+-------+
    |      1 | z3   |      1 |  1000 |
    |      2 | l4   |      2 |  2000 |
    +--------+------+--------+-------+
    ```

  - **第三步：根据JOIN的类型添加外部行**

    - 如果是LEFT JOIN，左表未出现在上述结果的数据添加进来，剩余字段填充为null

      ```shell
      +--------+------+--------+-------+
      | userid | name | userid | money |
      +--------+------+--------+-------+
      |      1 | z3   |      1 |  1000 |
      |      2 | l4   |      2 |  2000 |
      |      3 | w5   |   NULL |  NULL |
      |      6 | s6   |   NULL |  NULL |
      +--------+------+--------+-------+
      ```

    - 如果是RIGHT JOIN，右表未出现在上述结果的数据添加进来，剩余字段填充为null

      ```shell
      +--------+------+--------+-------+
      | userid | name | userid | money |
      +--------+------+--------+-------+
      |      1 | z3   |      1 |  1000 |
      |      2 | l4   |      2 |  2000 |
      |   NULL | NULL |      4 |  4000 |
      |   NULL | NULL |      5 |  5000 |
      +--------+------+--------+-------+
      ```

    - 如果是INNER JOIN，不添加任何外部行，直接忽略第三步

  - **第四步：WHERE条件过滤**

    过滤掉不满足条件`a.userid =1`的数据，还剩1条

    ```shell
    +--------+------+--------+-------+
    | userid | name | userid | money |
    +--------+------+--------+-------+
    |      1 | z3   |      1 |  1000 |
    +--------+------+--------+-------+
    ```

  - **第五步：SELECT取出字段，流程结束**

    ```shell
    +------+-------+
    | name | money |
    +------+-------+
    | z3   |  1000 |
    +------+-------+
    ```

  - **结果验证**

    ```shell
    mysql> select a.name,b.money from user_info a right join user_account b on a.userid=b.userid where a.userid =1;
    +------+-------+
    | name | money |
    +------+-------+
    | z3   |  1000 |
    +------+-------+
    1 row in set (0.00 sec)
    ```

    **完全正确**

# **四、四种JOIN类型**

![](http://img.xianzilei.cn/JOIN%E7%B1%BB%E5%9E%8B.png)

- **1）INNER [OUTER] JOIN（内连接）**：两个表的**交集**
- **2）LEAF  [OUTER] JOIN（左连接）**：两个表的**交集**外加**左表**剩下的数据
- **3）RIGHT  [OUTER] JOIN（右连接）**：两个表的**交集**外加**右表**剩下的数据
- **4）FULL  [OUTER] JOIN（全连接）**：两个表的**并集**

**注意：MySQL不支持直接使用FULL JOIN，不过可以通过LEFT JOIN + UNION + RIGHT JOIN 来实现。**