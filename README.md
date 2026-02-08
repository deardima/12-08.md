Задание 1. Резервное копирование
Кейс
Финансовая компания решила увеличить надёжность работы баз данных и их резервного копирования.

Необходимо описать в свободной форме, какие варианты резервного копирования подходят в случаях:

1.1. Необходимо восстанавливать данные в полном объёме за предыдущий день.

1.2. Необходимо восстанавливать данные за час до предполагаемой поломки.

Приведите ответ в свободной форме.

Задание 2. PostgreSQL
2.1. С помощью официальной документации приведите пример команды резервирования данных и восстановления БД (pgdump/pgrestore).

Приведите ответ в свободной форме.

Задание 3. MySQL
3.1. С помощью официальной документации приведите пример команды инкрементного резервного копирования базы данных MySQL.

Приведите ответ в свободной форме.

---

## Ответы на задания

### Задание 1. Резервное копирование

#### 1.1. Необходимо восстанавливать данные в полном объёме за предыдущий день.
Для этого подходит **ежедневное полное резервное копирование (Full Backup)**, выполняемое один раз в сутки (например, ночью, когда нагрузка минимальна).

**Почему это подходит:**
- Резервная копия содержит все данные на момент окончания предыдущего дня.
- Восстановление происходит быстро из одного архива.
- Минус: большой объём хранимых данных и высокая нагрузка на систему во время создания бэкапа.

**Альтернатива:** Еженедельное полное копирование + ежедневные инкрементные/дифференциальные бэкапы, но восстановление за предыдущий день потребует применения цепочки бэкапов.

#### 1.2. Необходимо восстанавливать данные за час до предполагаемой поломки.
Здесь нужен **комбинированный подход**:
1. **Полное резервное копирование** (раз в сутки или реже).
2. **Инкрементное или дифференциальное копирование** каждые несколько часов.
3. **Резервирование журналов транзакций (WAL в PostgreSQL, binlog в MySQL)** для восстановления на момент времени (Point-in-Time Recovery, PITR).

**Почему это подходит:**
- Инкрементные/дифференциальные бэкапы уменьшают нагрузку и место на диске.
- Журналы транзакций позволяют восстановить данные на точный момент (до часа до сбоя).
- Пример: полный бэкап в 00:00 + инкрементные каждые 4 часа + непрерывная архивация WAL/binlog.

---

### Задание 2. PostgreSQL

#### 2.1. С помощью официальной документации приведите пример команды резервирования данных и восстановления БД (pgdump/pgrestore).

