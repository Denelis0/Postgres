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




# Готовый комплект: скрипт + service + timer

## 1. Скрипт `/opt/skripts/copy_weekly_dump.sh`

```bash
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

# Гарантируем, что каталог для логов существует
mkdir -p "$(dirname "$LOG_FILE")"

log_msg() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

log_msg "=========================================================="
log_msg "ЗАПУСК ЕЖЕНЕДЕЛЬНОГО КОПИРОВАНИЯ: $START_TIME_HUMAN"

# 1. Проверка монтирования папок (защита от записи на системный диск)
if ! mountpoint -q "$MOUNT_A"; then
    log_msg "ОШИБКА: ИСТОЧНИК $MOUNT_A не примонтирован! Копирование невозможно."
    exit 1
fi

if ! mountpoint -q "$MOUNT_B"; then
    log_msg "ОШИБКА: ПРИЕМНИК $MOUNT_B не примонтирован! Копирование невозможно."
    exit 1
fi

# 2. Проверка, что каталоги существуют
if [ ! -d "$SOURCE_DIR" ]; then
    log_msg "ОШИБКА: Каталог-источник $SOURCE_DIR не найден."
    exit 1
fi

if [ ! -d "$TARGET_DIR" ]; then
    log_msg "ПРЕДУПРЕЖДЕНИЕ: Каталог-приемник $TARGET_DIR не найден. Создаю."
    mkdir -p "$TARGET_DIR" || {
        log_msg "ОШИБКА: Не удалось создать $TARGET_DIR."
        exit 1
    }
fi

# 3. Поиск самого свежего дампа (.dir или .sql)
LATEST_DUMP=$(ls -td "$SOURCE_DIR"/*.dir "$SOURCE_DIR"/*.sql 2>/dev/null | head -1)

if [ -z "$LATEST_DUMP" ]; then
    log_msg "ОШИБКА: В папке $SOURCE_DIR не найдено дампов (.dir или .sql)."
    exit 1
fi

log_msg "Найден объект для переноса: $(basename "$LATEST_DUMP")"

# 4. Процесс копирования с замером времени
log_msg "Начало копирования в $TARGET_DIR..."
COPY_START=$(date +%s)

# cp -pr: p - сохранение прав и дат, r - рекурсивно (для .dir)
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

exit 0
```

Установка прав:

```bash
chmod +x /opt/skripts/copy_weekly_dump.sh
```

---

## 2. Service unit `/etc/systemd/system/copy-weekly-dump.service`

```ini
[Unit]
Description=Copy latest PostgreSQL dump from /backup01/daily to /backup02/weekly
After=network-online.target remote-fs.target
Wants=network-online.target
RequiresMountsFor=/backup01 /backup02

[Service]
Type=oneshot
User=root
Group=root
ExecStart=/opt/skripts/copy_weekly_dump.sh
# Ограничение времени на выполнение (защита от "зависшего" cp на сетевой шаре).
# Поставь с запасом — например, 6 часов.
TimeoutStartSec=21600
StandardOutput=journal
StandardError=journal
```

---

## 3. Timer unit `/etc/systemd/system/copy-weekly-dump.timer`

```ini
[Unit]
Description=Run copy-weekly-dump every Monday and Thursday at 03:00

[Timer]
OnCalendar=Mon,Thu *-*-* 03:00:00
Persistent=true
RandomizedDelaySec=5m
Unit=copy-weekly-dump.service

[Install]
WantedBy=timers.target
```

---

## 4. Установка и запуск

```bash
# 1. Перезагрузить конфигурацию systemd
systemctl daemon-reload

# 2. Включить и запустить таймер
systemctl enable --now copy-weekly-dump.timer

# 3. Проверить, что таймер активен и когда следующий запуск
systemctl list-timers copy-weekly-dump.timer

# 4. Проверить корректность расписания
systemd-analyze calendar "Mon,Thu *-*-* 03:00:00"
```

---

## 5. Проверка и отладка

Запустить вручную прямо сейчас:

```bash
systemctl start copy-weekly-dump.service
systemctl status copy-weekly-dump.service
```

Смотреть логи:

```bash
# Через journald
journalctl -u copy-weekly-dump.service -n 100 --no-pager
journalctl -u copy-weekly-dump.service -f

# Через файл лога скрипта
tail -f /var/log/postgresql/copy_weekly.log
```

---

## Что важно проверить у себя

- **`RequiresMountsFor=/backup01 /backup02`** — systemd дождётся монтирования, если шары прописаны в `/etc/fstab`. Если монтируешь вручную (например, через autofs/скрипт), эту строку лучше убрать, чтобы сервис не падал раньше времени.
- **`User=root`** — если шары доступны под другим пользователем, поменяй.
- **`TimeoutStartSec=21600`** — 6 часов. Если бэкап может быть больше — увеличь.
- **Путь `/var/log/postgresql/`** — должен существовать и быть доступен на запись (в скрипте есть `mkdir -p`, но если прав нет — упадёт).

Если нужно — могу добавить ротацию логов через `logrotate` или проверку свободного места на приёмнике перед копированием.
