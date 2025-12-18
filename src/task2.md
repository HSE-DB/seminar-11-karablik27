## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
Bitmap Heap Scan on t_books  (cost=21.03..1335.59 rows=750 width=33) (actual time=0.080..0.081 rows=1 loops=1)
"  Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
  Heap Blocks: exact=1
  ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.067..0.068 rows=1 loops=1)
"        Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
Planning Time: 0.973 ms
Execution Time: 0.134 ms

    
    *Объясните результат:*
PostgreSQL использует GIN-индекс t_books_fts_idx, созданный по выражению to_tsvector('english', title).
На этапе Bitmap Index Scan индекс быстро находит документы, содержащие лексему expert.

Затем выполняется Bitmap Heap Scan, который читает только те строки таблицы, для которых индекс указал возможное совпадение, с повторной проверкой условия (Recheck Cond).
Количество затронутых блоков минимально (Heap Blocks: exact=1), что указывает на высокую селективность полнотекстового индекса и эффективное выполнение запроса.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.050..0.050 rows=1 loops=1)
  Index Cond: ((item_key)::text = '0000000455'::text)
Planning Time: 0.397 ms
Execution Time: 0.091 ms

     
     *Объясните результат:*
PostgreSQL использует Index Scan по первичному ключу t_lookup_pk, так как условие запроса — точное сравнение по item_key.
B-tree индекс позволяет быстро найти нужную запись по ключу без последовательного сканирования таблицы.

После нахождения записи в индексе выполняется обращение к heap для получения значения item_value.
Запрос возвращает одну строку, что подтверждает высокую селективность условия и эффективное использование индекса.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.340..0.342 rows=1 loops=1)
  Index Cond: ((item_key)::text = '0000000455'::text)
Planning Time: 3.970 ms
Execution Time: 0.417 ms

     
     *Объясните результат:*
PostgreSQL использует Index Scan по первичному ключу t_lookup_clustered_pkey, так как условие запроса — точное сравнение по item_key.

Поскольку таблица была кластеризована по этому индексу, данные в heap физически упорядочены в соответствии с ключом.
Это позволяет быстрее получить нужную строку после обращения к индексу, так как индексная запись и соответствующая строка таблицы находятся близко друг к другу на диске.

Запрос возвращает одну строку, и использование индексного доступа обеспечивает эффективное выполнение поиска по ключу.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.087..0.088 rows=0 loops=1)
  Index Cond: ((item_value)::text = 'T_BOOKS'::text)
Planning Time: 1.085 ms
Execution Time: 0.148 ms

     
     *Объясните результат:*
PostgreSQL использует Index Scan по B-tree индексу t_lookup_value_idx, созданному по колонке item_value.
Так как в запросе выполняется точное сравнение по значению, индекс позволяет напрямую определить потенциальные строки без последовательного сканирования таблицы.

Запрос не возвращает строк (rows=0), что означает отсутствие значения 'T_BOOKS' в таблице.
При этом использование индекса обеспечивает минимальные накладные расходы и быстрое выполнение запроса.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.174..0.174 rows=0 loops=1)
  Index Cond: ((item_value)::text = 'T_BOOKS'::text)
Planning Time: 1.212 ms
Execution Time: 0.246 ms

     
     *Объясните результат:*
PostgreSQL использует Index Scan по индексу t_lookup_clustered_value_idx, созданному по колонке item_value.
Условие запроса — точное сравнение по значению, поэтому используется B-tree индекс, а последовательное сканирование таблицы не требуется.

Запрос не возвращает строк (rows=0), что указывает на отсутствие значения 'T_BOOKS' в таблице.
Кластеризация таблицы не даёт дополнительного выигрыша в данном случае, так как поиск выполняется по отдельному индексу, не связанному с физическим порядком строк.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
Поиск по значению item_value в обычной и кластеризованной таблицах выполняется через B-tree индекс и использует Index Scan в обоих случаях.

Кластеризация таблицы не оказывает заметного влияния на производительность, так как поиск выполняется по индексу, не связанному с физическим порядком строк.
Время выполнения запросов сопоставимо, а различия носят несущественный характер и обусловлены накладными расходами планирования и кэша.
