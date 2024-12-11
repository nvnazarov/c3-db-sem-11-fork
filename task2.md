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
    ```
     Bitmap Heap Scan on t_books  (cost=21.03..1335.59 rows=750 width=33) (actual time=0.012..0.012 rows=1 loops=1)
       Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
       Heap Blocks: exact=1
       ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.009..0.009 rows=1 loops=1)
             Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
     Planning Time: 0.186 ms
     Execution Time: 0.030 ms
    (7 rows)
    ```
    
    *Объясните результат:*
    
    Планировщик решил использовать `Bitmap Index Scan`, чтобы определить, на каких страницах точно нет записей со словом `'expert'` в названии, а на каких может быть. Оказалось, что подходящая страница только одна (`Heap Blocks: exact=1`). 

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

    ```
    INSERT 0 150000
    CLUSTER
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
    ```
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.012..0.012 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.099 ms
     Execution Time: 0.021 ms
    (4 rows)
    ```
    
    *Объясните результат:*
    
    Поскольку производится поиск по ключу (который уникален и имеется в индексе по PK), планировщик решил использовать `Index Scan`.

14. Выполните поиск по ключу в кластеризованной таблице:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
    ```
     
    *План выполнения:*
    ```
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.036..0.037 rows=1     loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.070 ms
     Execution Time: 0.045 ms
    (4 rows)
    ```
     
    *Объясните результат:*
    
    В целом ничего не поменялось по сравнению с п. 13, поскольку запрос все равно ищет ровно одну запись, а `CLUSTER` здесь никак не помогает. 

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
    ```
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.015..0.015 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.174 ms
     Execution Time: 0.025 ms
    (4 rows)
    ```
    
    *Объясните результат:*
    
    Планировщик использует `Index Scan` для поиска необходимых записей.

18. Выполните поиск по значению в кластеризованной таблице:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
    ```
    
    *План выполнения:*
    ```
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.012..0.012 rows=0     loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.115 ms
     Execution Time: 0.022 ms
    (4 rows)
    ```
    
    *Объясните результат:*
    
    Аналогично п. 17. В кластеризованной таблице `CLUSTER` был применен для PK (`item_key`), но не для колонки `item_value`, поэтому кластеризация никак не помогла.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
    *Сравнение:*
    
    `CLUSTER` никак не помогает, если производится поиск ровно одной записи (например, поиск по PK) или поиск по колонке, которая не была кластеризована. Так что время поиска в обычной и кластеризованной таблицах в примерах выше не отличается.