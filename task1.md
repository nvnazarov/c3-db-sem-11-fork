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
   ```
    Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.007..0.008 rows=0 loops=1)
      Recheck Cond: (category IS NULL)
      ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.006..0.006 rows=0 loops=1)
            Index Cond: (category IS NULL)
    Planning Time: 0.147 ms
    Execution Time: 0.041 ms
   (6 rows)
   ```
   
   *Объясните результат:*
   
   Поскольку BRIN хранит информацию о наличии `NULL` (или не `NULL`) значений в страницах, он может использоваться для отсечения части страниц диска, на которых нет нужной информации (`category is NULL`). Планировщик так и поступил: использовал `Bitmap Index Scan` в комбинации с BRIN для отсечения ненужных страниц, а далее методом `Bitmap Heap Scan` обошел оставшиеся страницы, перепроверяя условие `category is NULL` для каждой записи на странице.

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
   ```
    Bitmap Heap Scan on t_books  (cost=12.15..2297.56 rows=1 width=33) (actual time=11.199..11.200 rows=0 loops=1)
      Recheck Cond: ((category)::text = 'INDEX'::text)
      Rows Removed by Index Recheck: 150000
      Filter: ((author)::text = 'SYSTEM'::text)
      Heap Blocks: lossy=1225
      ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.15 rows=70694 width=0) (actual time=0.036..0.036 rows=12250 loops=1)
            Index Cond: ((category)::text = 'INDEX'::text)
    Planning Time: 0.277 ms
    Execution Time: 11.216 ms
   (9 rows)
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   
   Здесь BRIN использовался только для отсечения страниц, точно не содержащих записи с `category = 'INDEX'`. Скорее всего планировщик посчитал это выгодным, поскольку в колонке `category` мало возможных значений (до 10), то есть BRIN сможет отбросить много страниц.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
    Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=24.694..24.695 rows=6 loops=1)
      Sort Key: category
      Sort Method: quicksort  Memory: 25kB
      ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=24.683..24.684 rows=6 loops=1)
            Group Key: category
            Batches: 1  Memory Usage: 24kB
            ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.004..6.475 rows=150000 loops=1)
    Planning Time: 0.077 ms
    Execution Time: 24.717 ms
   (9 rows)
   ```
   
   *Объясните результат:*

    Здесь все говорит о том, что PostgreSQL ничего не знает про множество значений колонки `category`. Поэтому планировщик проходится по всем записям `t_books`, при этом группируя методом `HashAggregate` по колонке `category`, после чего сортирует полученную последовательность категорий.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
    Aggregate  (cost=3100.03..3100.05 rows=1 width=8) (actual time=9.462..9.463 rows=1 loops=1)
      ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=14 width=0) (actual time=9.459..9.459 rows=0 loops=1)
            Filter: ((author)::text ~~ 'S%'::text)
            Rows Removed by Filter: 150000
    Planning Time: 1.609 ms
    Execution Time: 9.481 ms
   (6 rows)
   ```
   
   *Объясните результат:*
   
   BRIN не умеет индексировать паттерны (то есть не может сказать, есть ли на странице запись с `author LIKE 'S%'`), поэтому планировщик использует `Seq Scan` с фильтром, после чего считает общее число подходящих записей с методом `Aggregate`.

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
   ```
    Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=33.913..33.914 rows=1 loops=1)
      ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=33.907..33.909 rows=1 loops=1)
            Filter: (lower((title)::text) ~~ 'o%'::text)
            Rows Removed by Filter: 149999
    Planning Time: 0.173 ms
    Execution Time: 33.930 ms
   (6 rows)
   ```
   
   *Объясните результат:*
   
   Хотя мы и создали B-Tree индекс на колонке `LOWER(title)`, он не может использоваться для поиска совпадений по паттерну, так что все аналогично пункту 9.

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
   ```
    Bitmap Heap Scan on t_books  (cost=12.15..2297.56 rows=1 width=33) (actual time=0.751..0.751 rows=0 loops=1)
      Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
      Rows Removed by Index Recheck: 8843
      Heap Blocks: lossy=73
      ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.15 rows=70694 width=0) (actual time=0.015..0.015 rows=730 loops=1)
            Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
    Planning Time: 0.113 ms
    Execution Time: 0.766 ms
   (8 rows)
   ```
   
   *Объясните результат:*
   
   Использование составного BRIN позволило не использовать `Filter` после `Recheck Cond`, а также одновременно отбрасывать страницы, точно не содержащие `category = 'INDEX'` или точно не содержащие `author = 'SYSTEM'`. По сравнению с п. 7, составной BRIN отбросил намного больше страниц, оставив только `Heap Blocks: lossy = 73` (в сравнении с 1225 в п. 7).