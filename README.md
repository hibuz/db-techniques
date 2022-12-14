# π Next-Level Database Techniques For Developers
> Learn secret and lesser-known SQL features and approaches to become a database wizard.


## 1. Data Manipulation
μΌλΆ μ μ μΈ μ»¨νμΈ λ§ μλ μ νλ¦¬μΌμ΄μμ΄λ μ‘΄μ¬νμ§λ§ λλΆλΆ λ°μ΄ν° λ³κ²½μ νμλ‘ ν©λλ€. λ°μ΄ν°λ² μ΄μ€μ INSERT, UPDATE λλ DELETE μΏΌλ¦¬ μ¬μ©μ SQLμμ κ°μ₯ λ³΅μ‘νμ§ μμ κΈ°λ₯μΈ κ²μ²λΌ λ³΄μ΄μ§λ§, μ¬μ ν μ νλ¦¬μΌμ΄μμ κ°μ  ν¬μΈνΈκ° λ  μ μμ΅λλ€. λμ€ν¬κ° 1μ΄μ μνν  μ μλ μ°κΈ° μμμ μλ λ§€μ° μ νμ μ΄λΌλ μ μ ν­μ κΈ°μ΅νμ­μμ€. μ΄λΉ μμ μλ₯Ό μ€μΌ μ μλ€λ©΄ μ νλ¦¬μΌμ΄μμ μ±λ₯μ ν¨μ¬ ν₯μλ©λλ€. 1μ₯μμλ λ€λ₯Έ νμ΄λΈμ μ λ³΄λ₯Ό κΈ°λ°μΌλ‘ νμ μλ°μ΄νΈνκ±°λ μ€λ³΅ νμ μ­μ νκ±°λ μ κΈ κ²½ν©μ μ κ±°νμ¬ μ νλ¦¬μΌμ΄μμ λ λΉ λ₯΄κ² λ§λλ λ°©λ²μ μλ €μ€λλ€. λ§μ§λ§ νμ μ’μ’ μ±λ₯ λ¬Έμ λΌλ κ²μ μμκΈ° λλ¬Έμ μμΈν νμ΅ν΄μΌ ν©λλ€.

### 1.1 Prevent Lock Contention For Updates On Hot Rows
> μ¦μ μλ°μ΄νΈκ° νμν νμ λν μ κΈ κ²½ν© λ°©μ§νκΈ°
```sql
-- MySQL

INSERT INTO tweet_statistics (
  tweet_id, fanout, likes_count
) VALUES (
  1475870220422107137, FLOOR(RAND() * 10), 1
) ON DUPLICATE KEY UPDATE likes_count =
likes_count + VALUES(likes_count);

-- PostgreSQL
INSERT INTO tweet_statistics (
  tweet_id, fanout, likes_count
) VALUES (
  1475870220422107137, FLOOR(RANDOM() * 10), 1
) ON CONFLICT (tweet_id, fanout) DO UPDATE SET likes_count =
tweet_statistics.likes_count + excluded.likes_count;
```

νΈμν°μ νΈμ μΉ΄μ΄ν°κ° μ§μμ μΈ μλ°μ΄νΈλ₯Ό νμλ‘ ν λ, νΈλν½ κΈμ¦ λλ μΈκΈ° νΈμμ κ²½μ° μΉ΄μ΄ν°κ° 1μ΄ μμ μμμ΄ μλ°μ΄νΈλ  μ μμ΅λλ€. μ΄ κ²½μ° λ°μ΄ν°λ² μ΄μ€μ λμμ± μ μ΄λ‘ μΈν΄ ν λ²μ νλμ νΈλμ­μ(μΏΌλ¦¬)λ§ νμ μ κΈ μ μμΌλ―λ‘ μλ°μ΄νΈκ° μλ‘ κ°μ­νκΈ° μμν©λλ€.
λͺ¨λ  μλ°μ΄νΈλ λλ¦½λ νμ λν΄ λ³λ ¬ μ€ν λμ  μ°¨λ‘λ‘ μ€νλ©λλ€. λ¨μΌ νμ μλ°μ΄νΈνλ λμ  μ¦λΆμ μλ₯Ό λ€μ΄ νΉλ³ν μΉ΄μ΄ν° νμ΄λΈμ 100 κ°μ λ€λ₯Έ νμΌλ‘ ν¬ μμλ©λλ€. μ΄μ  scaling factorλ μΉ΄μ΄ν°κ° κΈ°λ‘λλ μΆκ° νμ μμ λ°λΌ μ¦κ°ν©λλ€. μ΄λ¬ν κ°μ λμ€μ λ¨μΌ κ°μΌλ‘ μ§κ³λκ³  μ κΈ κ²½ν©μ΄ λ°μνμ μλ μ΄μ μ μ₯λ©λλ€.

