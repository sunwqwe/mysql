mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.10 sec)

mysql> create database itcast;
Query OK, 1 row affected (0.03 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| itcast             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)


mysql> create database if not exists itcast;
Query OK, 1 row affected, 1 warning (0.02 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| itcast             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)


mysql> create database if not exists test;
Query OK, 1 row affected (0.01 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| itcast             |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
6 rows in set (0.00 sec)

mysql> create database itheima default charset utf8mb4;
Query OK, 1 row affected (0.01 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| itcast             |
| itheima            |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
7 rows in set (0.00 sec)


#删除数据
mysql> drop database test;
Query OK, 0 rows affected (0.03 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| itcast             |
| itheima            |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
6 rows in set (0.00 sec)

mysql> drop database if exists test;
Query OK, 0 rows affected, 1 warning (0.02 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| itcast             |
| itheima            |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
6 rows in set (0.00 sec)


mysql> use itcast;
Database changed

mysql> select database();
+------------+
| database() |
+------------+
| itcast     |
+------------+
1 row in set (0.00 sec)


mysql> use itheima;
Database changed
mysql> select database();
+------------+
| database() |
+------------+
| itheima    |
+------------+
1 row in set (0.00 sec)