## PIVOT절
- 행 -> 열로 전환 (데이터 요약에 용이함)
- aggregate_function : 집계할 열 지정
- FOR 절은 PIVOT할 열 지정
- IN 절은 PIVOT할 열 값 지정

## UNPIVOT절
- 열 ->  행으로 전환 (이미 집계된 데이터를 활용하여 추가 분석할 때 용이)
- FOR절은 UNPIVOT된 값을 설명할 값이 들어갈 열을 지정
- IN절은 UNPIVOT할 열과 설명할 값의 리터럴 값 지정



## 문제

-> 각 직업별 pivot table을 만들고 만약 없다면 NULL 출력
![alt text](image-16.png)

```sql
WITH SortedOccupations AS (
    SELECT
        Name,
        Occupation,
        ROW_NUMBER() OVER (PARTITION BY Occupation ORDER BY Name) as RowNum
    FROM Occupations
) -- 각 직업별로 이름을 알파벳 순서로 정렬하고, 각 행에 순번을 매기는 CTE

SELECT
    MAX(CASE WHEN Occupation = 'Doctor' THEN Name END) as Doctor,
    MAX(CASE WHEN Occupation = 'Professor' THEN Name END) as Professor,
    MAX(CASE WHEN Occupation = 'Singer' THEN Name END) as Singer,
    MAX(CASE WHEN Occupation = 'Actor' THEN Name END) as Actor
FROM
    SortedOccupations
GROUP BY RowNum
ORDER BY RowNum;
```

- ROW_NUMBER() 함수로 각 직업별로 이름을 정렬하고 고유 행 번호(RowNum)를 생성

- 직업별로 이름을 선택:
    - Occupation = 'Doctor'인 경우 -> 이름 : Doctor 열
    - Occupation = 'Professor'인 경우 -> 이름 : Professor 열

- 출력된 결과를 행 번호 순서로 정렬

## 성능 최적화 기법

`인덱싱(indexing)` : 쿼리를 통해 데이터베이스에게 원하는 정보를 빠르고 정확하게 찾아달라고 요청하고 이 과정에서 중요한 역할을 하는 것 

1. 좌벼을사용하지 않을 것
    - 인덱스 컬럼에 함수 사용 지양, 데이터 원본을 변형하여, 내가 찾고자 하는 범위와 비교하는 연산은 데이터베이스가 인덱스를 제대로 활용할 수 없게 된다.

    -> 원본 데이터를 직접 비교하는 조건을 사용하는 것이 인덱스를 최대한 활용하고 쿼리 성능을 향상시킬 수 있음

2. OR 대신 UNION 사용
    - 인덱스는 단일 값에 대한 빠른 검색을 위해 최적화되어 있는데, OR은 여러 값을 동시에 찾아야 함. 이는 인덱스의 장점을 살리지 못하고, 전체 데이터를 모두 뒤져야 하는 비효율을 초래함
    
    -> UNION or UNION ALL 사용

3. 필요한 열과 행만 선택
    - 불필요한 데이터 처리를 줄임으로써 응답 시간을 단축하고, 시스템 자원을 효율적으로 활용
    - 필요한 열과 조건에 해당하는 행만 조회하도록 쿼리를 설계

4. 분석 함수를 활용
    - ROW_NUMBER(), RANK(), DENSE_RANK() 같은 함수는 데이터 내 각 항목의 순위를 정하는 데 유용
    - LEAD()와 LAG() 함수는 현재 Row와 관련하여 이전 또는 다음 Row의 데이터를 참조


5. 와일드카드(%)는 끝에 작성
    - 데이터베이스가 인덱스를 활용해서 검색 범위를 효과적으로 좁힐 수 있음
    - 가급적 문자열 끝에 와일드카드를 두기

6. 계산값을 미리 저장해두었다가, 나중에 조회
    - 실시간 계산을 위해 서브쿼리까지 사용되고 있다면 쿼리 속도가 느려지는 단점 존재
    - 계산 결과를 별도의 테이블에 저장하여 필요할 때 사용하기
    - 계치, 집계값 등은 미리 계산해서 저장해 두는 것이 성능 최적화에 큰 도움이 됨

### 문제