### 1.2 Updates Based On A Select Query
> μλ νΈ μΏΌλ¦¬μ κΈ°λ°ν μλ°μ΄νΈ
```sql
-- MySQL
UPDATE products
JOIN categories USING(category_id)
SET price = price_base - price_base * categories.discount;

-- PostgreSQL
UPDATE products
SET price = price_base - price_base * categories.discount
FROM categories
WHERE products.category_id = categories.category_id;
```
νλμ νμ΄λΈμ΄ λμμ μΌλ‘ μλ°μ΄νΈ λκΈ°λ νμ§λ§ λ€λ₯Έ νμ΄λΈμ μ μ₯λ μ λ³΄λ₯Ό κΈ°λ°μΌλ‘ κ°μ΄ μλ°μ΄νΈ λκΈ°λ ν©λλ€. μλ₯Ό λ€μ΄, λλμ μΈ νμ¬κΈ°κ° λμ λͺ¨λ  μ νμ ν μΈνλ κ²½μ°, κ° μνμ μΉ΄νκ³ λ¦¬λ³ ν μΈμ¨μ μ μ© λ°μ΅λλ€. λͺ¨λ  μΉ΄νκ³ λ¦¬μ λν΄ μλ°μ΄νΈ μΏΌλ¦¬λ₯Ό μ€ννλ λ¨μν λ°©λ² λμ  μνμ μΉ΄νκ³ λ¦¬μ μ‘°μΈνμ¬ μλ°μ΄νΈν  μ μμ΅λλ€. μμ©νλ‘κ·Έλ¨μ μλ μ‘°μΈμ λ°μ΄ν°λ² μ΄μ€μ μν΄ λ³΄λ€ ν¨μ¨μ μΈ μ‘°μΈμΌλ‘ λμ²΄λ©λλ€.

