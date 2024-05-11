# 📙 Next-Level Database Techniques For Developers
> Learn secret and lesser-known SQL features and approaches to become a database wizard.


## 1. Data Manipulation
일부 정적인 컨텐츠만 있는 애플리케이션이도 존재하지만 대부분 데이터 변경을 필요로 합니다. 데이터베이스의 INSERT, UPDATE 또는 DELETE 쿼리 사용은 SQL에서 가장 복잡하지 않은 기능인 것처럼 보이지만, 여전히 애플리케이션의 개선 포인트가 될 수 있습니다. 디스크가 1초에 수행할 수 있는 쓰기 작업의 수는 매우 제한적이라는 점을 항상 기억하십시오. 초당 작업 수를 줄일 수 있다면 애플리케이션의 성능은 훨씬 향상됩니다. 1장에서는 다른 테이블의 정보를 기반으로 행을 업데이트하거나 중복 행을 삭제하거나 잠금 경합을 제거하여 애플리케이션을 더 빠르게 만드는 방법을 알려줍니다. 마지막 팁은 종종 성능 문제라는 것을 알았기 때문에 자세히 학습해야 합니다.

### 1.1 Prevent Lock Contention For Updates On Hot Rows
> 잦은 업데이트가 필요한 행에 대한 잠금 경합 방지하기
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
트위터의 트윗 카운터가 지속적인 업데이트를 필요로 할때, 트래픽 급증 또는 인기 트윗의 경우 카운터가 1초 안에 수없이 업데이트될 수 있습니다. 이 경우 데이터베이스의 동시성 제어로 인해 한 번에 하나의 트랜잭션(쿼리)만 행을 잠글 수 있으므로 업데이트가 서로 간섭하기 시작합니다.
모든 업데이트는 독립된 행에 대해 병렬 실행 대신 차례로 실행됩니다. 단일 행을 업데이트하는 대신 증분은 예를 들어 특별한 카운터 테이블의 100 개의 다른 행으로 팬 아웃됩니다. 이제 scaling factor는 카운터가 기록되는 추가 행의 수에 따라 증가합니다. 이러한 값은 나중에 단일 값으로 집계되고 잠금 경합이 발생했을 원래 열에 저장됩니다.

### 1.2 Updates Based On A Select Query
> 셀렉트 쿼리에 기반한 업데이트
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
하나의 테이블이 독자적으로 업데이트 되기도 하지만 다른 테이블에 저장된 정보를 기반으로 값이 업데이트 되기도 합니다. 예를 들어, 대대적인 행사기간 동안 모든 제품을 할인하는 경우, 각 상품은 카테고리별 할인율을 적용 받습니다. 모든 카테고리에 대해 업데이트 쿼리를 실행하는 단순한 방법 대신 상품을 카테고리와 조인하여 업데이트할 수 있습니다. 응용프로그램의 수동 조인은 데이터베이스에 의해 보다 효율적인 조인으로 대체됩니다.