여러분은 `customer_orders`라는 테이블을 관리하는 데이터베이스 관리자로 일하고 있습니다. 이 테이블에는 고객의 주문 정보가 저장되어 있으며, 각 고객이 주문한 제품과 수량, 가격 정보가 포함되어 있습니다. 또한, 고객들이 특정 제품을 재구매한 비율을 계산하려고 합니다.

#### 테이블 구조:

1. **customers** 테이블
   - `customer_id` (고객 ID, PRIMARY KEY)
   - `name` (고객 이름)
2. **orders** 테이블
   - `order_id` (주문 ID, PRIMARY KEY)
   - `customer_id` (고객 ID, FOREIGN KEY)
   - `order_date` (주문 날짜)
3. **order_details** 테이블
   - `order_id` (주문 ID, FOREIGN KEY)
   - `product_id` (제품 ID)
   - `quantity` (수량)
   - `unit_price` (단가)

---

#### 요구 사항:

1. **`avg_order_value`**: 고객별 평균 주문 금액을 계산하여 `customers` 테이블에 업데이트하세요.
   - `avg_order_value`는 각 고객이 한 번의 주문에서 지출한 평균 금액입니다.
2. **`total_spent`**: 고객별 총 지출 금액을 계산하여 `customers` 테이블에 업데이트하세요.
   - `total_spent`는 고객이 지금까지 지출한 총 금액입니다.
3. **`num_orders`**: 고객이 총 몇 번의 주문을 했는지 계산하여 `customers` 테이블에 업데이트하세요.
   - `num_orders`는 고객이 주문한 총 개수입니다.
4. **`repurchase_rate`**: 고객의 재구매 비율을 계산하여 `customers` 테이블에 업데이트하세요.
   - `repurchase_rate`는 각 고객이 2번 이상 주문한 제품 비율을 의미합니다. (즉, 재구매한 제품이 전체 구매 제품 중 몇 퍼센트를 차지하는지)

#### 예시:

- 고객 A는 3번 주문을 했고, 그 중 2개의 제품을 재구매했습니다.
  - 평균 주문 금액: 100,000원
  - 총 지출 금액: 300,000원
  - 주문 횟수: 3번
  - 재구매 비율: 66.67%

### 정답

```sql

-- 1. avg_order_value (평균 구매 금액)
UPDATE customers c
SET
    avg_order_value = (
        SELECT AVG(od.quantity * od.unit_price)
        FROM order_details od
        JOIN orders o ON od.order_id = o.order_id
        WHERE o.customer_id = c.customer_id
    ),

    -- 2. total_spent (총 지출 금액)
    total_spent = (
        SELECT SUM(od.quantity * od.unit_price)
        FROM order_details od
        JOIN orders o ON od.order_id = o.order_id
        WHERE o.customer_id = c.customer_id
    ),

    -- 3. num_orders (총 주문 수)
    num_orders = (
        SELECT COUNT(DISTINCT o.order_id)
        FROM orders o
        WHERE o.customer_id = c.customer_id
    ),

    -- 4. repurchase_rate (재구매율)
    repurchase_rate = (
        SELECT
            COUNT(DISTINCT CASE WHEN od.product_id IN (
                SELECT product_id
                FROM order_details
                JOIN orders o ON order_details.order_id = o.order_id
                WHERE o.customer_id = c.customer_id
                GROUP BY order_details.product_id
                HAVING COUNT(order_details.product_id) > 1
            ) THEN od.product_id END) * 1.0 / COUNT(DISTINCT od.product_id)
        FROM order_details od
        JOIN orders o ON od.order_id = o.order_id
        WHERE o.customer_id = c.customer_id
    );
```

### 추가질문

1. 정답 쿼리에서 `repurchase_rate`를 구할 때 사용한 `HAVING COUNT(order_details.product_id) > 1`의 의미는 무엇인가요?

- 같은 product_id가 2번 이상 등장한 경우(=재구매된 경우)를 필터링 하기 위해 사용

2. 이 문제에서 사용될 수 있는 성능을 최적화하기 위한 방법은 무엇일까요?

- 서브쿼리를 반복적으로 실행하는 대신, 계산 결과를 미리 임시 테이블에 저장한 후 사용
- 쿼리 리팩토링(with절 사용 / 중복되는 join 제거)을 통해 중복되는 JOIN과 서브쿼리를 제거하여 반복 작업을 줄이기




