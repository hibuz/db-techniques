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
> 데이터베이스 전문 웹 사이트인 SQLFordevs.com 에서 UPDATE from a SELECT와 관련된 자세한 내용을 확인할 수 있습니다.

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
### 1.5 Table Maintenance After Bulk Modifications

## 2. Querying Data
### 2.1 Reduce The Amount Of Group By Columns
### 2.2 Fill Tables With Large Amounts Of Test Data
### 2.3 Simplified Inequality Checks With Nullable Columns
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