> **:bulb:참고**  
> 데이터베이스 전문 웹 사이트인 SQLFordevs.com: [UPDATE from a SELECT](https://sqlfordevs.com/update-from-select) 에서 이 주제와 관련된 자세한 내용을 확인할 수 있습니다.

### 1.3 Return The Values Of Modified Rows
> 변경된 행의 반환 값 사용
```sql
-- PostgreSQL:
DELETE FROM sessions
WHERE ip = '127.0.0.1'
RETURNING id, user_agent, last_access;
```
대부분의 유지보수 작업은 특정 행을 찾아 처리(예: 이메일 전송 또는 일부 통계 계산)하고 처리된 행으로 표시하는 것을 기반으로 합니다. 일반적으로 행 내의 플래그는 더 이상 필요하지 않으므로 업데이트되거나 삭제됩니다. RETURNING 기능을 사용하여 데이터 조작 및 데이터 선택을 한 단계로 수행해서 처리 워크 플로우를 단순화 할 수 있습니다. 이 기능은 DELETE, INSERT 및 UPDATE 쿼리에 사용할 수 있으며 데이터 변경이 완료된 후 데이터(예를들면 모든 트리거가 실행되고 생성된 값이 사용 가능한 Inserted 또는 Updated 데이터)를 반환합니다.

> **:bulb:참고**  
> 이 기능은 PostgreSQL에서만 가능합니다.

### 1.4 Delete Duplicate Rows
> 중복 행 삭제

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
시간이 지나면 대부분의 애플리케이션에서 행이 중복되어 사용자 경험이 나빠지고 스토리지 요구 사항이 증가하며 데이터베이스 성능이 저하됩니다. CTE(Common Table Expression)을 사용하여 중복 행을 중요도별로 식별하고 정렬하여 보관할 수 있습니다. 단일 삭제 쿼리는 이후에 보관할 특정 개수를 제외한 모든 중복 항목을 삭제할 수 있습니다. 이러한 복잡한 로직을 하나의 단순한 SQL 쿼리로 수행합니다.

> **:bulb:참고**  
> * CTE는 서브쿼리로 쓰이는 파생테이블(derived table)과 비슷한 개념으로 사용됩니다.  
> CTE 구문이 WITH로 시작해서 WITH 절이라고도 하고 MySQL 8.0이상부터 지원합니다.  
> CTE는 권한이 필요 없고 하나의 쿼리문이 끝날때까지만 지속되는 일회성 테이블이라는 점이 VIEW와는 다릅니다.
> * 데이터베이스 전문 웹 사이트인 SQLFordevs.com: [Delete Duplicate Rows](https://sqlfordevs.com/delete-duplicate-rows) 에서 이 주제와 관련된 자세한 내용을 확인할 수 있습니다.

### 1.5 Table Maintenance After Bulk Modifications
> 벌크 수정 후 테이블 유지관리

```sql
-- MySQL
ANALYZE TABLE users;

-- PostgreSQL
ANALYZE SKIP_LOCKED users;
```
데이터베이스는 쿼리를 실행하는 가장 효율적인 방법을 계산하기 위해 대략적인 행 수, 값의 데이터 분포 등 테이블에 대한 최신 통계가 필요합니다. 데이터에 영향을 미치는 행이 생성, 업데이트 또는 삭제 될 때마다 자동으로 변경되는 인덱스와 달리 모든 변경시마다 통계는 계산되지 않습니다. 재 계산은 테이블에 대한 변경 임계 값을 초과할 때만 트리거됩니다. 테이블의 큰 부분을 변경할 때마다 영향을 받는 행의 수는 여전히 통계 재계산 임계값보다 낮지만 통계가 부정확해질 정도의 의미가 있을 수 있습니다. 데이터베이스가 테이블에 대한 잘못된 정보를 기반으로 최상의 쿼리 계획을 예측하기 때문에 일부 쿼리는 매우 느려질 수 있습니다. 따라서 중요한 변경이 있을 때 통계 재계산을 트리거하기 위해 ANALYZE TABLE을 하면 쿼리 속도를 높일 수 있습니다.

## 2. Querying Data
작성하고 실행하는 대부분의 SQL 쿼리는 데이터베이스에서 데이터를 읽는 쿼리입니다. 데이터를 표시하지 않으면 애플리케이션이 쓸모없게 되므로 애플리케이션의 기본이 됩니다. 그러나 보다 정교한 쿼리 접근 방식을 사용하여 많은 애플리케이션의 보일러플레이트를 제거할 수 있는 가장 좋은 기회이기도 합니다. 대부분의 사례를 보면 이러한 접근 방식은 데이터를 모두 애플리케이션으로 보내지 않고 데이터가 있는 곳에서 데이터를 처리할 때 성능을 향상시킵니다. 이 챕터는 SQL 내의 for-each 루프, 몇 가지 null 처리 트릭, 페이지 지정 실수와 같은 예외적인 기능을 보여줄 것입니다. CTE를 사용하여 데이터 정제(refinement) 팁을 매우 자세히 읽고 나서 이해한다면 자주 사용할 수 있습니다.

### 2.1 Reduce The Amount Of Group By Columns
> Group By 컬럼 갯수 줄이기

```sql
SELECT actors.firstname, actors.lastname, COUNT(*) as count
FROM actors
JOIN actors_movies USING(actor_id)
GROUP BY actors.id
```
일부 열을 그룹화할때는 모든 SELECT 열을 GROUP BY에 추가해야 한다는 것을 오래전에 배웠을 것입니다. 그러나 PK로 GROUP BY 하면 데이터베이스가 자동으로 추가하므로 동일한 테이블의 모든 열을 생략할 수 있습니다. 결과적으로 쿼리가 짧아져서 읽고 이해하기가 더 쉬워집니다.

### 2.2 Fill Tables With Large Amounts Of Test Data
> 대량의 테스트 데이터 만들기

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
때로는 성능 테스트를 위해 테스트용 테이블에 데이터를 준비해야 합니다. 이 데이터는 보통 fake 데이터 생성 라이브러리를 쓰거나 많은 코드로 테스트 데이터를 현실적으로 만들기도 합니다. 하지만 많은 테스트 데이터를 천천히 하나씩 insert하는 대신 이러한 쿼리로 많은 더미 데이터를 생성하여 인덱스의 효율성을 테스트 할 수 있습니다.

> **:warning:주의**  
> 의미 있는 벤치마크를 위해서는 현실적인 데이터와 값 분포가 필요합니다. 그러나 당신의 쿼리의 영향을 받지 않는 더 많은 테스트 데이터 생성을 위해 이 방법을 사용할 수 있습니다.

### 2.3 Simplified Inequality Checks With Nullable Columns
> Nullable 컬럼에 대한 간단한 부등식 검사

```sql
-- MySQL
SELECT * FROM example WHERE NOT(column <> 'value');

-- PostgreSQL
SELECT * FROM example WHERE column IS DISTINCT FROM 'value';
```
Nullable 컬럼에 대해 특정 값과 같지 않은 데이터를 검색하는 것은 복잡합니다. 많이 사용하는 col != 'value' 조건절은 null 값에 대해서는 적용되지 않아 결과에 포함되지 않습니다. 그래서 항상 원하는 결과를 얻으려고 더 복잡한 (col IS NULL OR col!= 'value') 조건절을 사용합니다. 다행히도 두 데이터베이스 모두 null 처리를 포함하는 부등식 검사 기능을 지원합니다.

### 2.4 Prevent Division By Zero Errors
> 0으로 나누기 에러 방지

```sql
SELECT visitors_today / NULLIF(visitors_yesterday, 0)
FROM logs_aggregated;
```
데이터베이스 통계 작업은 어렵지 않고 아마 수백 번을 수행했을 것입니다. 그러나 쿼리를 작성할 때 수행한 가정이 더 이상 유효하지 않아 몇 달 후에 이러한 쿼리에서 에러가 발생했을 수 있습니다. 아마도 사이트가 다운되어 특정한 날에 방문자가 없었거나, 어제 처음으로 온라인 스토어 판매 건수가 없을 수도 있습니다. 그날에는 데이터가 없어 SUM(visitors_yesterday)으로 나눗셈 계산 시 에러가 발생하게 됩니다. 따라서 일부 데이터가 누락된 경우를 고려하여 항상 0으로 나누지 않도록 해야 합니다. 나누는 수를 0에서 Null 값으로 변환하면 해당 문제가 해결됩니다.

### 2.5 Sorting Order With Nullable Columns
> Nullable 컬럼의 정렬순서

```sql
-- MySQL: NULL값 처음 배치 (기본)
SELECT * FROM customers ORDER BY country ASC;
SELECT * FROM customers ORDER BY country IS NOT NULL, country ASC;
-- MYSQL: NULL값 마지막 배치
SELECT * FROM customers ORDER BY country IS NULL, country ASC;

-- PostgreSQL: NULL값 처음 배치
SELECT * FROM customers ORDER BY country ASC NULLS FIRST;
-- PostgreSQL: NULL값 마지막 배치 (기본)
SELECT * FROM customers ORDER BY country ASC;
SELECT * FROM customers ORDER BY country ASC NULLS LAST;
```
MySQL과 PostgreSQL은 nullable 컬럼의 NULL 값을 완전히 다르게 정렬합니다. 오름차순 정렬 시 MySQL에서는 NULL 값이 처음에 나오는 반면 PostgreSQL에서는 맨뒤에서 조회됩니다. NULL 값에 대한 정렬 순서 규칙이 SQL 표준에서 누락되어 DB구현에 따라 기본값이 다를 수 있습니다. 따라서 애플리케션에서는 사용자 경험을 향상시키기 위해 상황에 맞는 제어가 필요합니다. 일반적으로 오름차순, 내림차순으로 지정된 컬럼들은 특정 행을 검색할 때 NULL값을 보는데 관심이 없으므로 마지막에 있어야 합니다. 그러나 컬럼의 빈값이 최신데이터로 의미가 부여된 경우 해당 정보가 가장 먼저 표시되어야 좋은 경우도 있습니다.

> **:bulb:참고**  
> 데이터베이스 전문 웹 사이트인 SQLFordevs.com: [ORDER BY with nullable columns](https://sqlfordevs.com/order-by-with-null) 에서 이 주제와 관련된 자세한 내용을 확인할 수 있습니다.

### 2.6 Deterministic Ordering for Pagination
> 페이징의 결정적 순서

```sql
SELECT *
FROM users
ORDER BY firstname ASC, lastname ASC, user_id ASC
LIMIT {contents 개수} OFFSET {page number}
```
MySQL과 PostgreSQL은 nullable 컬럼의 NULL 값을 완전히 다르게 정렬합니다. 오름차순 정렬 시 MySQL에서는 NULL 값이 처음에 나오는 반면 PostgreSQL에서는 맨뒤에서 조회됩니다. NULL 값에 대한 정렬 순서 규칙이 SQL 표준에서 누락되어 DB구현에 따라 기본값이 다를 수 있습니다. 따라서 애플리케션에서는 사용자 경험을 향상시키기 위해 상황에 맞는 제어가 필요합니다. 일반적으로 오름차순, 내림차순으로 지정된 컬럼들은 특정 행을 검색할 때 NULL값을 보는데 관심이 없으므로 마지막에 있어야 합니다. 그러나 컬럼의 빈값이 최신데이터로 의미가 부여된 경우 해당 정보가 가장 먼저 표시되어야 좋은 경우도 있습니다.

> **:bulb:참고**  
> 데이터베이스 전문 웹 사이트인 SQLFordevs.com: [ORDER BY with nullable columns](https://sqlfordevs.com/order-by-with-null) 에서 이 주제와 관련된 자세한 내용을 확인할 수 있습니다.

### 2.7 More Efficient Pagination Than LIMIT OFFSET
> LIMIT OFFSET 기반 페이지네이션 사용시 주의점
```sql
-- MySQL, PostgreSQL
SELECT *
FROM users
WHERE (firstname, lastname, id) > ('John', 'Doe', 3150)
ORDER BY firstname ASC, lastname ASC, user_id ASC
LIMIT 30
```
일부 행을 건너뛰는 LIMIT OFFSET 키워드를 사용하는 가장 단순한 페이지네이션 접근 방식은 때로는 심각한 문제가 발생합니다. 페이지네이션의 과정 중 데이터가 추가되거나 삭제될 경우, 데이터가 누락되거나 중복될 가능성이 있습니다. 데이터의 중복 유무가 중요하지 않은 관리자 게시판 같은 간단한 경우라면 문제가 없지만, 조회된 데이터를 기반으로 중복없이 추가처리가 필요한 상황이라면 커서기반 페이지네이션을 사용하는 것이 보다 안전합니다. 이 경우에도 정렬할 컬럼에 중복된 값이 존재하면 안되고, 순차적이어야 합니다. 따라서 모든 행이 고유하도록 더 많은 열을 추가하여 정렬할 컬럼에 중복된 값이 존재하면 않도록 하여 항상 순서를 결정적으로 만드는 것이 중요합니다. timstamp 같은 순차적이고 고유한 기본 키를 추가하는 것이 가장 간단한 접근 방식입니다.

### 2.8 Database-Backed Locks With Safety Guarantees
> 안전한 DB기반 잠금 장치
```sql
START TRANSACTION;

SELECT balance FROM account WHERE account_id = 7 FOR UPDATE;

-- 데이터 처리 후 레이스 컨디션 free update
UPDATE account SET balance = 540 WHERE account_id = 7;

COMMIT;
```
거의 모든 애플리케이션은 레이스 컨디션이 발생하기 쉽습니다. 즉, DB에서 값이 선택되고 애플리케이션이 일부 데이터를 처리하며 값이 새 값으로 업데이트 됩니다. 어떤 데이터를 새로운 값으로 업데이트 하기 위해 애플리케이션은 DB에서 값을 조회합니다. 그러나 계산 중에 발생하는 업데이트로부터 애플리케이션은 보호되지 않습니다. 일부 중요한 부분은 레이스 컨디션에 대비한 잠금 솔루션으로 보호됩니다. 그러나 충돌이 발생하는 애플리케이션 논리는 마지막에 락 해제를 놓칠 수 있으므로 잠금 알고리즘을 구축하기가 어렵습니다. 그리고 이 문제를 해결하기 위한 자동 시간 기반 잠금 해제 접근 방식은 여전히 ​​락을 너무 오랫동안 유지하게 됩니다.  
데이터 수정쿼리를(예: UPDATE)를 실행하면 DB는 정확성을 보장하기 위해 트랜잭션이 끝날때 까지 영향을 받는 모든 행을 자체적으로 잠금니다. 애플리케이션에서 잠금 접근 방식을 사용하는 대신 SELECT 쿼리에 FOR UPDATE를 사용하여 레이스 컨디션을 대해 읽기 데이터를 잠그는 방식으로 DB와 협업할 수 있습니다. 이러한 잠금은 트랜잭션이 완료되거나 클리이언트와 연결이 끊어지면 자동으로 해제됩니다.

> **:warning:주의**  
> 값을 업데이트 하는 모든 로직은 락을 사용하지 않으면 레이스 컨디션은 원치 않는 결과로 끝나게 됩니다. 어디에서나 사용해야 하고 아주 작은 부분만으로는 레이스 컨디션 문제를 해결할 수 없습니다.

> **:bulb:참고**  
> 데이터베이스 전문 웹 사이트인 SQLFordevs.com: [Transactional Locking to Prevent Race Conditions](https://sqlfordevs.com/transaction-locking-prevent-race-condition) 에서 이 주제와 관련된 광범위한 글을 작성했습니다.

### 2.9 Refinement Of Data With Common Table Expressions
> CTE를 사용한 데이터 구체화
```sql
WITH most_popular_products AS (
  SELECT products.*, COUNT(*) as sales
  FROM products
  JOIN users_orders_products USING(product_id)
  JOIN users_orders USING(order_id)
  WHERE users_orders.created_at BETWEEN '2022-01-01' AND '2022-06-30'
  GROUP BY products.product_id
  ORDER BY COUNT(*) DESC
  LIMIT 10
), applicable_users (
  SELECT DISTINCT users.*
  FROM users
  JOIN users_raffle USING(user_id)
  WHERE users_raffle.correct_answers > 8
), applicable_users_bought_most_popular_product AS (
  SELECT applicable_users.user_id, most_popular_products.product_id
  FROM applicable_users
  JOIN users_orders USING(order_id)
  JOIN users_orders_products USING(product_id)
  JOIN most_popular_products USING(product_id)
) raffle AS (
  SELECT product_id, user_id, RANK() OVER(
    PARTITION BY product_id
    ORDER BY RANDOM()
  ) AS winner_order
  FROM applicable_users_bought_most_popular_product
)
SELECT product_id, user_id FROM raffle WHERE winner_order = 1;
```
단일 쿼리 하나로 수행하기 어려운 복잡한 규칙이 있는 DB에서 행을 가져와야 하는 경우 CTE(Common Table Expressions)를 사용하여 이를 분할할 수 있습니다. 모든 단계에서 데이터를 구체화하고 나중에 구체화하여 최종적으로 원하는 결과를 얻을 수 있습니다.  
예를 들어 중첩된 서브쿼리가 많거나 조인이 수십 개이면 CTE는 전통적인 접근 방식보다 가독성이 좋고 반복 단계를 고립시켜 별도로 디버깅이 가능합니다. 성능 측면에서 두 가지 접근방식은 모두 동일합니다. DB는 이를 내부적으로 중첩된 서브쿼리로 변환하거나 여러 번 사용되는 단일 단계를 캐싱하여 보다 효율적인 접근 방식을 찾을 수 있습니다.

### 2.10 First Row Of Many Similar Ones
> 많은 유사한 항목 중 첫 번째 행
```sql
-- PostgreSQL
SELECT DISTINCT ON (customer_id) *
FROM orders
WHERE EXTRACT (YEAR FROM created_at) = 2022
ORDER BY customer_id ASC, price DESC;
```
때로는 수많은 행(예를들면 모든 고객)이 있고 그중에 하나만 원하는 경우가 있습니다. 이전에 설명한 대로 for-each-loop와 같은 lateral 조인을 고수하거나 PostgreSQL의 DISTINCT ON invention을 사용할 수 있습니다.
표준 DISTINCT 쿼리는 행의 모든 컬럼에서 정확히 일치하는 행을 필터링합니다. 그러나 위 쿼리처럼 사용하면 컬럼의 서브셋을 지정하여 구별할 수 있으며 정렬 후 첫 번째로 일치하는 행만 유지됩니다.

> **:bulb:참고**  
> 이 기능은 PostgreSQL에서만 사용할 수 있습니다.

### 2.11 Multiple Aggregates In One Query
> 하나의 쿼리로 다중 집계하기
```sql
-- MySQL
SELECT
  SUM(released_at = 2001) AS released_2001,
  SUM(released_at = 2002) AS released_2002,
  SUM(director = 'Steven Spielberg') AS director_stevenspielberg,
  SUM(director = 'James Cameron') AS director_jamescameron
FROM movies
WHERE streamingservice = 'Netflix';
-- PostgreSQL
SELECT
  COUNT(*) FILTER (WHERE released_at = 2001) AS released_2001,
  COUNT(*) FILTER (WHERE released_at = 2002) AS released_2002,
  COUNT(*) FILTER (WHERE director = 'Steven Spielberg') AS
director_stevenspielberg,
  COUNT(*) FILTER (WHERE director = 'James Cameron') AS
director_jamescameron
FROM movies
WHERE streamingservice = 'Netflix';
```
어떤 경우에는 여러 가지 다른 통계를 계산해야 합니다. 수많은 쿼리를 실행하는 대신, 데이터를 한 번에 통과하여 모든 정보를 수집하는 쿼리를 작성할 수 있습니다.
데이터와 인덱스에 따라 실행 시간이 빨라지거나 느려질 수 있으므로 반드시 애플리케이션에서 테스트해야 합니다.

> **:bulb:참고**  
> 데이터베이스 전문 웹 사이트인 SQLFordevs.com: [Multiple Aggregates in One Query](https://sqlfordevs.com/multiple-aggregates-in-one-query) 에서 이 주제와 관련된 광범위한 글을 작성했습니다.

### 2.12 Limit Rows Also Including Ties
> 동일 값을 가진 행도 포함시키는 limit
```sql
-- PostgreSQL
SELECT *
FROM teams
ORDER BY winning_games DESC
FETCH FIRST 3 ROWS WITH TIES;
```
스포츠 리그 팀의 순위를 매기고 상위 3개 팀을 보여주고 싶다고 상상해 보세요. 드문 경우지만, 시즌이 끝날 때 최소 2개 팀의 승률이 동일할 수 있습니다. 둘 다 3위인 경우 둘 다 포함하도록 limit을 확장할 수 있습니다.
만약 양팀 모두 3번째에 위치할 때 양팀을 포함해서 limit 을 확장하기를 원할 수 있습니다. 이 떄 WITH TIES 옵션을 사용하면 됩니다. 포함된 값과 동일한 값을 가진 일부 행이 제외되지 않게 limit을 초과하더라도 해당 행까지 포함시켜 줍니다.

> **:bulb:참고**  
> 이 기능은 PostgreSQL에서만 사용할 수 있습니다.

### 2.13 Fast Row Count Estimates
> 빠른 행 개수 추정
```sql
-- MySQL
EXPLAIN FORMAT=TREE SELECT * FROM movies WHERE rating = 'NC-17' AND price < 4.99;
-- PostgreSQL
EXPLAIN SELECT * FROM movies WHERE rating = 'NC-17' AND price < 4.99;
```
일치하는 행 수를 표시하는 것은 대부분의 애플리케이션에서 중요한 기능이지만 대규모 DB에서는 구현하기 어려운 경우가 있습니다. DB가 클수록 행 수 계산 속도가 느려집니다.
개수를 계산하는 데 도움이 되는 인덱스가 없으면 쿼리 속도가 매우 느려집니다. 그러나 기존 인덱스로도 수십만 개의 인덱스를 빠르게 계산할 수는 없습니다.
하지만 일부 사용 사례에서는 대략적인 행 수만으로도 충분할 수 있습니다. DB 쿼리 플래너는 항상 DB에 실행 계획을 요청하여 추출할 수 있는 쿼리에 대한 대략적인 행 수를 계산합니다.

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


출처 : [SqlForDevs.com](https://sqlfordevs.com/ebook)
