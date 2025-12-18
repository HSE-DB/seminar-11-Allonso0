# Задание 3

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

    ``` markdown
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=20.806..223.246 rows=500684 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=8334
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=19.611..19.612 rows=500684 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.441 ms
    Execution Time: 241.913 ms
    ```

    *Объясните результат:*

    ``` markdown
    Bitmap Heap Scan с использованием индекса по category.
    Запрос возвращает 500 684 строк, время 241.913 мс.
    До кластеризации данные разбросаны
    ```

4. Выполните кластеризацию:

    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```

    *Результат:*

    ``` markdown
    [2025-12-18 19:45:02] workshop.public> CLUSTER test_cluster USING test_cluster_cat_idx
    [2025-12-18 19:45:04] completed in 1 s 704 ms
    ```

5. Измерьте производительность после кластеризации:

    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```

    *План выполнения:*

    ``` markdown
    Bitmap Heap Scan on test_cluster  (cost=5576.98..20163.47 rows=500200 width=39) (actual time=12.823..91.942 rows=500684 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=4173
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5451.93 rows=500200 width=0) (actual time=12.088..12.088 rows=500684 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.970 ms
    Execution Time: 111.758 ms
    ```

    *Объясните результат:*

    ``` markdown
    После CLUSTER время уменьшилось до 111.758 мс, количество блоков сократилось до 4173. Кластеризация упорядочила данные по категории. Это улучшило производительность
    ```

6. Сравните производительность до и после кластеризации:

    *Сравнение:*

    ``` markdown
    До кластеризации: 241.913 мс, 8334 блоков.
    После кластеризации: 111.758 мс, 4173 блоков.
    Ускорение в 2.2 раза. Кластеризация эффективна для запросов, которые читают много строк по значению индекса
    ```
