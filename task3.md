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
    ```
     Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=10.602..78.590 rows=500291 loops=1)
       Recheck Cond: (category = 'A'::text)
       Heap Blocks: exact=8334
       ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=9.695..9.695 rows=500291 loops=1)
             Index Cond: (category = 'A'::text)
     Planning Time: 0.196 ms
     Execution Time: 90.269 ms
    (7 rows)
    ```
    
    *Объясните результат:*
    
    Планировщик использует типичную комбинацию `Bitmap Heap Scan` и `Bitmap Index Scan` для отсечки лишних страниц и проверки всех записей на оставщихся страницах. В целом понятно, почему других вариантов здесь особо нет - записи с `category = A` разбросаны по таблице неупорядоченно.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    ```
    workshop=# CLUSTER test_cluster USING test_cluster_cat_idx;
    CLUSTER
    ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
     Bitmap Heap Scan on test_cluster  (cost=5520.42..20041.51 rows=494967 width=39) (actual time=8.007..41.119 rows=500291 loops=1)
       Recheck Cond: (category = 'A'::text)
       Heap Blocks: exact=4170
       ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5396.68 rows=494967 width=0) (actual time=7.628..7.629 rows=500291 loops=1)
             Index Cond: (category = 'A'::text)
     Planning Time: 0.129 ms
     Execution Time: 52.396 ms
    (7 rows)
    ```
    
    *Объясните результат:*
    
    Планировщик все еще использует тот же подход с `Bitmap Heap Scan` + `Bitmap Index Scan`, однако количество оставшихся после отсечки страниц в ~2 раза меньше. Это так, поскольку `CLUSTER` упорядочил записи физически на диске, из-за чего записи с `category = A` занимают страницы целиком, то есть более "компактно". 

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    
    Запрос выполнился в ~2 раза быстрее после применения `CLUSTER`, поскольку после физического упорядочивания записей на диске по колонке `category` обрабатывалось ~2 раза меньше страниц (записи с `category = A` занимают страницы целиком, то есть более "компактно").