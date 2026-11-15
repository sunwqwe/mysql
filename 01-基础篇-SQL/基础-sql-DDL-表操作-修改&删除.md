mysql> alter table emp add nickname varchar(20) comment '昵称';
Query OK, 0 rows affected (0.11 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> desc emp;
+-----------+------------------+------+-----+---------+-------+
| Field     | Type             | Null | Key | Default | Extra |
+-----------+------------------+------+-----+---------+-------+
| id        | int              | YES  |     | NULL    |       |
| workno    | varchar(10)      | YES  |     | NULL    |       |
| name      | varchar(10)      | YES  |     | NULL    |       |
| gender    | char(1)          | YES  |     | NULL    |       |
| age       | tinyint unsigned | YES  |     | NULL    |       |
| idcard    | char(18)         | YES  |     | NULL    |       |
| entrydate | date             | YES  |     | NULL    |       |
| nickname  | varchar(20)      | YES  |     | NULL    |       |
+-----------+------------------+------+-----+---------+-------+
8 rows in set (0.00 sec)


mysql> alter table emp change nickname username varchar(30) comment '用户名';
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> desc emp;
+-----------+------------------+------+-----+---------+-------+
| Field     | Type             | Null | Key | Default | Extra |
+-----------+------------------+------+-----+---------+-------+
| id        | int              | YES  |     | NULL    |       |
| workno    | varchar(10)      | YES  |     | NULL    |       |
| name      | varchar(10)      | YES  |     | NULL    |       |
| gender    | char(1)          | YES  |     | NULL    |       |
| age       | tinyint unsigned | YES  |     | NULL    |       |
| idcard    | char(18)         | YES  |     | NULL    |       |
| entrydate | date             | YES  |     | NULL    |       |
| username  | varchar(30)      | YES  |     | NULL    |       |
+-----------+------------------+------+-----+---------+-------+
8 rows in set (0.00 sec)


mysql> alter table emp drop username;
Query OK, 0 rows affected (0.10 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> desc emp;
+-----------+------------------+------+-----+---------+-------+
| Field     | Type             | Null | Key | Default | Extra |
+-----------+------------------+------+-----+---------+-------+
| id        | int              | YES  |     | NULL    |       |
| workno    | varchar(10)      | YES  |     | NULL    |       |
| name      | varchar(10)      | YES  |     | NULL    |       |
| gender    | char(1)          | YES  |     | NULL    |       |
| age       | tinyint unsigned | YES  |     | NULL    |       |
| idcard    | char(18)         | YES  |     | NULL    |       |
| entrydate | date             | YES  |     | NULL    |       |
+-----------+------------------+------+-----+---------+-------+
7 rows in set (0.00 sec)


mysql> alter table emp rename to employee;
Query OK, 0 rows affected (0.03 sec)

mysql> show tables;
+------------------+
| Tables_in_itcast |
+------------------+
| employee         |
| tb_user          |
+------------------+
2 rows in set (0.00 sec)


mysql> drop table if exists tb_user;
Query OK, 0 rows affected (0.03 sec)

mysql> show tables;
+------------------+
| Tables_in_itcast |
+------------------+
| employee         |
+------------------+
1 row in set (0.00 sec)


mysql> truncate table employee;
Query OK, 0 rows affected (0.06 sec)

mysql> show tables;
+------------------+
| Tables_in_itcast |
+------------------+
| employee         |
+------------------+
1 row in set (0.00 sec)