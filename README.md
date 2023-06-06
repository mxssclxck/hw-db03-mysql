# Домашнее задание к занятию 3. «MySQL»

# Никоноров Денис - FOPS-6

## Задача 1

Используя Docker, поднимите инстанс MySQL (версию 8). Данные БД сохраните в volume.

Изучите [бэкап БД](https://github.com/netology-code/virt-homeworks/tree/virt-11/06-db-03-mysql/test_data) и 
восстановитесь из него.

Перейдите в управляющую консоль `mysql` внутри контейнера.

Используя команду `\h`, получите список управляющих команд.

Найдите команду для выдачи статуса БД и **приведите в ответе** из её вывода версию сервера БД.

Подключитесь к восстановленной БД и получите список таблиц из этой БД.

**Приведите в ответе** количество записей с `price` > 300.

В следующих заданиях мы будем продолжать работу с этим контейнером.<br>
***
Создаем контейнер с MySQL8
```yml
version: '3'

services:
  db:
    image: mysql
    container_name: mysql8
    environment:
      MYSQL_ROOT_PASSWORD: password
    volumes:
      - ./db_data:/var/lib/mysql
      - ./test-dump:/dump


  adminer:
    image: adminer:latest
    container_name: adminer
    ports:
      - 8080:8080
```
Проверяем что контейнер запущен
```bash
docker ps -a
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                                                  NAMES
8180238a62ae   mysql                  "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   mysql8
531e8ee0338a   chatgpt-telegram-bot   "python bot/main.py"     5 days ago      Up 3 days                                                             chatgpt-telegram-bot

docker exec -it mysql8 bash -c "mysql -u root -p test_db --version;"
mysql  Ver 8.0.33 for Linux on x86_64 (MySQL Community Server - GPL)
```
```bash
mysql> use test_db
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> \s
--------------
mysql  Ver 8.0.33 for Linux on x86_64 (MySQL Community Server - GPL)

Connection id:		21
Current database:	test_db
Current user:		root@localhost
SSL:			Not in use
Current pager:		stdout
Using outfile:		''
Using delimiter:	;
Server version:		8.0.33 MySQL Community Server - GPL
Protocol version:	10
Connection:		Localhost via UNIX socket
Server characterset:	utf8mb4
Db     characterset:	utf8mb4
Client characterset:	latin1
Conn.  characterset:	latin1
UNIX socket:		/var/run/mysqld/mysqld.sock
Binary data as:		Hexadecimal
Uptime:			39 min 14 sec

Threads: 2  Questions: 93  Slow queries: 0  Opens: 185  Flush tables: 3  Open tables: 103  Queries per second avg: 0.039
```

```bash
mysql> show tables;
+-------------------+
| Tables_in_test_db |
+-------------------+
| orders            |
+-------------------+
1 row in set (0.00 sec)
```

```bash
mysql> SELECT COUNT(*) FROM orders WHERE price > 300;
+----------+
| COUNT(*) |
+----------+
|        1 |
+----------+
1 row in set (0.00 sec)
```

## Задача 2

Создайте пользователя test в БД c паролем test-pass, используя:

- плагин авторизации mysql_native_password
- срок истечения пароля — 180 дней 
- количество попыток авторизации — 3 
- максимальное количество запросов в час — 100
- аттрибуты пользователя:
    - Фамилия "Pretty"
    - Имя "James".
***
```bash
CREATE USER 'test'@'localhost'
  IDENTIFIED WITH mysql_native_password BY 'test-pass'
  WITH MAX_QUERIES_PER_HOUR 100
  PASSWORD EXPIRE INTERVAL 180 DAY
  FAILED_LOGIN_ATTEMPTS 3
  ATTRIBUTE '{"fname": "James", "lname": "Pretty"}';
Query OK, 0 rows affected (0.02 sec)
```
Проверим атрибуты пользователя `test`:

```bash
SELECT user, plugin, password_lifetime, max_questions, User_attributes FROM mysql.user WHERE user = 'test';
+------+-----------------------+-------------------+---------------+-------------------------------------------------------------------------------------------------------------------------------------+
| user | plugin                | password_lifetime | max_questions | User_attributes                                                                                                                     |
+------+-----------------------+-------------------+---------------+-------------------------------------------------------------------------------------------------------------------------------------+
| test | mysql_native_password |               180 |           100 | {"metadata": {"fname": "James", "lname": "Pretty"}, "Password_locking": {"failed_login_attempts": 3, "password_lock_time_days": 0}} |
+------+-----------------------+-------------------+---------------+-------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

Даннеые information_schema.user_attributes:
```bash
SELECT * FROM information_schema.user_attributes WHERE user = 'test';
+------+-----------+---------------------------------------+
| USER | HOST      | ATTRIBUTE                             |
+------+-----------+---------------------------------------+
| test | localhost | {"fname": "James", "lname": "Pretty"} |
+------+-----------+---------------------------------------+
1 row in set (0.00 sec)
```

## Задача 3

Установите профилирование `SET profiling = 1`.
Изучите вывод профилирования команд `SHOW PROFILES;`.

Исследуйте, какой `engine` используется в таблице БД `test_db` и **приведите в ответе**.

Измените `engine` и **приведите время выполнения и запрос на изменения из профайлера в ответе**:
- на `MyISAM`,
- на `InnoDB`.
***
Установка профилирования:
```bash
mysql> SET profiling = 1;
Query OK, 0 rows affected, 1 warning (0.00 sec)
```
Зафиксируем время выполнения запроса `SELECT * FROM orders WHERE price > 100;`:
```bash
mysql> SELECT * FROM orders WHERE price >100;
+----+-----------------------+-------+
| id | title                 | price |
+----+-----------------------+-------+
|  2 | My little pony        |   500 |
|  3 | Adventure mysql times |   300 |
|  4 | Server gravity falls  |   300 |
|  5 | Log gossips           |   123 |
+----+-----------------------+-------+
4 rows in set (0.00 sec)

mysql> SHOW profiles;
+----------+------------+---------------------------------------+
| Query_ID | Duration   | Query                                 |
+----------+------------+---------------------------------------+
|        1 | 0.00044675 | SELECT * FROM orders WHERE price >100 |
+----------+------------+---------------------------------------+
1 row in set, 1 warning (0.00 sec)
```
Проверим текущий engine:
```bash
mysql> SELECT table_schema, table_name, engine FROM information_schema.tables WHERE table_name = 'orders';
+--------------+------------+--------+
| TABLE_SCHEMA | TABLE_NAME | ENGINE |
+--------------+------------+--------+
| test_db      | orders     | InnoDB |
+--------------+------------+--------+
1 row in set (0.01 sec)
```
Используется движок InnoDB. Сменим на MyISAM и так же зафиксируем время выполнения:

```bash
mysql> ALTER TABLE orders ENGINE = MyISAM;
Query OK, 5 rows affected (0.04 sec)
Records: 5  Duplicates: 0  Warnings: 0

mysql> SELECT * FROM orders WHERE price > 100;
+----+-----------------------+-------+
| id | title                 | price |
+----+-----------------------+-------+
|  2 | My little pony        |   500 |
|  3 | Adventure mysql times |   300 |
|  4 | Server gravity falls  |   300 |
|  5 | Log gossips           |   123 |
+----+-----------------------+-------+
4 rows in set (0.00 sec)

mysql> SHOW profiles;
+----------+------------+----------------------------------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                                              |
+----------+------------+----------------------------------------------------------------------------------------------------+
|        1 | 0.00044675 | SELECT * FROM orders WHERE price >100                                                              |
|        2 | 0.00116150 | SELECT table_schema, table_name, engine FROM information_schema.tables WHERE table_name = 'orders' |
|        3 | 0.03212100 | ALTER TABLE orders ENGINE = MyISAM                                                                 |
|        4 | 0.00054425 | SELECT * FROM orders WHERE price > 100                                                             |
+----------+------------+----------------------------------------------------------------------------------------------------+
4 rows in set, 1 warning (0.00 sec)
```

* `InnoDB`: Время выполнения: 0.00044675.
* `MyISAM`: Время выполнения: 0.00054425.

## Задача 4 

Изучите файл `my.cnf` в директории /etc/mysql.

Измените его согласно ТЗ (движок InnoDB):

- скорость IO важнее сохранности данных;
- нужна компрессия таблиц для экономии места на диске;
- размер буффера с незакомиченными транзакциями 1 Мб;
- буффер кеширования 30% от ОЗУ;
- размер файла логов операций 100 Мб.

Приведите в ответе изменённый файл `my.cnf`.
***
Сохраним копию файла `/etc/my.cnf`:
```bash
cp /etc/my.cnf /etc/my.cnf.bak
```
Дополним файл my.cnf следующими строками (в порядке предъявляемых требований):
```
innodb_flush_method=O_DSYNC
innodb_flush_log_at_trx_commit=2
innodb_file_per_table=ON
innodb_log_buffer_size=1M
innodb_buffer_pool_size=2G
innodb_log_file_size=100M
```

![alt text](/img/1.png)