/opt/skripts/copy_weekly_dump.sh



#!/bin/bash

# =================== НАСТРОЙКИ ===================
# Точки монтирования (для проверки)
MOUNT_A="/backup01"
MOUNT_B="/backup02"

# Конкретные папки
SOURCE_DIR="$MOUNT_A/daily"
TARGET_DIR="$MOUNT_B/weekly"

LOG_FILE="/var/log/postgresql/copy_weekly.log"
# =================================================

# Фиксируем время начала
START_TIME_HUMAN=$(date '+%Y-%m-%d %H:%M:%S')
START_TIME_SECONDS=$(date +%s)

log_msg() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# 1. Проверяем день недели (1 - Пн, 4 - Чт)
CURRENT_DAY=$(date +%u)

if [ "$CURRENT_DAY" -ne 1 ] && [ "$CURRENT_DAY" -ne 4 ]; then
    # Если запустить вручную в другой день, он просто выйдет. 
    # Если хочешь, чтобы при ручном запуске всегда копировал — убери это условие.
    exit 0
fi

log_msg "=========================================================="
log_msg "ЗАПУСК ЕЖЕНЕДЕЛЬНОГО КОПИРОВАНИЯ: $START_TIME_HUMAN"

# 2. Проверка монтирования папок (защита от записи на системный диск)
if ! mountpoint -q "$MOUNT_A"; then
    log_msg "ОШИБКА: ИСТОЧНИК $MOUNT_A не примонтирован! Копирование невозможно."
    exit 1
fi

if ! mountpoint -q "$MOUNT_B"; then
    log_msg "ОШИБКА: ПРИЕМНИК $MOUNT_B не примонтирован! Копирование невозможно."
    exit 1
fi

# 3. Поиск самого свежего дампа
# Ищем и папки .dir и файлы .sql
LATEST_DUMP=$(ls -td "$SOURCE_DIR"/*.dir "$SOURCE_DIR"/*.sql 2>/dev/null | head -1)

if [ -z "$LATEST_DUMP" ]; then
    log_msg "ОШИБКА: В папке $SOURCE_DIR не найдено дампов (.dir или .sql)."
    exit 1
fi

log_msg "Найден объект для переноса: $(basename "$LATEST_DUMP")"

# 4. Процесс копирования с замером времени
log_msg "Начало копирования в $TARGET_DIR..."
COPY_START=$(date +%s)

# Используем cp -pr (p - сохранение прав и дат, r - рекурсивно для папок .dir)
if cp -pr "$LATEST_DUMP" "$TARGET_DIR/"; then
    COPY_END=$(date +%s)
    DIFF=$((COPY_END - COPY_START))
    
    log_msg "УСПЕХ: Копирование завершено успешно."
    log_msg "Время переноса: $((DIFF / 60)) мин. $((DIFF % 60)) сек."
else
    log_msg "КРИТИЧЕСКАЯ ОШИБКА: Команда cp вернула ошибку."
    exit 1
fi

# 5. Финальный отчет
FINISH_TIME_SECONDS=$(date +%s)
TOTAL_DIFF=$((FINISH_TIME_SECONDS - START_TIME_SECONDS))

log_msg "ОБЩЕЕ ВРЕМЯ РАБОТЫ СКРИПТА: $((TOTAL_DIFF / 60)) мин. $((TOTAL_DIFF % 60)) сек."
log_msg "=========================================================="
