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

   ``` markdown
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.012..0.012 rows=0 loops=1)
   Recheck Cond: (category IS NULL)
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.009..0.010 rows=0 loops=1)
         Index Cond: (category IS NULL)
   Planning Time: 0.723 ms
   Execution Time: 0.044 ms
   ```

   *Объясните результат:*

   ``` markdown
   Используется Bitmap Heap Scan с BRIN индексом t_books_brin_cat_idx. В данном случае строк с NULL нет, поэтому план выполняется быстро. Recheck Cond показывает, что после индекса происходит повторная проверка условия
   ```

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

   ``` markdown
   Bitmap Heap Scan on t_books  (cost=12.17..2405.34 rows=1 width=33) (actual time=14.183..14.185 rows=0 loops=1)
   Recheck Cond: ((category)::text = 'INDEX'::text)
   Rows Removed by Index Recheck: 150000
   Filter: ((author)::text = 'SYSTEM'::text)
   Heap Blocks: lossy=1225
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.17 rows=77878 width=0) (actual time=0.291..0.291 rows=12250 loops=1)
         Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.348 ms
   Execution Time: 14.229 ms
   ```

   *Объясните результат (обратите внимание на bitmap scan):*

   ``` markdown
   План показывает Bitmap Heap Scan с использованием BRIN индекса по category. Из-за BRIN индекса происходит lossy сканирование, что означает, что индекс указывает на грубые блоки, а не точные строки. 
   Затем выполняется Recheck Cond и Filter по author, что приводит к проверке 150 000 строк. BRIN индекс недостаточно точен для сложных условий, требуется дополнительная фильтрация в памяти.
   ```

8. Получите список уникальных категорий:

   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```

   *План выполнения:*

   ``` markdown
   Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=34.202..34.204 rows=6 loops=1)
   Sort Key: category
   Sort Method: quicksort  Memory: 25kB
   ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=34.167..34.168 rows=6 loops=1)
         Group Key: category
         Batches: 1  Memory Usage: 24kB
         ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.006..7.368 rows=150000 loops=1)
   Planning Time: 0.136 ms
   Execution Time: 34.271 ms
   ```

   *Объясните результат:*

   ``` markdown
   Используется Seq Scan, так как для получения уникальных значений необходимо просканировать всю таблицу. HashAggregate группирует категории, затем Sort сортирует результат. BRIN индекс не используется, потому что запрос требует полного перебора данных
   ```

9. Подсчитайте книги, где автор начинается на 'S':

   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```

   *План выполнения:*

   ``` markdown
   Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=11.451..11.452 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=11.447..11.448 rows=0 loops=1)
         Filter: ((author)::text ~~ 'S%'::text)
         Rows Removed by Filter: 150000
   Planning Time: 0.275 ms
   Execution Time: 11.470 ms
   ```

   *Объясните результат:*

   ``` markdown
   Seq Scan с фильтром LIKE 'S%'. Индекс по author не используется, потому что LIKE с префиксом неэффективен для BRIN
   ```

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

   ``` markdown
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=47.640..47.641 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=47.633..47.635 rows=1 loops=1)
         Filter: (lower((title)::text) ~~ 'o%'::text)
         Rows Removed by Filter: 149999
   Planning Time: 0.459 ms
   Execution Time: 47.662 ms
   ```

   *Объясните результат:*

   ``` markdown
   Создан индекс по LOWER(title), но план показывает Seq Scan. Скорее всего оптимизатор решил, что полное сканирование быстрее.
   Индекс не используется, возможно из-за статистики или стоимости доступа
   ```

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

   ``` markdown
   Bitmap Heap Scan on t_books  (cost=12.17..2405.34 rows=1 width=33) (actual time=0.836..0.837 rows=0 loops=1)
   Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Rows Removed by Index Recheck: 8849
   Heap Blocks: lossy=73
   ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.17 rows=77878 width=0) (actual time=0.033..0.033 rows=730 loops=1)
         Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.293 ms
   Execution Time: 0.865 ms
   ```

   *Объясните результат:*

   ``` markdown
   Составной BRIN индекс (category, author) используется в Bitmap Index Scan. 
   Время выполнения уменьшилось с 14.229 мс до 0.865 мс. 
   Количество блоков для recheck уменьшилось с 1225 до 73. Это указывает на большую точность составного индекса.
   ```
