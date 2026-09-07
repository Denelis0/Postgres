#!/bin/bash

# =================== НАСТРОЙКИ ===================
PORT="5432"
TARGET_DB="erp_nightly"
DIR_A="/back01/daily"
DIR_B="/back02/daily"

# Пути к утилитам (проверь, чтобы соответствовали твоей установке)
PG_PATH="/opt/pgpro/ent-17/bin"
PSQL="$PG_PATH/psql"
DROPDB="$PG_PATH/dropdb"
CREATEDB="$PG_PATH/createdb"
PG_RESTORE="$PG_PATH/pg_restore"

LOG_FILE="/var/log/postgresql/restore_nightly.log"
# =================================================

# Фиксируем время старта
START_TIME_HUMAN=$(date '+%Y-%m-%d %H:%M:%S')
START_TIME_SECONDS=$(date +%s)

log_msg() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

log_msg "=========================================================="
log_msg "НАЧАЛО ОБНОВЛЕНИЯ БАЗЫ: $START_TIME_HUMAN"
log_msg "Целевая БД: $TARGET_DB"

# 1. Поиск самого свежего дампа
LATEST_DUMP=$(ls -td "$DIR_A"/*.dir "$DIR_B"/*.dir 2>/dev/null | head -1)

if [ -z "$LATEST_DUMP" ]; then
    log_msg "ОШИБКА: Дамп не найден! Проверь монтирование /mnt/backup..."
    exit 1
fi

log_msg "Использую дамп: $LATEST_DUMP"

# 2. Сброс соединений
log_msg "Выгоняем пользователей из $TARGET_DB..."
$PSQL -p $PORT -U postgres -d postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = '$TARGET_DB' AND pid <> pg_backend_pid();" > /dev/null 2>&1

# 3. Пересоздание базы
log_msg "Пересоздание (DROP/CREATE) базы $TARGET_DB..."
$DROPDB -p $PORT -U postgres --if-exists $TARGET_DB
$CREATEDB -p $PORT -U postgres $TARGET_DB

# 4. Восстановление ( pg_restore )
log_msg "Запуск восстановления в 4 потока. Пожалуйста, подождите..."
# Замеряем время чистого pg_restore
RESTORE_START=$(date +%s)

if $PG_RESTORE -p $PORT -U postgres -d $TARGET_DB -j 4 "$LATEST_DUMP" 2>> "$LOG_FILE"; then
    RESTORE_END=$(date +%s)
    RESTORE_DIFF=$((RESTORE_END - RESTORE_START))
    log_msg "Данные успешно загружены. Время загрузки: $((RESTORE_DIFF / 60)) мин. $((RESTORE_DIFF % 60)) сек."
else
    log_msg "ВНИМАНИЕ: Восстановление завершилось с ошибками/предупреждениями."
fi

# 5. Сбор статистики
log_msg "Запуск ANALYZE для оптимизации производительности..."
$PSQL -p $PORT -U postgres -d $TARGET_DB -c "ANALYZE VERBOSE;" >> "$LOG_FILE" 2>&1

# Финальные расчеты
FINISH_TIME_SECONDS=$(date +%s)
TOTAL_DIFF=$((FINISH_TIME_SECONDS - START_TIME_SECONDS))

log_msg "ФИНИШ: $(date '+%Y-%m-%d %H:%M:%S')"
log_msg "ОБЩЕЕ ВРЕМЯ РАБОТЫ СКРИПТА: $((TOTAL_DIFF / 60)) мин. $((TOTAL_DIFF % 60)) сек."
log_msg "=========================================================="








2026-03-20 03:00:00 - ==========================================================
2026-03-20 03:00:00 - НАЧАЛО ОБНОВЛЕНИЯ БАЗЫ: 2026-03-20 03:00:00
2026-03-20 03:00:00 - Целевая БД: erp_nightly
2026-03-20 03:00:01 - Использую дамп: /backup01/daily/20.03.2026.dir
2026-03-20 03:00:01 - Выгоняем пользователей из erp_nightly...
2026-03-20 03:00:02 - Пересоздание (DROP/CREATE) базы erp_nightly...
2026-03-20 03:00:05 - Запуск восстановления в 4 потока. Пожалуйста, подождите...
2026-03-20 03:15:45 - Данные успешно загружены. Время загрузки: 15 мин. 40 сек.
2026-03-20 03:15:45 - Запуск ANALYZE для оптимизации производительности...
2026-03-20 03:17:10 - ФИНИШ: 2026-03-20 03:17:10
2026-03-20 03:17:10 - ОБЩЕЕ ВРЕМЯ РАБОТЫ СКРИПТА: 17 мин. 10 сек.
2026-03-20 03:17:10 - ==========================================================
