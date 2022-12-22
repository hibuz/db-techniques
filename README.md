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


출처 : [SqlForDevs.com](https://sqlfordevs.com/ebook)

