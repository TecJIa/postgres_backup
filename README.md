# homework- postgres_backup

Описание домашнего задания
---
1) Настроить hot_standby репликацию с использованием слотов


2) Настроить правильное резервное копирование


---
ОС для настройки: Ubuntu 22.04

Vagrant версии 2.4.1

VirtualBox версии 7.0.18

---
- Этап 1: Создаем Vagrantfine, запускаем ВМ.

После того, как VM развернулись, получаем 3 хоста:

**Node1**, **Node3**, **Barman** - будет нашим мастером


**PS** Node2 пала смертью храбрых в процессе тестирования решения проблем. Описано ниже


**PS PS** **ВАЖНО!!!*** В ходе работы я сталкивался с разными ошибками. Описан будет полный ход, ВКЛЮЧАЯ неверные шаги!! Поэтому, чтобы предотвратить их повторения в будщем, используя это руководство, я буду ставить отметки **НЕВЕРНЫЙ ШАГ** с примечанием, где описано исправление  

---
- Этап 2: Настройка hot_standby репликации с использованием слотов.

**На node1 + node3**


**Устанавливаем postgresql-server 14, стартуем, добавляем в автозагрузку**


```bash
apt install postgresql postgresql-contrib
systemctl start postgresql
systemctl enable postgresql
```

---
**На хосте node1**


**Заходим в psql**

```bash
sudo -u postgres psql
```


---
**НЕВЕРНЫЙ ШАГ_1** 
---


**В psql создаём пользователя replicator c правами репликации и паролем «Otus2024!»**

```bash
CREATE USER replicator WITH REPLICATION Encrypted PASSWORD 'Otus2022!';
```


---
**В файле /etc/postgresql/14/main/postgresql.conf указываем следующие параметры**


```bash
#Указываем ip-адреса, на которых postgres будет слушать трафик на порту 5432 (параметр port)
listen_addresses = 'localhost, 192.168.57.11'
#Указываем порт порт postgres
port = 5432 
#Устанавливаем максимально 100 одновременных подключений
max_connections = 100
log_directory = 'log' 
log_filename = 'postgresql-%a.log' 
log_rotation_age = 1d 
log_rotation_size = 0 
log_truncate_on_rotation = on 
max_wal_size = 1GB
min_wal_size = 80MB
log_line_prefix = '%m [%p] ' 
#Указываем часовой пояс для Москвы
log_timezone = 'UTC+3'
timezone = 'UTC+3'
datestyle = 'iso, mdy'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8' 
lc_numeric = 'en_US.UTF-8' 
lc_time = 'en_US.UTF-8' 
default_text_search_config = 'pg_catalog.english'
#можно или нет подключаться к postgresql для выполнения запросов в процессе восстановления; 
hot_standby = on
#Включаем репликацию
wal_level = replica
#Количество планируемых слейвов
max_wal_senders = 3
#Максимальное количество слотов репликации
max_replication_slots = 3
#будет ли сервер slave сообщать мастеру о запросах, которые он выполняет.
hot_standby_feedback = on
#Включаем использование зашифрованных паролей
password_encryption = scram-sha-256
```


---
**Настраиваем параметры подключения в файле /etc/postgresql/14/main/pg_hba.conf** Добавляем строки в конец


```bash
host    replication replication    192.168.57.11/32        scram-sha-256
host    replication replication    192.168.57.12/32        scram-sha-256
```


---
**Перезапускаем postgresql-server** 


```bash
systemctl restart postgresql
```


---
**ПЕРЕХОДИМ на хосте node3**


**Останавливаем postgresql-server**


```bash
systemctl stop postgresql
```


---
**С помощью утилиты pg_basebackup копируем данные с node1**


**команда из методички с ошибкой** "pg_basebackup -h 192.168.57.11 -U    /var/lib/postgresql/14/main/ -R -P"


**Верная команду вот**

```bash
#логинимся в пользователя postgres (Этот шаг ВЕРНЫЙ, но на скрине я сначала упустил его)
su - postgres
#пробуем подключиться
pg_basebackup -h 192.168.57.11 -U replication -R -P -D /var/lib/postgresql/14/main/
```


**Получаем ошибку доступа** Долго разбираемся, почему. 

Пользователь кому разрешена репликация у нас replicat**ion**, мы указывали его в конфиге на node1

**Но** создавали мы вроде другого пользователя?...replicat**or**


