## Prepare data

```sql
db=# insert into test select generate_series(1,1000000) as first, (random() * 10000) as second, md5(random()::text) as note;
INSERT 0 1000000
```

```sql
db=# select count(1) from test;
  count  
---------
 1000000
(1 row)
```

```sql
CREATE INDEX first_second_idx ON test (first, second)
```
## AND Condition

```sql
db=# explain analyze select * from test where first = 3291 and second = 123;
                                             QUERY PLAN                                              
-----------------------------------------------------------------------------------------------------
 Seq Scan on test  (cost=0.00..24346.00 rows=1 width=41) (actual time=91.999..91.999 rows=0 loops=1)
   Filter: ((first = 3291) AND (second = 123))
   Rows Removed by Filter: 1000000
 Planning time: 0.045 ms
 Execution time: 92.013 ms
(5 rows)
```

```sql
drdb=# explain analyze select * from test where second = 3291 and first = 123;
                                                       QUERY PLAN                                                       
------------------------------------------------------------------------------------------------------------------------
 Index Scan using first_second_idx on test  (cost=0.42..8.45 rows=1 width=41) (actual time=0.025..0.025 rows=0 loops=1)
   Index Cond: ((first = 123) AND (second = 3291))
 Planning time: 0.063 ms
 Execution time: 0.040 ms
(4 rows)
```

When `AND` is used in `WHERE` clause, the order of first and second is not important, `WHERE second = ? And first = ?` will be transformed to `WHERE first = ? AND second = ?`.

What if we have more than 2 columns in the index, as below:

```sql
create table test2 (first int, second int, third int, note text);
```

```sql
create index fst_idx on test2 (first, second, third);
```

```sql
db=# explain analyze select * from test2 where second = 100 and first = 2849 and third = 31323;
                                                   QUERY PLAN                                                   
----------------------------------------------------------------------------------------------------------------
 Index Scan using fst_idx on test2  (cost=0.42..8.45 rows=1 width=45) (actual time=0.012..0.012 rows=0 loops=1)
   Index Cond: ((first = 2849) AND (second = 100) AND (third = 31323))
 Planning time: 0.067 ms
 Execution time: 0.027 ms
(4 rows)
```

```sql
drdb=# explain analyze select * from test2 where first = 2849 and second = 100 and third = 31323;
                                                   QUERY PLAN                                                   
----------------------------------------------------------------------------------------------------------------
 Index Scan using fst_idx on test2  (cost=0.42..8.45 rows=1 width=45) (actual time=0.012..0.012 rows=0 loops=1)
   Index Cond: ((first = 2849) AND (second = 100) AND (third = 31323))
 Planning time: 0.062 ms
 Execution time: 0.026 ms
(4 rows)
```

Although the order of columns in WHERE clause is different from the order in index, PostgreSQL will reorder the conditions of query
and use the index to search. So the order of columns is not important when we use `AND` and equality in `WHERE` clause.

## Range queries

Index (first, second)

```sql
db=# explain analyze select * from test where first >= 100000 and first <= 200000 and second = 500;
                                                         QUERY PLAN                                                          
-----------------------------------------------------------------------------------------------------------------------------
 Index Scan using first_second_idx on test  (cost=0.42..2335.49 rows=10 width=41) (actual time=0.587..3.568 rows=11 loops=1)
   Index Cond: ((first >= 100000) AND (first <= 200000) AND (second = 500))
 Planning time: 0.067 ms
 Execution time: 3.586 ms
(4 rows)
```

Now we use range condition on second column, equality on first column

```sql
db=# explain analyze select * from test where first = 100000 and second >= 100 and second <= 500;
                                                       QUERY PLAN                                                       
------------------------------------------------------------------------------------------------------------------------
 Index Scan using first_second_idx on test  (cost=0.42..8.45 rows=1 width=41) (actual time=0.012..0.012 rows=0 loops=1)
   Index Cond: ((first = 100000) AND (second >= 100) AND (second <= 500))
 Planning time: 0.090 ms
 Execution time: 0.028 ms
(4 rows)
```

This query runs 128 times faster than the previous one. Let's add an (second, first) index and run the first query again.

```sql
create index second_first_idx on test (second, first);
```

```sql
db=# explain analyze select * from test where first >= 100000 and first <= 200000 and second = 500;
                                                        QUERY PLAN                                                         
---------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on test  (cost=4.55..43.75 rows=10 width=41) (actual time=0.021..0.032 rows=11 loops=1)
   Recheck Cond: ((second = 500) AND (first >= 100000) AND (first <= 200000))
   Heap Blocks: exact=11
   ->  Bitmap Index Scan on second_first_idx  (cost=0.00..4.55 rows=10 width=0) (actual time=0.014..0.014 rows=11 loops=1)
         Index Cond: ((second = 500) AND (first >= 100000) AND (first <= 200000))
 Planning time: 0.079 ms
 Execution time: 0.052 ms
(7 rows)
```

After add (second, first) index, the query uses this index and runs 68 times faster. So we can say that we should use equality comparison on the leftmost column in index.

(to be continued)
