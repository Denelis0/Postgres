Общий процент промаха:
SELECT
    sum(heap_blks_read) AS heap_disk_reads,
    sum(heap_blks_hit)  AS heap_cache_hits,
    sum(idx_blks_read)  AS idx_disk_reads,
    sum(idx_blks_hit)   AS idx_cache_hits,
    -- Общий коэффициент попаданий (heap + indexes)
    round(
        (sum(heap_blks_hit) + sum(idx_blks_hit))::numeric /
        nullif(
            sum(heap_blks_hit) + sum(heap_blks_read) +
            sum(idx_blks_hit)  + sum(idx_blks_read),
            0
        ) * 100,
        2
    ) AS total_cache_hit_ratio
FROM pg_statio_user_tables;


10 самых нагруженный таблиц по промаху:
SELECT
    schemaname,
    relname,
    heap_blks_read,
    heap_blks_hit,
    idx_blks_read,
    idx_blks_hit,
    round(
        (heap_blks_hit + idx_blks_hit)::numeric /
        nullif(heap_blks_hit + heap_blks_read + idx_blks_hit + idx_blks_read, 0) * 100,
        2
    ) AS hit_ratio
FROM pg_statio_user_tables
WHERE (heap_blks_read + idx_blks_read) > 0
ORDER BY (heap_blks_read + idx_blks_read) DESC
LIMIT 10;


Количество грязных страниц в буферном кэше:
SELECT count(*) AS dirty_buffers
FROM pg_buffercache
WHERE isdirty = true;