---
**_ВЕРНЫЙ ШАГ_1**
---
**Создадим другого пользователя**



**Повторим попытку подклчючения** Но упираемся в другую проблему! Этот момент в методичке не описан

Ковырялся я с ним довольно долго, и именно тут убил node2, поэтому опишу эти попытки в самом конце работы. 

**Сразу к верному решению**


1) Логинимся под пользователем postgres
2) Удаляем дефолтную директорию
3) Создаем новую, с таким же названием
4) Даем ей права
5) Проверяем
6) Только теперь пробуем подключиться


**В файле  /etc/postgresql/14/main/postgresql.conf меняем параметр**


```bash
listen_addresses = 'localhost, 192.168.57.12'
```


**Запускаем службу**

```bash
systemctl start postgresql
```

---
**Проверяем репликацию**


**Переходим на хост node1** и логинимся в psql


```bash
#создадим базу otus_test_N_2 
CREATE DATABASE otus_test;
#и выведем список БД
\l
```

---
**На хосте node2** также в psql также проверим список БД (команда \l), в списке БД должна появится БД otus_test_N_2

**Примечание** На скрине ниже показан вывод списка БД **ДО** создания на node1 и **ПОСЛЕ**. 


---
**Также можно проверить репликацию другим способом**


На хосте node1 в psql вводим команду: select * from pg_stat_replication;
На хосте node2 в psql вводим команду: select * from pg_stat_wal_receiver;
Вывод обеих команд должен быть не пустым. 











```bash
# Вот так работает
yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm
percona-release setup ps57
yum install Percona-Server-server-57
```


![images2](./images/mysql_1.png)
![images2](./images/mysql_2.png)
![images2](./images/mysql_3.png)
![images2](./images/mysql_4.png)


**По умолчанию Percona хранит файлы в таком виде:**

● Основной конфиг в /etc/my.cnf

● Так же инклудится директория /etc/my.cnf.d/ - куда мы и будем складывать наши конфиги.

● Дата файлы в /var/lib/mysql


