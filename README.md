# Резервное копирование баз данных
## Косарев Д.О.

---

### Ответ на задание 1.1:
**Для восстановления данных в полном объёме за предыдущий день** подходит **ежедневное полное резервное копирование (Full Backup)**.

**Причины:**
1. Резервная копия содержит полный снимок всех данных на конец предыдущего дня
2. Восстановление выполняется быстро из одного файла без необходимости сборки из нескольких источников
3. Простота администрирования и контроля

**Рекомендуемая схема:**
- Выполнение полного бэкапа каждую ночь в 00:00-03:00 (период минимальной нагрузки)
- Хранение на отдельном сервере или облачном хранилище
- Регулярное тестирование восстановления

---

### Ответ на задание 1.2:
**Для восстановления данных за час до поломки** требуется **комбинированная стратегия с использованием Point-in-Time Recovery (PITR)**.

**Многоуровневая схема:**
1. **Полное резервное копирование** - выполняется 1 раз в сутки
2. **Инкрементное копирование** - выполняется каждые 2-4 часа
3. **Непрерывная архивация журналов транзакций** - WAL для PostgreSQL или binlog для MySQL

**Техническая реализация:**
График резервного копирования:
00:00 - Полный бэкап
04:00 - Инкрементный бэкап
08:00 - Инкрементный бэкап
12:00 - Инкрементный бэкап
16:00 - Инкрементный бэкап
20:00 - Инкрементный бэкап

Непрерывная архивация журналов (каждые 5-10 минут)

text

**Восстановление:**
1. Восстанавливается последний полный бэкап
2. Применяются все инкрементные бэкапы после него
3. Восстанавливаются журналы транзакций до нужного момента времени

---

### Ответ на задание 2.1:
**Примеры команд PostgreSQL для резервирования и восстановления:**

**Создание резервной копии:**
\`\`\`bash
# Создание полного дампа базы данных
pg_dump -U postgres -d company_finance -F c -v -Z 9 -f /backup/finance_$(date +%Y%m%d).dump

# Опции:
# -U postgres           - пользователь PostgreSQL
# -d company_finance    - база данных для резервирования
# -F c                  - формат custom (поддерживает сжатие)
# -v                    - подробный вывод
# -Z 9                  - максимальное сжатие
# -f                    - файл для сохранения
\`\`\`

**Восстановление базы данных:**
\`\`\`bash
# Полное восстановление базы
pg_restore -U postgres -d finance_restored /backup/finance_20240115.dump

# Восстановление только структуры (без данных)
pg_restore -U postgres -d new_database --schema-only /backup/finance_20240115.dump

# Восстановление только данных
pg_restore -U postgres -d existing_db --data-only /backup/finance_20240115.dump

# Восстановление конкретных таблиц
pg_restore -U postgres -d database --table=customers --table=transactions /backup/finance_20240115.dump
\`\`\`

**Настройка WAL архивации для PITR:**
\`\`\`bash
# В postgresql.conf:
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /backup/wal/%f && cp %p /backup/wal/%f'
\`\`\`

---

### Ответ на задание 3.1:
**Пример инкрементного резервного копирования MySQL:**

**1. Настройка MySQL для binlog:**
\`\`\`ini
# В файле /etc/mysql/my.cnf
[mysqld]
server-id = 1
log-bin = /var/log/mysql/mysql-bin.log
binlog-format = ROW
expire-logs-days = 14
max_binlog_size = 100M
sync_binlog = 1
\`\`\`

**2. Полное резервное копирование:**
\`\`\`bash
# Создание полного бэкапа с фиксацией позиции binlog
mysqldump -u backup_user -p'secure_password' \
  --single-transaction \
  --flush-logs \
  --master-data=2 \
  --all-databases \
  --routines \
  --events \
  --triggers \
  --hex-blob \
  > /backup/full_backup_$(date +%Y%m%d_%H%M%S).sql

# Проверка позиции binlog в дампе
grep "CHANGE MASTER" /backup/full_backup_20240115_000000.sql
\`\`\`

**3. Инкрементное копирование (автоматизация через скрипт):**
\`\`\`bash
#!/bin/bash
# Скрипт incremental_backup.sh
BACKUP_DIR="/backup/incremental"
LOG_FILE="/var/log/mysql_backup.log"

# Определяем последний скопированный binlog
LAST_BACKUP=$(ls -t $BACKUP_DIR/mysql-bin.* 2>/dev/null | head -1)

if [ -z "$LAST_BACKUP" ]; then
    # Первый инкрементный бэкап - копируем все binlog-файлы
    cp /var/log/mysql/mysql-bin.* $BACKUP_DIR/
else
    # Копируем только новые файлы
    LAST_FILE=$(basename $LAST_BACKUP)
    CURRENT_NUM=$(echo $LAST_FILE | grep -o '[0-9]*' | tail -1)
    
    for file in /var/log/mysql/mysql-bin.*; do
        FILE_NUM=$(echo $file | grep -o '[0-9]*' | tail -1)
        if [ "$FILE_NUM" -gt "$CURRENT_NUM" ]; then
            cp "$file" "$BACKUP_DIR/"
        fi
    done
fi

# Очистка старых binlog-файлов на сервере
mysql -u root -p'password' -e "PURGE BINARY LOGS BEFORE NOW() - INTERVAL 7 DAY"
\`\`\`

**4. Восстановление из инкрементного бэкапа:**
\`\`\`bash
# Шаг 1: Восстановление полного бэкапа
mysql -u root -p < /backup/full_backup_20240115.sql

# Шаг 2: Применение инкрементных изменений
mysqlbinlog \
  --start-datetime="2024-01-15 00:00:00" \
  --stop-datetime="2024-01-15 23:00:00" \
  /backup/incremental/mysql-bin.[0-9]* \
  | mysql -u root -p

# Или восстановление до конкретной позиции
mysqlbinlog \
  --start-position=107 \
  --stop-position=4500 \
  /backup/incremental/mysql-bin.000002 \
  | mysql -u root -p
\`\`\`

**5. Использование XtraBackup для физических инкрементных бэкапов:**
\`\`\`bash
# Полный бэкап
xtrabackup --backup --target-dir=/backup/full --user=root --password

# Инкрементный бэкап (раз в 4 часа)
xtrabackup --backup \
  --target-dir=/backup/inc_$(date +%H) \
  --incremental-basedir=/backup/full \
  --user=root \
  --password

# Подготовка к восстановлению
xtrabackup --prepare --apply-log-only --target-dir=/backup/full
xtrabackup --prepare --target-dir=/backup/full --incremental-dir=/backup/inc_04
xtrabackup --prepare --target-dir=/backup/full --incremental-dir=/backup/inc_08
\`\`\`

