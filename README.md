# homework_2_3
CREATE TABLE jsonb_table (
    id SERIAL PRIMARY KEY,
    data JSONB
);


INSERT INTO jsonb_table (data)
SELECT jsonb_build_object(
    'field1', repeat('x', 1000),
    'field2', repeat('y', 1000)
)
FROM generate_series(1, 1000000);


CREATE INDEX idxgin_jsonb_data ON jsonb_table USING gin (data jsonb_path_ops);

ALTER TABLE jsonb_table ALTER COLUMN data SET STORAGE EXTERNAL;

DO $$
BEGIN
  FOR i IN 1..10000 LOOP
    UPDATE jsonb_table
    SET data = jsonb_set(data, '{field1}', to_jsonb(repeat(chr(65 + (i % 26)), 1000)))
    WHERE id = 1;
  END LOOP;
END;
$$;


SELECT reltoastrelid FROM pg_class WHERE relname = 'jsonb_table';
-- 36873
SELECT
    relname AS table,
    pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_size
FROM pg_class
WHERE relname = 'jsonb_table' AND reltoastrelid <> 0;
--jsonb_table	52 MB
SELECT
    indexrelid::regclass AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    pg_size_pretty(pg_relation_size(indrelid)) AS table_size
FROM pg_index
JOIN pg_class ON pg_class.oid = pg_index.indexrelid
WHERE indrelid = 'jsonb_table'::regclass;
---jsonb_table_pkey	22 MB	98 MB
--idxgin_jsonb_data	3720 kB	98 MB

VACUUM FULL jsonb_table;
ANALYZE jsonb_table;



SELECT pg_size_pretty(pg_relation_size('jsonb_table')) AS table_size;
-- 96 MB

SELECT
    relname AS table,
    pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_size
FROM pg_class
WHERE relname = 'jsonb_table' AND reltoastrelid <> 0;

-- 8192 bytes
SELECT
    indexrelid::regclass AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_index
WHERE indrelid = 'jsonb_table'::regclass;
--jsonb_table_pkey	21 MB
--idxgin_jsonb_data	2144 kB
ВЫВОД:
TOAST размер упал с 52 MB до 8 KB! Значит освобождён огромный объем "мертвых" версий jsonb.