---
**Копируем конфиги в /etc/my.cnf.d/**

![images2](./images/mysql_5.png)


**Запускаем службу**

```bash
systemctl start mysql
```

![images2](./images/mysql_6.png)


*При установке Percona автоматически генерирует пароль для пользователя root и кладет его в
файл /var/log/mysqld.log*


**Смотрим пароль**

```bash
cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'
```


**Подключаемся к mysql**


```bash
mysql -uroot -p'<PASS_из_предыдущей_команды>'
```

![images2](./images/mysql_7.png)




**После подклчения меняем пароль** Кстати, тут есть требования к сложности пароля


```bash
ALTER USER USER() IDENTIFIED BY '<YourStrongPassword>';
```

![images2](./images/mysql_8.png)


---
**Следует обратить внимание**, что атрибут server-id на мастер-сервере должен обязательно отличаться от server-id слейв-сервера. Проверить какая переменная установлена в текущий момент можно следующим образом:

```bash
mysql> SELECT @@server_id;
```


**Убеждаемся, что GTID Включен**

```bash
mysql> SHOW VARIABLES LIKE 'gtid_mode';
```


![images2](./images/mysql_9.png)



---
**Создаем тестовую базу данных bet и загружаем в нее дамп**

```bash
CREATE DATABASE bet;
```

![images2](./images/mysql_10.png)


```bash
mysql -uroot -p -D bet < /vagrant/bet.dmp
```


**Вот тут я долго тупил, так как не заметил, где выполняю команду** Подгрузка дампа идет с основной машины, только потом проваливаемся в нее и смотрим 


![images2](./images/mysql_11.png)


```bash
mysql> USE bet;
mysql> SHOW TABLES;
```

![images2](./images/mysql_12.png)



---
**Создаем пользователя для репликации и даем ему права на выполнение репликации**

```bash
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '!OtusLinux2024';
mysql> SELECT user,host FROM mysql.user where user='repl';
# даем права
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '!OtusLinux2024';
```


![images2](./images/mysql_13.png)



---
**Делаем дамп базы для дальнейшего импорта на slave и игнорируем таблицы, по заданию** (команда в одну строку)

```bash
mysql> mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql
```


![images2](./images/mysql_14.png)



---
- Этап 3: Настройка Slave


**Созданный дамп нужно закинуть с Мастера на Слэйф**

Я использовал scp

![images2](./images/mysql_15.png)


---
На Slave сервере так же создаем\перекидываем конфиги в каталог /etc/my.cnf.d/


![images2](./images/mysql_16.png)



---
**В конфиге /etc/my.cnf.d/01-base.cnf поправим директиву server-id = 2**, так как они должны отличаться с мастером


![images2](./images/mysql_17.png)



---
**Аналогично как на Мастере, устанавливаем сервер, меняем пароль** На заметку, пока не стартануть, пароль не создается 


![images2](./images/mysql_18.png)


---
**Проверяем директиву server-id = 2**


```bash
mysql> SELECT @@server_id;
```


![images2](./images/mysql_19.png)



---
**Раскомментируем в /etc/my.cnf.d/05-binlog.cnf строки** Это нужно, чтобы указать таблицы, которые будут игнорироваться при репликации данных

```bash
#replicate-ignore-table=bet.events_on_demand
#replicate-ignore-table=bet.v_same_event
```

![images2](./images/mysql_20.png)




---
- Этап 4: Импорт дампа

**Заливаем дамп**

**Примечание**. Если ошибиться с директорией, получаем ошибку 


```bash
mysql> SOURCE /mnt/master.sql
```


![images2](./images/mysql_21.png)

![images2](./images/mysql_22.png)

![images2](./images/mysql_23.png)



---
**Проверяем залитую базу** 


```bash
mysql> SHOW DATABASES LIKE 'bet';
mysql> USE bet;
mysql> SHOW TABLES;

# видим что таблиц v_same_event и events_on_demand нет
```


![images2](./images/mysql_24.png)


---
**Подключаемся и запускаем слэйв** 


```bash
CHANGE MASTER TO MASTER_HOST = "192.168.57.10", MASTER_PORT = 3306, MASTER_USER = "repl", MASTER_PASSWORD = "!OtusLinux2024", MASTER_AUTO_POSITION = 1;
# не забываем сменить пароль и IP адрес, если надо
```


![images2](./images/mysql_25.png)


```bash
mysql> START SLAVE;
mysql> SHOW SLAVE STATUS\G
```



---
**А теперь залипаем на часа полтора, потому что ловим ошибку** (увы, форумы гугла от 2005 года не особо помогли в решении(((( )

![images2](./images/mysql_26.png)
![images2](./images/mysql_27.png)



---
**Смотрим, что было сделано до этого, где что упустил**

**Нашел при заливке дампа вот такое. Наверно, это что-то нехорошее :DDD**

![images2](./images/mysql_28.png)


**Гугл опять не особо помогает, либо я не там ищу где-то**



```bash
Пробовал пересоздать таблицу
удалить
залить дамп без таблицы

Не помогло ничего
```


**Возникла мысль**, что, может при передачи файла с мастера на слейф - он побился?! Не, ну мало ли, виртуалки же, ресурсов не очень много. Пошел проверять элементарно размеры файла на мастере и на слэйве


![images2](./images/mysql_29.png)


**Одинаково……** НО, почему-то мой взгляд зацепился, что пользователь файла vagrant….а я же запускаюсь как root….

**Расширяем права на файл**


![images2](./images/mysql_30.png)


```bash
После этого
Подключаюсь к базе на слэйве
Удаляю старую таблицу bet (на всякий случай)
Создаю заново эту таблицу
Заливаю дамп повторно

Ошибку не увидел! Ура)) 
```


![images2](./images/mysql_31.png)


---
**Делаем проверку**


![images2](./images/mysql_32.png)


---
**Второй раз** кстати подключаться к мастеру не надо


![images2](./images/mysql_33.png)


**Просто стартуем и проверяем**


![images2](./images/mysql_34.png)
![images2](./images/mysql_35.png)


**Наконец-то, репликациā работает, gtid работает и игнорируются нужные таблички**



---
- Этап 5: Проверяем работу репликации

**На мастере** вносим изменения в таблицу

```bash
mysql> USE bet;
mysql> INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet');
mysql> SELECT * FROM bookmaker; 
```

![images2](./images/mysql_36.png)


**На слэйве** просто проверяем

```bash
mysql> SELECT * FROM bookmaker;
```

![images2](./images/mysql_37.png)


**Видим, что на слэйве появились изменения!**. Если что, 1xbet - это не реклама ставок :D