> **:bulb:μ°Έκ³ **  
> λ°μ΄ν°λ² μ΄μ€ μ λ¬Έ μΉ μ¬μ΄νΈμΈ SQLFordevs.com: [UPDATE from a SELECT](https://sqlfordevs.com/update-from-select) μμ μ΄ μ£Όμ μ κ΄λ ¨λ μμΈν λ΄μ©μ νμΈν  μ μμ΅λλ€.

### 1.3 Return The Values Of Modified Rows
> λ³κ²½λ νμ λ°ν κ° μ¬μ©
```sql
-- PostgreSQL:
DELETE FROM sessions
WHERE ip = '127.0.0.1'
RETURNING id, user_agent, last_access;
```
λλΆλΆμ μ μ§λ³΄μ μμμ νΉμ  νμ μ°Ύμ μ²λ¦¬(μ: μ΄λ©μΌ μ μ‘ λλ μΌλΆ ν΅κ³ κ³μ°)νκ³  μ²λ¦¬λ νμΌλ‘ νμνλ κ²μ κΈ°λ°μΌλ‘ ν©λλ€. μΌλ°μ μΌλ‘ ν λ΄μ νλκ·Έλ λ μ΄μ νμνμ§ μμΌλ―λ‘ μλ°μ΄νΈλκ±°λ μ­μ λ©λλ€. RETURNING κΈ°λ₯μ μ¬μ©νμ¬ λ°μ΄ν° μ‘°μ λ° λ°μ΄ν° μ νμ ν λ¨κ³λ‘ μνν΄μ μ²λ¦¬ μν¬ νλ‘μ°λ₯Ό λ¨μν ν  μ μμ΅λλ€. μ΄ κΈ°λ₯μ DELETE, INSERT λ° UPDATE μΏΌλ¦¬μ μ¬μ©ν  μ μμΌλ©° λ°μ΄ν° λ³κ²½μ΄ μλ£λ ν λ°μ΄ν°(μλ₯Όλ€λ©΄ λͺ¨λ  νΈλ¦¬κ±°κ° μ€νλκ³  μμ±λ κ°μ΄ μ¬μ© κ°λ₯ν Inserted λλ Updated λ°μ΄ν°)λ₯Ό λ°νν©λλ€.

> **:bulb:μ°Έκ³ **  
> μ΄ κΈ°λ₯μ PostgreSQLμμλ§ κ°λ₯ν©λλ€.

### 1.4 Delete Duplicate Rows
> μ€λ³΅ ν μ­μ 

```sql
-- MySQL
WITH duplicates AS (
  SELECT id, ROW_NUMBER() OVER(
    PARTITION BY firstname, lastname, email
    ORDER BY age DESC
  ) AS rownum
  FROM contacts
)
DELETE contacts
FROM contacts
JOIN duplicates USING(id)
WHERE duplicates.rownum > 1;

-- PostgreSQL
WITH duplicates AS (
  SELECT id, ROW_NUMBER() OVER(
    PARTITION BY firstname, lastname, email
    ORDER BY age DESC
  ) AS rownum
  FROM contacts
)
DELETE FROM contacts
USING duplicates
WHERE contacts.id = duplicates.id AND duplicates.rownum > 1;
```
μκ°μ΄ μ§λλ©΄ λλΆλΆμ μ νλ¦¬μΌμ΄μμμ νμ΄ μ€λ³΅λμ΄ μ¬μ©μ κ²½νμ΄ λλΉ μ§κ³  μ€ν λ¦¬μ§ μκ΅¬ μ¬ν­μ΄ μ¦κ°νλ©° λ°μ΄ν°λ² μ΄μ€ μ±λ₯μ΄ μ νλ©λλ€. CTE(Common Table Expression)μ μ¬μ©νμ¬ μ€λ³΅ νμ μ€μλλ³λ‘ μλ³νκ³  μ λ ¬νμ¬ λ³΄κ΄ν  μ μμ΅λλ€. λ¨μΌ μ­μ  μΏΌλ¦¬λ μ΄νμ λ³΄κ΄ν  νΉμ  κ°μλ₯Ό μ μΈν λͺ¨λ  μ€λ³΅ ν­λͺ©μ μ­μ ν  μ μμ΅λλ€. μ΄λ¬ν λ³΅μ‘ν λ‘μ§μ νλμ λ¨μν SQL μΏΌλ¦¬λ‘ μνν©λλ€.

> **:bulb:μ°Έκ³ **  
> * CTEλ μλΈμΏΌλ¦¬λ‘ μ°μ΄λ νμνμ΄λΈ(derived table)κ³Ό λΉμ·ν κ°λμΌλ‘ μ¬μ©λ©λλ€.  
> CTE κ΅¬λ¬Έμ΄ WITHλ‘ μμν΄μ WITH μ μ΄λΌκ³ λ νκ³  MySQL 8.0μ΄μλΆν° μ§μν©λλ€.  
> CTEλ κΆνμ΄ νμ μκ³  νλμ μΏΌλ¦¬λ¬Έμ΄ λλ λκΉμ§λ§ μ§μλλ μΌνμ± νμ΄λΈμ΄λΌλ μ μ΄ VIEWμλ λ€λ¦λλ€.
> * λ°μ΄ν°λ² μ΄μ€ μ λ¬Έ μΉ μ¬μ΄νΈμΈ SQLFordevs.com: [Delete Duplicate Rows](https://sqlfordevs.com/delete-duplicate-rows) μμ μ΄ μ£Όμ μ κ΄λ ¨λ μμΈν λ΄μ©μ νμΈν  μ μμ΅λλ€.

### 1.5 Table Maintenance After Bulk Modifications
> λ²ν¬ μμ  ν νμ΄λΈ μ μ§κ΄λ¦¬

```sql
-- MySQL
ANALYZE TABLE users;

-- PostgreSQL
ANALYZE SKIP_LOCKED users;
```
λ°μ΄ν°λ² μ΄μ€λ μΏΌλ¦¬λ₯Ό μ€ννλ κ°μ₯ ν¨μ¨μ μΈ λ°©λ²μ κ³μ°νκΈ° μν΄ λλ΅μ μΈ ν μ, κ°μ λ°μ΄ν° λΆν¬ λ± νμ΄λΈμ λν μ΅μ  ν΅κ³κ° νμν©λλ€. λ°μ΄ν°μ μν₯μ λ―ΈμΉλ νμ΄ μμ±, μλ°μ΄νΈ λλ μ­μ  λ  λλ§λ€ μλμΌλ‘ λ³κ²½λλ μΈλ±μ€μ λ¬λ¦¬ λͺ¨λ  λ³κ²½μλ§λ€ ν΅κ³λ κ³μ°λμ§ μμ΅λλ€. μ¬ κ³μ°μ νμ΄λΈμ λν λ³κ²½ μκ³ κ°μ μ΄κ³Όν  λλ§ νΈλ¦¬κ±°λ©λλ€. νμ΄λΈμ ν° λΆλΆμ λ³κ²½ν  λλ§λ€ μν₯μ λ°λ νμ μλ μ¬μ ν ν΅κ³ μ¬κ³μ° μκ³κ°λ³΄λ€ λ?μ§λ§ ν΅κ³κ° λΆμ νν΄μ§ μ λμ μλ―Έκ° μμ μ μμ΅λλ€. λ°μ΄ν°λ² μ΄μ€κ° νμ΄λΈμ λν μλͺ»λ μ λ³΄λ₯Ό κΈ°λ°μΌλ‘ μ΅μμ μΏΌλ¦¬ κ³νμ μμΈ‘νκΈ° λλ¬Έμ μΌλΆ μΏΌλ¦¬λ λ§€μ° λλ €μ§ μ μμ΅λλ€. λ°λΌμ μ€μν λ³κ²½μ΄ μμ λ ν΅κ³ μ¬κ³μ°μ νΈλ¦¬κ±°νκΈ° μν΄ ANALYZE TABLEμ νλ©΄ μΏΌλ¦¬ μλλ₯Ό λμΌ μ μμ΅λλ€.

## 2. Querying Data
μμ±νκ³  μ€ννλ λλΆλΆμ SQL μΏΌλ¦¬λ λ°μ΄ν°λ² μ΄μ€μμ λ°μ΄ν°λ₯Ό μ½λ μΏΌλ¦¬μλλ€. λ°μ΄ν°λ₯Ό νμνμ§ μμΌλ©΄ μ νλ¦¬μΌμ΄μμ΄ μΈλͺ¨μκ² λλ―λ‘ μ νλ¦¬μΌμ΄μμ κΈ°λ³Έμ΄ λ©λλ€. κ·Έλ¬λ λ³΄λ€ μ κ΅ν μΏΌλ¦¬ μ κ·Ό λ°©μμ μ¬μ©νμ¬ λ§μ μ νλ¦¬μΌμ΄μμ λ³΄μΌλ¬νλ μ΄νΈλ₯Ό μ κ±°ν  μ μλ κ°μ₯ μ’μ κΈ°νμ΄κΈ°λ ν©λλ€. λλΆλΆμ μ¬λ‘λ₯Ό λ³΄λ©΄ μ΄λ¬ν μ κ·Ό λ°©μμ λ°μ΄ν°λ₯Ό λͺ¨λ μ νλ¦¬μΌμ΄μμΌλ‘ λ³΄λ΄μ§ μκ³  λ°μ΄ν°κ° μλ κ³³μμ λ°μ΄ν°λ₯Ό μ²λ¦¬ν  λ μ±λ₯μ ν₯μμν΅λλ€. μ΄ μ±ν°λ SQL λ΄μ for-each λ£¨ν, λͺ κ°μ§ null μ²λ¦¬ νΈλ¦­, νμ΄μ§ μ§μ  μ€μμ κ°μ μμΈμ μΈ κΈ°λ₯μ λ³΄μ¬μ€ κ²μλλ€. CTEλ₯Ό μ¬μ©νμ¬ λ°μ΄ν° μ μ (refinement) νμ λ§€μ° μμΈν μ½κ³  λμ μ΄ν΄νλ€λ©΄ μμ£Ό μ¬μ©ν  μ μμ΅λλ€.

### 2.1 Reduce The Amount Of Group By Columns
> Group By μ»¬λΌ κ°―μ μ€μ΄κΈ°

```sql
SELECT actors.firstname, actors.lastname, COUNT(*) as count
FROM actors
JOIN actors_movies USING(actor_id)
GROUP BY actors.id
```

μΌλΆ μ΄μ κ·Έλ£Ήνν λλ λͺ¨λ  SELECT μ΄μ GROUP BYμ μΆκ°ν΄μΌ νλ€λ κ²μ μ€λμ μ λ°°μ μ κ²μλλ€. κ·Έλ¬λ PKλ‘ GROUP BY νλ©΄ λ°μ΄ν°λ² μ΄μ€κ° μλμΌλ‘ μΆκ°νλ―λ‘ λμΌν νμ΄λΈμ λͺ¨λ  μ΄μ μλ΅ν  μ μμ΅λλ€. κ²°κ³Όμ μΌλ‘ μΏΌλ¦¬κ° μ§§μμ Έμ μ½κ³  μ΄ν΄νκΈ°κ° λ μ¬μμ§λλ€.

### 2.2 Fill Tables With Large Amounts Of Test Data
> λλμ νμ€νΈ λ°μ΄ν° λ§λ€κΈ°

```sql
-- MySQL
SET cte_max_recursion_depth = 4294967295;
INSERT INTO contacts (firstname, lastname)
WITH RECURSIVE counter(n) AS(
  SELECT 1 AS n
  UNION ALL
  SELECT n + 1 FROM counter WHERE n < 100000
)
SELECT CONCAT('firstname-', counter.n), CONCAT('lastname-', counter.n)
FROM counter

-- PostgreSQL
INSERT INTO contacts (firstname, lastname)
SELECT CONCAT('firstname-', i), CONCAT('lastname-', i)
FROM generate_series(1, 100000) as i;
```

λλ‘λ μ±λ₯ νμ€νΈλ₯Ό μν΄ νμ€νΈμ© νμ΄λΈμ λ°μ΄ν°λ₯Ό μ€λΉν΄μΌ ν©λλ€. μ΄ λ°μ΄ν°λ λ³΄ν΅ fake λ°μ΄ν° μμ± λΌμ΄λΈλ¬λ¦¬λ₯Ό μ°κ±°λ λ§μ μ½λλ‘ νμ€νΈ λ°μ΄ν°λ₯Ό νμ€μ μΌλ‘ λ§λ€κΈ°λ ν©λλ€. νμ§λ§ λ§μ νμ€νΈ λ°μ΄ν°λ₯Ό μ²μ²ν νλμ© insertνλ λμ  μ΄λ¬ν μΏΌλ¦¬λ‘ λ§μ λλ―Έ λ°μ΄ν°λ₯Ό μμ±νμ¬ μΈλ±μ€μ ν¨μ¨μ±μ νμ€νΈ ν  μ μμ΅λλ€.

> **:warning:μ£Όμ**  
> μλ―Έ μλ λ²€μΉλ§ν¬λ₯Ό μν΄μλ νμ€μ μΈ λ°μ΄ν°μ κ° λΆν¬κ° νμν©λλ€. κ·Έλ¬λ λΉμ μ μΏΌλ¦¬μ μν₯μ λ°μ§ μλ λ λ§μ νμ€νΈ λ°μ΄ν° μμ±μ μν΄ μ΄ λ°©λ²μ μ¬μ©ν  μ μμ΅λλ€.


### 2.3 Simplified Inequality Checks With Nullable Columns
> Nullable μ»¬λΌμ λν κ°λ¨ν λΆλ±μ κ²μ¬

```sql
-- MySQL
SELECT * FROM example WHERE NOT(column <> 'value');

-- PostgreSQL
SELECT * FROM example WHERE column IS DISTINCT FROM 'value';
```

Nullable μ»¬λΌμ λν΄ νΉμ  κ°κ³Ό κ°μ§ μμ λ°μ΄ν°λ₯Ό κ²μνλ κ²μ λ³΅μ‘ν©λλ€. λ§μ΄ μ¬μ©νλ col != 'value' μ‘°κ±΄μ μ null κ°μ λν΄μλ μ μ©λμ§ μμ κ²°κ³Όμ ν¬ν¨λμ§ μμ΅λλ€. κ·Έλμ ν­μ μνλ κ²°κ³Όλ₯Ό μ»μΌλ €κ³  λ λ³΅μ‘ν (col IS NULL OR col!= 'value') μ‘°κ±΄μ μ μ¬μ©ν©λλ€. λ€ννλ λ λ°μ΄ν°λ² μ΄μ€ λͺ¨λ null μ²λ¦¬λ₯Ό ν¬ν¨νλ λΆλ±μ κ²μ¬ κΈ°λ₯μ μ§μν©λλ€.

### 2.4 Prevent Division By Zero Errors
> 0μΌλ‘ λλκΈ° μλ¬ λ°©μ§

```sql
SELECT visitors_today / NULLIF(visitors_yesterday, 0)
FROM logs_aggregated;
```

λ°μ΄ν°λ² μ΄μ€ ν΅κ³ μμμ μ΄λ ΅μ§ μκ³  μλ§ μλ°± λ²μ μννμ κ²μλλ€. κ·Έλ¬λ μΏΌλ¦¬λ₯Ό μμ±ν  λ μνν κ°μ μ΄ λ μ΄μ μ ν¨νμ§ μμ λͺ λ¬ νμ μ΄λ¬ν μΏΌλ¦¬μμ μλ¬κ° λ°μνμ μ μμ΅λλ€. μλ§λ μ¬μ΄νΈκ° λ€μ΄λμ΄ νΉμ ν λ μ λ°©λ¬Έμκ° μμκ±°λ, μ΄μ  μ²μμΌλ‘ μ¨λΌμΈ μ€ν μ΄ νλ§€ κ±΄μκ° μμ μλ μμ΅λλ€. κ·Έλ μλ λ°μ΄ν°κ° μμ΄ SUM(visitors_yesterday)μΌλ‘ λλμ κ³μ° μ μλ¬κ° λ°μνκ² λ©λλ€. λ°λΌμ μΌλΆ λ°μ΄ν°κ° λλ½λ κ²½μ°λ₯Ό κ³ λ €νμ¬ ν­μ 0μΌλ‘ λλμ§ μλλ‘ ν΄μΌ ν©λλ€. λλλ μλ₯Ό 0μμ Null κ°μΌλ‘ λ³ννλ©΄ ν΄λΉ λ¬Έμ κ° ν΄κ²°λ©λλ€.

### 2.5 Sorting Order With Nullable Columns
### 2.6 Deterministic Ordering for Pagination
### 2.7 More Efficient Pagination Than LIMIT OFFSET
### 2.8 Database-Backed Locks With Safety Guarantees
### 2.9 Refinement Of Data With Common Table Expressions
### 2.10 First Row Of Many Similar Ones
### 2.11 Multiple Aggregates In One Query
### 2.12 Limit Rows Also Including Ties
### 2.13 Fast Row Count Estimates
### 2.14 Date-Based Statistical Queries With Gap-Filling
### 2.15 Table Joins With A For-Each Loop

## 3. Schema

### 3.1 Rows Without Overlapping Dates
### 3.2 Store Trees As Materialized Paths
### 3.3 JSON Columns to Combine NoSQL and Relational Databases
### 3.4 Alternative Tag Storage With JSON Arrays
### 3.5 Constraints for Improved Data Strictness
### 3.6 Validation Of JSON Colums Against A Schema
### 3.7 UUID Keys Against Enumeration Attacks
### 3.8 Fast Delete Of Big Data With Partitions
### 3.9 Pre-Sorted Tables For Faster Access
### 3.10 Pre-Aggregation of Values for Faster Queries

## 4. Indexes

### 4.1 Indexes On Functions And Expressions
### 4.2 Find Unused Indexes
### 4.3 Safely Deleting Unused Indexes
### 4.4 Index-Only Operations By Including More Columns
### 4.5 Partial Indexes To Reduce Index Size
### 4.6 Partial Indexes For Uniqueness Constraints
### 4.7 Index Support For Wildcard Searches
### 4.8 Rules For Multi-Column Indexes
### 4.9 Hash Indexes To Descrease Index Size
### 4.10 Descending Indexes For Order By
### 4.11 Ghost Conditions Against Unindexed Columns


μΆμ² : [SqlForDevs.com](https://sqlfordevs.com/ebook)

