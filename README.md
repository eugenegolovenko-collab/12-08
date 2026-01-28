# Домашнее задание к занятию "`Резервное копирование баз данных`" - `Евгений Головенко`
### Задание 1. Резервное копирование

### Кейс
Финансовая компания решила увеличить надёжность работы баз данных и их резервного копирования. 

Необходимо описать, какие варианты резервного копирования подходят в случаях: 

1.1. Необходимо восстанавливать данные в полном объёме за предыдущий день.

1.2. Необходимо восстанавливать данные за час до предполагаемой поломки.

1.3.* Возможен ли кейс, когда при поломке базы происходило моментальное переключение на работающую или починенную базу данных.

*Приведите ответ в свободной форме.*

### Решение 1

1.1. Необходимо восстанавливать данные в полном объёме за предыдущий день (RPO = 24 часа) с проверкой восстановления. Про “потерю внутри дня” речи нет.
Приемлемый вариант резервного копирования - полное ежедневное резервное копирование (Full Backup).
Каждый день снимается полный бэкап всей БД. Хранятся, допустим, 30 последних копий.
Восстановление осуществляется поднятием необходимого бэкапа, в данном случае последнего.


1.2. Необходимо восстанавливать данные за час до предполагаемой поломки. 
Можно использовать полный бэкап раз в сутки и диффиренциальный бэкап каждый час, но лучше чаще, если говорим про финтех.
Для восстановления берется последний полный бэкап и последний дифференциальный бэкап.

Если критичен размер дискового пространства, то возможно использовать инкрементный бэкап. В этом случае для восстановления берется последний полный бэкап и все инкременты с момента последнего полного бэкапа. Это займет больше времени, что может быть неприемлимым.

1.3.* Возможен ли кейс, когда при поломке базы происходило моментальное переключение на работающую или починенную базу данных?

Да, вполне. Схема может включать `master` и `standby slave` с синхронной репликацией (когда данные записываются в `master`, система ждёт, пока они точно попадут `slave`, перед тем как подтвердить операцию пользователю), лучше организовать несколько реплик под разные задачи, в том числе и под `standby`. Разумеется еще должно быть организовано дифференциальное резервное копирование с достаточной частотой. При падении `master` автоматический failover сразу переключает клиентов на `standby slave`, который становиться основным источником истины для системы (пока не поднимут упавшего). Такой подход должен обеспечить минимальный простой (RTO занимает секунды), исключить потерю данных (RPO = 0). Все работает в реальном времени и не надо ждать бэкапов. Таким образом, доступность системы из проблемы превращается в расходы, но репутация дороже.

---

### Задание 2. PostgreSQL

2.1. С помощью официальной документации приведите пример команды резервирования данных и восстановления БД (pgdump/pgrestore).

### Решение 2.

Что будет сделано:
- создание БД `netology_12_08`;
- создание таблицы `users` и наполнение ее данными;
- создание бэкапа `netology_12_08_backup.dump`;
- создание резервной базы данных `netology_12_08_restore`, чтобы не перезаписывать существующую;
- восстановление данных из бэкапа.

```sql
CREATE DATABASE netology_12_08 --создание БД netology_12_08

--создание таблицы `users` и наполнение ее данными
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50)
);

INSERT INTO users (name) VALUES
('Neo'),
('White_rabbit'),
('Johny_Fuckifaster'),
('Trinity'),
('Multifinger');

select * from users;
```
```
id|name             |
--+-----------------+
 1|Neo              |
 2|White_rabbit     |
 3|Johny_Fuckifaster|
 4|Trinity          |
 5|Multifinger      |
```

Создание бэкапа:

```cmd
pg_dump -U postgres -F c -b -v -f "e:\DB\PostgreSQL\backup\netology_12_08_backup.dump" netology_12_08
```
- -U postgres — пользователь PostgreSQL;
- -F c — формат резервной копии: custom;
- -b — включить большие объекты (BLOB);
- -v — verbose (подробный вывод);
- -f "e:\DB\PostgreSQL\backup\netology_12_08_backup.dump" — имя файла резервной копии;
- netology_12_08 — имя резервируемой базы данных.

<img width="1119" height="900" alt="Capture1" src="https://github.com/user-attachments/assets/882668d7-5ab8-4ee5-b7de-0991074ea5e6" />

<img width="701" height="223" alt="Capture2" src="https://github.com/user-attachments/assets/425d7b2c-f239-4c15-88e1-7f442f7cd1c8" />

Создание резервной базы данных:

```
CREATE DATABASE netology_12_08_restore;
```

