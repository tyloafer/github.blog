---
abbrlink: 0
---
mysql> explain SELECT `code`, `platform`, `os_version`, `app_version`, `brand_model` FROM `user_info` WHERE `code` IN (103022, 100818, 102284, 102912, 1005550, 101732, 105107, 1004861, 3000, 1003939, 1005238, 103077, 100122, 1005049, 1006959, 1006892, 1004428, 1007251, 1006642, 1003685);
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | user_info | NULL       | ALL  | code          | NULL | NULL    | NULL | 6410 |    50.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+

user_info | CREATE TABLE `user_info` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `code` varchar(255) NOT NULL);



+----+-------------+----------+------------+-------+---------------+-------------+---------+------+------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key         | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+----------+------------+-------+---------------+-------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | location | NULL       | range | expires_idx   | expires_idx | 5       | NULL | 2318 |   100.00 | Using where; Using index |
+----+-------------+----------+------------+-------+---------------+-------------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> 
mysql> explain select count(*) from `location` use index(`expires_idx`) where `expires`>'2018-12-21 10:17:43'  AND `partition`=11 AND `keepalive`=1;
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | location | NULL       | ALL  | expires_idx   | NULL | NULL    | NULL | 2324 |     1.00 | Using where |
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> 
mysql> 
mysql> explain select count(*) from `location` use index(`expires_idx`) where `expires`>'2018-12-21 10:17:43'  AND `partition`=11;
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+-------------+

根据 filtered  来决定是否使用索引