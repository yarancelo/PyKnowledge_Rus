Установка и настройка будут производиться на Ubuntu-сервере

### **Установка и подключение**
Установка пакетов:
```bash
sudo apt install postgresql postgresql-contrib
```

Проверка статуса:
```bash
sudo systemctl status postgresql
```

Если ответ такой, то все установилось правильно:
```
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled; preset: enabled)
     Active: active (exited) since Mon 2025-10-13 10:55:23 MSK; 15s ago
   Main PID: 6557 (code=exited, status=0/SUCCESS)
        CPU: 1ms

Oct 13 10:55:23 SAD-IT-03 systemd[1]: Starting postgresql.service - PostgreSQL RDBMS...
Oct 13 10:55:23 SAD-IT-03 systemd[1]: Finished postgresql.service - PostgreSQL RDBMS.
```

По умолчанию в кластере postgres создается пользователь "postgres". Чтобы войти под ним в базу, нужно переключиться на него:
```bash
sudo -i -u postgres
```

После чего, зайти в интерактивную консоль командой:
```bash
psql
```

Откроется БД postgres и система пригласит ко вводу:
```bash
postgres@SAD-IT-03:~$ psql
psql (16.10 (Ubuntu 16.10-0ubuntu0.24.04.1))
Type "help" for help.

postgres=#
```

### **Базовая настройка**
Создание базы данных:
```sql
CREATE DATABASE mydb;
```
Вывод:
```
CREATE DATABASE
postgres=#
```

Создание отдельного пользователя и пароля:
```sql
CREATE USER userdb WITH PASSWORD 'password';
```
Вывод:
```sql
CREATE ROLE
postgres=#
```

Данная команда создает пользователя userdb с паролем 'password'. После создания, нужно выдать пользователю права для определенной базы данных.
```
ALTER DATABASE mydb OWNER TO userdb;
```
Вывод:
```
ALTER DATABASE
postgres=#
```

Данная команда назначает пользователя ```userdb``` владельцем базы данных ```mydb```. Однако, можно не назначать пользователя владельцем, а просто дать права: 
```
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;
```

После чего, можно попробовать войти в базу под созданным пользователем:
```bash
psql -h localhost -U userdb -d mydb
```
Вывод:
``` bash
Password for user userdb:
psql (16.10 (Ubuntu 16.10-0ubuntu0.24.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

mydb=>
```

После ввода команды система предложит ввести пароль. После ввода пароля нас присоединит не к кластеру, а уже к созданной нами базе ```mydb``` данных через созданного нами пользователя ```userdb```.

Последний шаг, который осталось выполнить в рамках базовой настройки - разрешить подключение к базе данных не только с локальный машины, но и извне. Для этого нужно зайти в конфигурационный файл:```sudo nano /etc/postgresql/16/main/postgresql.conf```и найти строку "listen_addresses", после чего, указать возможность подключения с любых адресов подставив звездочку вместо localhost:
```
listen_addresses = '*'
```

В конфигурационный файл  ```/etc/postgresql/<версия>/main/pg_hba.conf``` добавить следующую строку:
```
# Разрешаем подключение к mydb для пользователя myuser с любого IP по паролю
host    mydb    myuser    0.0.0.0/0    md5
```

Данная строка в конфиге разрешает подключение к базе данных только пользователю mydb и только к базе данных myuser.

После редактирования конфигурационных файлов нужно перезагрузить службу postgres:
```bash
sudo systemctl restart postgresql
```

### **Подключение к БД через СУБД DBeaver**

Для начала нужно узнать на каком порте работает кластер postgres:
```bash
sudo ss -ltnp | grep postgres
```

Далее в Dbeaver добавляем новую базу данных:
```
хост: <Ip/DNS сервера>
база данных: <имя бд>
аутентификация: <database native>
пользователь: <пользователь БД>
пароль: <пароль от пользователя <БД>
порт: <по-умолчанию 5432>
```