Восстановление базы данных из бэкапа:

```cmd
C:\Program Files\PostgreSQL\18\bin>pg_restore -U postgres -d netology_12_08_restore -v "e:\DB\PostgreSQL\backup\netology_12_08_backup.dump"
```
- -U postgres — пользователь;
- -d netology_12_08_restore — база, куда восстанавливаем;
- -v — подробный режим (verbose);
- "e:\DB\PostgreSQL\backup\netology_12_08_backup.dump" — файл резервной копии.

<img width="1117" height="219" alt="Capture3" src="https://github.com/user-attachments/assets/abb4f924-2228-4472-8159-5e7e8988d0d6" />

<img width="1025" height="886" alt="Capture4" src="https://github.com/user-attachments/assets/09e50c1e-3578-4a19-94ce-2f2555d4e604" />

---

### Задание 2.1.* Возможно ли автоматизировать этот процесс? Если да, то как?

*Приведите ответ в свободной форме.*

### Решение 2.1.*

Да, возможно.

В Windows создается bat-скрипт, нечто вроде:

```bat
@echo off

set PG_BIN="C:\Program Files\PostgreSQL\18\bin"
set BACKUP_DIR=E:\DB\PostgreSQL\backup
set DB_NAME=netology_12_08
set USER=postgres
:: set PGPASSWORD="my_password" - убрать коммент, чтоб пароль не спрашивал
set DATE=%DATE:~6,4%-%DATE:~3,2%-%DATE:~0,2%

%PG_BIN%\pg_dump.exe -U %USER% -F c -b -v -f "%BACKUP_DIR%\%DB_NAME%_%DATE%.dump" %DB_NAME%

echo Yo-hoo! Backup completed!
pause
```
и запускается через планировщик задач.

В Linux так же. Создается скрипт:
```bash
#!/bin/bash

BACKUP_DIR="/var/backups/postgres"
DB_NAME="netology_12_08"
USER="postgres"
DATE=$(date +%Y-%m-%d)

pg_dump -U $USER -F c -b -v -f "$BACKUP_DIR/${DB_NAME}_${DATE}.dump" $DB_NAME
```
и запускается через `cron`

```
crontab -e
0 2 * * * /usr/local/bin/backup_postgres.sh
```
Должен будет запускаться в 02:00 каждую ночь.

Чтоб не запрашивал пароль и все было автоматично, создается файл `~/.pgpass` со следующим содержимым:

```bash
localhost:5432:netology_12_08:postgres:my_password
```

Назначаются права: `chmod 600 ~/.pgpass`

При исполнении скрипта, когда `pg_dump` поймет, что нужен пароль, то он будет искать его в определенном месте, согласно документации. 
Собственно, там он и будет находиться.

---

### Задание 3. MySQL

3.1. С помощью официальной документации приведите пример команды инкрементного резервного копирования базы данных MySQL. 

### Решение 3.

Инкрементный бэкап осуществляется из последнего полного бэкапа и всех последующих изменений из `binary log`.

- Проверяем, включен ли `binary log`.

```sql
SHOW VARIABLES LIKE 'log_bin';
```
```sql
Variable_name|Value|
-------------+-----+
log_bin      |ON   |
```
Если нет, то включаем в `C:\ProgramData\MySQL\MySQL Server 8.0\my.ini`

- Проверяем список бинарных логов:
```sql
SHOW BINARY LOGS;
```
```
Log_name                |File_size|Encrypted|
------------------------+---------+---------+
EUGEN-DESKTOP-bin.000001|      180|No       |
EUGEN-DESKTOP-bin.000002|  2099398|No       |
```
- Создаем полный бэкап базы данных `world`:
```cmd
C:\Program Files\MySQL\MySQL Server 8.0\bin>mysqldump.exe -u root -p --single-transaction --databases world > E:\DB\MySQL\world_full.sql
```
- --databases world - сохраняет CREATE DATABASE + таблицы;
- --single-transaction - отправляет команду START TRANSACTION WITH CONSISTENT SNAPSHOT в начале дампа. Для InnoDB это создаёт консистентный снимок базы, который не меняется в течение всей операции дампа;
- E:\DB\MySQL\world_full.sql - путь на другой диск, чтобы не забивать C:\

<img width="969" height="90" alt="MySQL1" src="https://github.com/user-attachments/assets/0cd4efcf-6060-4e2b-b456-95b156164362" />