**Пример резервирования данных с помощью `pg_dump`:**
\`\`\`bash
# Создание полного бэкапа базы данных в кастомном формате
pg_dump -U postgres -d my_database -F c -f /backup/my_database_backup.dump

# Опции:
# -U postgres      — пользователь PostgreSQL
# -d my_database   — имя базы данных для резервного копирования
# -F c             — формат custom (сжатый, позволяет выборочное восстановление)
# -f               — указание файла для сохранения бэкапа
# -v               — подробный режим (опционально)
# --clean          — добавить команды для очистки объектов БД перед восстановлением (опционально)
\`\`\`

**Пример восстановления БД с помощью `pg_restore`:**
\`\`\`bash
# Восстановление всей базы данных из бэкапа
pg_restore -U postgres -d my_database_restored /backup/my_database_backup.dump

# Восстановление только структуры базы данных (без данных)
pg_restore -U postgres -d my_database_restored --schema-only /backup/my_database_backup.dump

# Восстановление только данных (в существующую структуру)
pg_restore -U postgres -d my_database_restored --data-only /backup/my_database_backup.dump

# Восстановление с созданием новой базы данных
createdb -U postgres my_new_database
pg_restore -U postgres -d my_new_database /backup/my_database_backup.dump

# Просмотр содержимого файла бэкапа
pg_restore -l /backup/my_database_backup.dump
\`\`\`

---

### Задание 3. MySQL

#### 3.1. С помощью официальной документации приведите пример команды инкрементного резервного копирования базы данных MySQL.

**Пример инкрементного резервного копирования MySQL на основе binlog:**

**1. Настройка MySQL для ведения бинарных логов (в файле my.cnf):**
\`\`\`ini
[mysqld]
server-id = 1
log-bin = /var/log/mysql/mysql-bin.log
binlog-format = ROW
expire-logs-days = 7
max_binlog_size = 100M
\`\`\`

**2. Создание полного бэкапа с фиксацией позиции binlog:**
\`\`\`bash
# Создание полного резервного копирования всех баз данных
mysqldump -u root -p \
  --single-transaction \
  --flush-logs \
  --master-data=2 \
  --all-databases \
  --routines \
  --events \
  --triggers \
  > /backup/full_backup_$(date +%Y%m%d_%H%M%S).sql

# Опции команды:
# --single-transaction — создает согласованный снимок данных для InnoDB таблиц
# --flush-logs        — закрывает текущий бинарный лог и начинает новый
# --master-data=2     — записывает позицию бинарного лога в виде комментария
# --all-databases     — включает все базы данных в дамп
# --routines          — включает хранимые процедуры и функции
# --events            — включает события
# --triggers          — включает триггеры
\`\`\`

**3. Инкрементное копирование бинарных логов:**
\`\`\`bash
# Копирование бинарных логов, созданных после полного бэкапа
cp /var/log/mysql/mysql-bin.0* /backup/incremental/

# Или использование mysqlbinlog для извлечения изменений
mysqlbinlog \
  --read-from-remote-server \
  --host=localhost \
  --user=root \
  --password \
  --raw \
  --result-file=/backup/incremental/ \
  mysql-bin.00000*

# Ротация и архивация бинарных логов по расписанию (пример cron)
# 0 */4 * * * cp /var/log/mysql/mysql-bin.0* /backup/incremental/ && mysql -e "PURGE BINARY LOGS BEFORE NOW() - INTERVAL 1 DAY"
\`\`\`

**4. Восстановление из инкрементного бэкапа:**
\`\`\`bash
# Восстановление полного бэкапа
mysql -u root -p < /backup/full_backup_20240101.sql

# Применение инкрементных изменений из бинарных логов
mysqlbinlog /backup/incremental/mysql-bin.000002 \
  /backup/incremental/mysql-bin.000003 \
  /backup/incremental/mysql-bin.000004 \
  | mysql -u root -p

# Восстановление до определённого момента времени
mysqlbinlog \
  --stop-datetime="2024-01-01 23:59:59" \
  /backup/incremental/mysql-bin.[0-9]* \
  | mysql -u root -p

# Восстановление до определённой позиции в binlog
mysqlbinlog \
  --stop-position=123456 \
  /backup/incremental/mysql-bin.000002 \
  | mysql -u root -p
\`\`\`

**5. Использование Percona XtraBackup для физического инкрементного копирования:**
\`\`\`bash
# Полный бэкап (базовый уровень)
xtrabackup \
  --backup \
  --target-dir=/backup/full \
  --user=root \
  --password=your_password

# Подготовка полного бэкапа для восстановления
xtrabackup --prepare --target-dir=/backup/full

# Первый инкрементный бэкап (относительно полного)
xtrabackup \
  --backup \
  --target-dir=/backup/inc1 \
  --incremental-basedir=/backup/full \
  --user=root \
  --password=your_password

# Второй инкрементный бэкап (относительно предыдущего инкрементного)
xtrabackup \
  --backup \
  --target-dir=/backup/inc2 \
  --incremental-basedir=/backup/inc1 \
  --user=root \
  --password=your_password

# Подготовка инкрементных бэкапов для восстановления
xtrabackup --prepare --apply-log-only --target-dir=/backup/full
xtrabackup --prepare --apply-log-only --target-dir=/backup/full --incremental-dir=/backup/inc1
xtrabackup --prepare --target-dir=/backup/full --incremental-dir=/backup/inc2

# Восстановление из подготовленного бэкапа
systemctl stop mysql
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/backup/full
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql
\`\`\`

**6. Использование MySQL Enterprise Backup для инкрементных бэкапов:**
\`\`\`bash
# Полный бэкап
mysqlbackup --backup-dir=/backup/full --backup-image=/backup/full.mbi backup-to-image

# Инкрементный бэкап
mysqlbackup --backup-dir=/backup/inc1 --incremental --incremental-base=history:last_full_backup backup-to-image

# Восстановление
mysqlbackup --backup-dir=/backup/full --backup-image=/backup/full.mbi copy-back-and-apply-log
mysqlbackup --backup-dir=/backup/inc1 --backup-image=/backup/inc1.mbi --incremental copy-back-and-apply-log
\`\`\`
