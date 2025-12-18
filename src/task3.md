## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=19.652..101.803 rows=500802 loops=1)
  Recheck Cond: (category = 'A'::text)
  Heap Blocks: exact=8334
  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=18.554..18.555 rows=500802 loops=1)
        Index Cond: (category = 'A'::text)
Planning Time: 0.745 ms
Execution Time: 113.796 ms

    
    *Объясните результат:*
PostgreSQL использует Bitmap Index Scan по индексу test_cluster_cat_idx для условия category = 'A'.
Так как условие выбирает очень большое количество строк (примерно половину таблицы), оптимизатор выбирает bitmap-доступ вместо обычного index scan.

На этапе Bitmap Heap Scan PostgreSQL читает большое число блоков таблицы (Heap Blocks: exact=8334), поскольку строки с категорией A распределены по таблице случайным образом.
Это приводит к значительному числу обращений к heap и увеличению общего времени выполнения запроса.

Таким образом, до кластеризации индекс используется корректно, но из-за отсутствия физической локальности данных чтение таблицы остаётся дорогой операцией.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
[2025-12-18 22:40:30] workshop.public> CLUSTER test_cluster USING test_cluster_cat_idx
[2025-12-18 22:40:32] completed in 1 s 532 ms

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
Bitmap Heap Scan on test_cluster  (cost=59.17..7668.56 rows=5000 width=68) (actual time=20.474..80.504 rows=500802 loops=1)
  Recheck Cond: (category = 'A'::text)
  Heap Blocks: exact=4174
  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=20.003..20.003 rows=500802 loops=1)
        Index Cond: (category = 'A'::text)
Planning Time: 1.346 ms
Execution Time: 93.572 ms

    
    *Объясните результат:*
После кластеризации таблицы по индексу test_cluster_cat_idx строки с одинаковым значением category стали физически расположены рядом.
В результате при выполнении запроса PostgreSQL по-прежнему использует Bitmap Index Scan + Bitmap Heap Scan, но читает меньше блоков таблицы (Heap Blocks: exact=4174 вместо 8334).

Улучшение физической локальности данных снижает количество обращений к heap и уменьшает время выполнения запроса.
Это приводит к заметному сокращению общего времени выполнения по сравнению с запросом до кластеризации.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
До кластеризации строки с одинаковым значением category были распределены по таблице случайным образом, что приводило к большому числу обращений к heap (Heap Blocks: exact=8334) и более высокому времени выполнения запроса (≈114 мс).

После кластеризации таблицы по индексу test_cluster_cat_idx данные с одинаковым значением category стали физически сгруппированы, что сократило количество читаемых блоков (Heap Blocks: exact=4174) и уменьшило время выполнения запроса (≈94 мс).

Таким образом, кластеризация улучшила физическую локальность данных и повысила производительность запросов, выбирающих большое количество строк по колонке category.