- Создаем инкрементный бэкап через `binlog`:
```sql
SHOW BINARY LOGS;
Log_name                |File_size|Encrypted|
------------------------+---------+---------+
EUGEN-DESKTOP-bin.000001|      180|No       |
EUGEN-DESKTOP-bin.000002|  2099398|No       |
```
Фиксируем точку для инкримента:
```sql
FLUSH LOGS;
```
MySQL закроет текущий binlog и создаст новый файл (EUGEN-DESKTOP-bin.000003).
```sql
SHOW BINARY LOGS;
Log_name                |File_size|Encrypted|
------------------------+---------+---------+
EUGEN-DESKTOP-bin.000001|      180|No       |
EUGEN-DESKTOP-bin.000002|  2099453|No       |
EUGEN-DESKTOP-bin.000003|      157|No       |
```
Сохраняем инкремент:
```cmd
C:\Program Files\MySQL\MySQL Server 8.0\bin\mysqlbinlog.exe" --database=world "C:\ProgramData\MySQL\MySQL Server 8.0\Data\EUGEN-DESKTOP-bin.000002" > E:\DB\MySQL\world_increment_000002.sql
```
<img width="570" height="226" alt="MySQL2" src="https://github.com/user-attachments/assets/fe6947e4-3195-45ba-a5cb-fa438fd41760" />

Файл будет содержать только изменения после полного бэкапа. 

Но есть нюанс. Полный бэкап был сделан только по базе `world`, значит бинарные логи нужно фильтровать по этой базе (`--database=world`), чтобы не выполнять операции с другими базами или пользователями. Иначе при восстановлении получим ошибку `ERROR 1396 (HY000) at line 78: Operation CREATE USER failed for 'admin'@'%'`. Бинарный лог сохраняет все изменения, включая:
CREATE TABLE, INSERT, UPDATE, создание пользователей, привилегии (CREATE USER, GRANT). Когда пытаемся применить лог, то MySQL видит попытку создать пользователя `admin`. Если такой пользователь уже есть, то выдаётся эта ошибка.

Можно делать несколько инкрементов, просто указывая новые бинарные логи.

- Восстановление базы

```cmd
mysql -u root -p world < E:\DB\MySQL\world_increment_000002.sql
```
<img width="944" height="73" alt="MySQL3" src="https://github.com/user-attachments/assets/c7bbd111-abb2-4524-9d33-f6c899b602a4" />

Сделать процесс автоматичным в Windows можно используя скрипт (запускать с правами администратора):

```bat
@echo off
set MYSQL_BIN="C:\Program Files\MySQL\MySQL Server 8.0\bin"
set BACKUP_DIR=E:\DB\MySQL
set DB_NAME=world
echo [%date% %time%] Create a full backup %DB_NAME%
%MYSQL_BIN%\mysqldump.exe --login-path=local --single-transaction --databases %DB_NAME% > %BACKUP_DIR%\%DB_NAME%_full.sql
echo [%date% %time%] Fixing the binlog
%MYSQL_BIN%\mysql.exe --login-path=local -e "FLUSH LOGS;"
for /f "tokens=1" %%i in ('%MYSQL_BIN%\mysql.exe --login-path=local -e "SHOW BINARY LOGS\G" ^| findstr File ^| tail -n 1') do set LAST_BINLOG=%%i
echo [%date% %time%] Create an incremental backup of the database %DB_NAME% from %LAST_BINLOG%
%MYSQL_BIN%\mysqlbinlog.exe --login-path=local --database=%DB_NAME% "C:\ProgramData\MySQL\MySQL Server 8.0\Data\%LAST_BINLOG%" > %BACKUP_DIR%\%DB_NAME%_increment_%LAST_BINLOG%.sql
echo [%date% %time%] Yo-hoo! Backup complete!
pause
```
Чтоб постоянно не вводить пароль, можно ввести перед запуском скрипта (только в первый раз):

`C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql_config_editor.exe set --login-path=local --user=root --password`

MySQL попросит ввести пароль один раз. Он будет сохранён в шифрованном виде (%APPDATA%\MySQL\login-paths.json).

---

3.1.* В каких случаях использование реплики будет давать преимущество по сравнению с обычным резервным копированием?

*Приведите ответ в свободной форме.*

### Решение 3.1.*

- Необходимо снизить нагрузку на основной сервер во время бэкапа. Официальная рекомендация MySQL: *For large databases, backups should be taken from a replica to avoid impact on the primary server.*
- Тестирование восстановления. Можно экспериментировать с дампами, миграциями или обновлениями версии без риска для прода.
- Минимизация времени простоя. Для больших баз, где дамп может занимать часы, реплика позволяет делать периодические бэкапы “в фоне”, пользователи продолжат работать на основной базе без блокировок.
---


