# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.072..0.073 rows=0 loops=1)
  Recheck Cond: (category IS NULL)
  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.055..0.055 rows=0 loops=1)
        Index Cond: (category IS NULL)
Planning Time: 1.120 ms
Execution Time: 0.179 ms

   
   *Объясните результат:*
PostgreSQL использует BRIN-индекс t_books_brin_cat_idx для определения диапазонов страниц, которые могут содержать NULL в колонке category.
Результат индексного сканирования формируется в виде bitmap.
Далее выполняется bitmap heap scan с повторной проверкой условия, так как BRIN-индекс не содержит точных ссылок на строки.
Подходящих строк не найдено, при этом полного сканирования таблицы не выполняется.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=29.636..29.636 rows=0 loops=1)
  Recheck Cond: ((category)::text = 'INDEX'::text)
  Rows Removed by Index Recheck: 150000
  Filter: ((author)::text = 'SYSTEM'::text)
  Heap Blocks: lossy=1225
  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.268..0.269 rows=12250 loops=1)
        Index Cond: ((category)::text = 'INDEX'::text)
Planning Time: 1.196 ms
Execution Time: 29.753 ms
   
   *Объясните результат (обратите внимание на bitmap scan):*
PostgreSQL использует bitmap index scan по BRIN-индексу t_books_brin_cat_idx для условия category = 'INDEX'.
BRIN-индекс возвращает крупные диапазоны страниц, поэтому bitmap получается lossy (Heap Blocks: lossy).

На этапе Bitmap Heap Scan выполняется:

 - повторная проверка условия по category (Recheck Cond);

 - дополнительная фильтрация по author = 'SYSTEM', так как BRIN-индекс по автору не используется в плане.

Из-за неточности BRIN-индекса происходит большое число повторных проверок строк (Rows Removed by Index Recheck: 150000), что увеличивает время выполнения запроса.
Bitmap scan выбран, так как он позволяет объединить результаты индексного отбора и затем фильтровать данные на уровне таблицы без полного sequential scan.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
Sort  (cost=3100.11..3100.12 rows=5 width=7) (actual time=38.730..38.732 rows=6 loops=1)
  Sort Key: category
  Sort Method: quicksort  Memory: 25kB
  ->  HashAggregate  (cost=3100.00..3100.05 rows=5 width=7) (actual time=38.486..38.487 rows=6 loops=1)
        Group Key: category
        Batches: 1  Memory Usage: 24kB
        ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.095..15.339 rows=150000 loops=1)
Planning Time: 1.021 ms
Execution Time: 39.061 ms

   
   *Объясните результат:*
PostgreSQL выполняет последовательное сканирование таблицы (Seq Scan), так как для получения всех уникальных значений category необходимо просмотреть все строки таблицы.

Уникальность обеспечивается оператором HashAggregate, который группирует значения category в хеш-таблице.
После агрегации выполняется Sort, так как запрос содержит ORDER BY category.

BRIN-индексы не используются, поскольку они предназначены для фильтрации по условиям, а не для получения множества уникальных значений.
При малом количестве уникальных категорий хеш-агрегация является более эффективной, чем индексное сканирование.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=17.358..17.360 rows=1 loops=1)
  ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=17.349..17.350 rows=0 loops=1)
        Filter: ((author)::text ~ ~ 'S%'::text)
        Rows Removed by Filter: 150000
Planning Time: 0.908 ms
Execution Time: 17.544 ms

   
   *Объясните результат:*
PostgreSQL выполняет последовательное сканирование таблицы (Seq Scan), так как условие author LIKE 'S%' не может эффективно использовать BRIN-индекс.

BRIN-индексы работают по диапазонам страниц и не подходят для префиксного поиска по строкам.
В ходе сканирования каждая строка проверяется фильтром, при этом все строки были отброшены (Rows Removed by Filter: 150000).

После применения фильтра выполняется агрегатная операция Aggregate для подсчёта количества подходящих строк.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=34.282..34.283 rows=1 loops=1)
  ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=34.276..34.277 rows=1 loops=1)
        Filter: (lower((title)::text) ~ ~ 'o%'::text)
        Rows Removed by Filter: 149999
Planning Time: 1.408 ms
Execution Time: 34.463 ms

   
   *Объясните результат:*
Несмотря на наличие функционального индекса LOWER(title), PostgreSQL выполняет последовательное сканирование таблицы (Seq Scan).

Причина в том, что условие LOWER(title) LIKE 'o%' в данном плане не было сопоставлено с индексом, поэтому фильтрация выполняется построчно.
В процессе сканирования почти все строки были отброшены фильтром (Rows Removed by Filter: 149999).

После применения фильтра выполняется агрегатная операция Aggregate для подсчёта количества подходящих строк.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=2.080..2.081 rows=0 loops=1)
  Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
  Rows Removed by Index Recheck: 8850
  Heap Blocks: lossy=73
  ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.084..0.084 rows=730 loops=1)
        Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
Planning Time: 1.295 ms
Execution Time: 2.189 ms

   
   *Объясните результат:*
PostgreSQL использует составной BRIN-индекс t_books_brin_cat_auth_idx, который одновременно учитывает условия по category и author.
Это позволяет значительно сузить количество подходящих диапазонов страниц ещё на этапе Bitmap Index Scan.

По сравнению с шагом 7:

количество страниц в bitmap существенно меньше (Heap Blocks: lossy=73 против 1225);

число строк, отброшенных при повторной проверке, уменьшилось (Rows Removed by Index Recheck: 8850);

общее время выполнения запроса значительно сократилось.

Bitmap heap scan выполняет повторную проверку условий, так как BRIN-индекс остаётся неточным, однако составной индекс заметно повышает селективность и эффективность выполнения запроса.
