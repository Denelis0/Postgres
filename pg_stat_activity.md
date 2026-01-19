# Тема 6: Мониторинг через pg_stat_activity

## 1. Что это такое?
`pg_stat_activity` — это системное представление (view), которое показывает **текущее состояние** всех процессов на сервере PostgreSQL.
*   Одна строка = Одно соединение (сессия).
*   Данные обновляются в реальном времени.

## 2. Ключевые столбцы

| Столбец | Описание | На что смотреть |
| :--- | :--- | :--- |
| **`pid`** | Process ID | Идентификатор процесса. Нужен, чтобы прервать сессию (`pg_terminate_backend`). |
| **`usename`** | Имя пользователя | Кто запустил запрос. |
| **`application_name`** | Приложение | Откуда подключились (1C, DBeaver, pgAdmin). |
| **`client_addr`** | IP адрес клиента | С какого компьютера пришел запрос. |
| **`state`** | Состояние процесса | **Самый важный индикатор** (см. ниже). |
| **`query_start`** | Время начала запроса | Используется для расчета длительности. |
| **`wait_event_type`** | Тип ожидания | Глобальная категория проблемы (Lock, IO, Client). |
| **`wait_event`** | Причина ожидания | Детали (чего именно ждем: диска, сети, другого пользователя). |
| **`query`** | Текст запроса | SQL код, который выполняется (или последний выполненный). |

## 3. Состояния процесса (`state`)

1.  **`active`** (Активен)
    *   Процесс прямо сейчас выполняет запрос (грузит CPU/Disk).
    *   *Норма:* Если выполняется быстро.
    *   *Проблема:* Если висит долго (> 5-10 сек для OLTP) и `wait_event` не пустой.
2.  **`idle`** (Свободен)
    *   Процесс ждет новой команды от клиента. Ничего не делает.
    *   *Безопасно.* Не потребляет процессор, не держит блокировки.
3.  **`idle in transaction`** (В транзакции, но спит) 🔴 **ОПАСНО**
    *   1С открыла транзакцию (начала менять данные), но "зависла" и не шлет команд.
    *   *Почему плохо:* Держит блокировки, мешает другим, не дает работать `VACUUM` (раздувает базу).
    *   *Действие:* Если висит долго (>10 мин) — убивать.
4.  **`idle in transaction (aborted)`**
    *   В транзакции была ошибка, ожидается `ROLLBACK`. Также вреден, как и пункт 3.

## 4. Типы ожиданий (`wait_event`) — Почему тормозим?

*   **`Lock`**: Логические блокировки.
    *   Пользователи блокируют друг друга (очередь к одной таблице/строке).
    *   *Решение:* Искать виновника (кто первый занял) и снимать его.
*   **`IO`** (`DataFileRead`, `WALWrite`): Проблемы с диском.
    *   Сервер не успевает читать/писать на диск.
    *   *Решение:* Проверить железо, настройки памяти (`shared_buffers`, `wal_buffers`), выключить `synchronous_commit` (для 1С).
*   **`LWLock`** (`buffer_mapping`, `WALWrite`): Внутренние блокировки.
    *   Высокая конкуренция за доступ к страницам в памяти.
*   **`Client`** (`ClientRead`): Ожидание сети/приложения.
    *   Обычно норма (ждем, пока 1С подумает). Если `state=active` и `ClientRead` — странно (медленная сеть или передача огромного файла).

## 5. Топ-3 запроса для диагностики

### А. Общая картина (кто грузит сервер?)
Показывает активные запросы и "зависшие" транзакции.
*Совет: включи `\x` перед запуском.*
```sql
SELECT pid, 
       usename, 
       application_name, 
       client_addr, 
       state, 
       now() - query_start AS duration, -- Длительность
       wait_event_type, 
       wait_event, 
       query 
FROM pg_stat_activity 
WHERE state != 'idle' 
  AND pid <> pg_backend_pid() -- Исключить сам мониторинг
ORDER BY duration DESC;
```

### Б. Дерево блокировок (кто держит 1С?)
Показывает пары "Виновник" -> "Жертва".
```sql
SELECT 
    blockinga.pid AS blocking_pid,       -- ВИНОВНИК
    blockinga.query AS blocking_query,   -- Что делает виновник
    '| BLOCKS >>> |',
    blockeda.pid AS blocked_pid,         -- ЖЕРТВА
    now() - blockeda.query_start AS waiting_duration,
    blockeda.query AS blocked_query
FROM pg_catalog.pg_locks blockedl
JOIN pg_stat_activity blockeda ON blockedl.pid = blockeda.pid
JOIN pg_catalog.pg_locks blockingl ON(
    ( (blockingl.transactionid=blockedl.transactionid) OR
      (blockingl.relation=blockedl.relation AND blockingl.locktype=blockedl.locktype)
    ) AND blockedl.pid != blockingl.pid)
JOIN pg_stat_activity blockinga ON blockingl.pid = blockinga.pid
WHERE NOT blockedl.granted;
```

### В. Что безопасно (ложные срабатывания)
Процессы, которые **не надо** убивать, даже если они висят сутками:
*   `autovacuum launcher` / `autovacuum worker`
*   `logical replication launcher`
*   `checkpointer`, `background writer`, `walwriter`
*   `cfs-worker` (сжатие в Postgres Pro)

## 6. Экстренные действия

Если найден вредитель, берем его `PID` и:

1.  **Попросить уйти (Soft Kill):**
    Посылает сигнал отмены запроса (`SIGINT`). Безопасно.
    ```sql
    SELECT pg_cancel_backend(PID);
    ```

2.  **Убить насмерть (Hard Kill):**
    Разрывает соединение (`SIGTERM`). Использовать, если `cancel` не помог.
    ```sql
    SELECT pg_terminate_backend(PID);
    ```

---
**Лайфхак:**
Если текст запроса слишком длинный и ломает таблицу, используй обрезку:
`left(query, 100)` в SELECT.
