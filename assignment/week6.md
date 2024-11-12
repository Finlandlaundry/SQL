## 틀린 코드 이유 분석(정답 코드 참고)

**틀린 코드**

```sql
SELECT *
FROM SELECT FOOD_TYPE, REST_ID, REST_NAME, MAX(FAVORITES) AS FAVORITES
FROM REST_INFO
GROUP BY FOOD_TYPE
ORDER BY FOOD_TYPE DESC
```

**틀린 이유**
```
- 오답 코드에서는  GROUP BY FOOD_TYPE을 사용했기 때문에 FOOD_TYPE 외의 REST_ID, REST_NAME은 적용되지 않는다.
- 음식종류별로 즐겨찾기수가 가장 많은 식당을 구현하기 위해 MAX(FAVORITES)를 사용했는데 오답 코드에서는 각 FOOD_TYPE별로
즐겨찾기 수의 최대값만 반환한다. REST_ID와 REST_NAME는 적용되지 않기 때문에 서브쿼리 순서를 바꿔줘야 한다.

```

**정답 코드**
```sql
    SELECT FOOD_TYPE, REST_ID, REST_NAME, FAVORITES
    FROM REST_INFO
    WHERE (FOOD_TYPE, FAVORITES) IN (
        SELECT FOOD_TYPE, MAX(FAVORITES)    
        FROM REST_INFO
        GROUP BY FOOD_TYPE
    ) 
    ORDER BY FOOD_TYPE DESC;
```


## 개선된 쿼리 학습

또한, 이 문제에서는 아래 **개선된 쿼리**로도 조회될 수 있습니다. 

**ROW_NUMBER 윈도우 함수**를 사용합니다.
### 성분으로 구분한 아이스크림 총 주문량

```sql
WITH RankedRest AS (                        -- WITH 절 사용
    SELECT 
        FOOD_TYPE,              
        REST_ID,               
        REST_NAME,              
        FAVORITES,              
        ROW_NUMBER() OVER (                  -- ROW_NUMBER 윈도우 함수 사용
            PARTITION BY FOOD_TYPE           -- 음식 종류별로 그룹화
            ORDER BY FAVORITES DESC, REST_ID -- 즐겨찾기 수 내림차순, 동점일 경우 REST_ID 오름차순 정렬
        ) AS rnk                     
    FROM REST_INFO
)
SELECT 
    FOOD_TYPE,                  
    REST_ID,                    
    REST_NAME,                  
    FAVORITES                   
FROM RankedRest
WHERE rnk = 1                   
ORDER BY FOOD_TYPE DESC; 
```

- 성능 이점) 기존 서브쿼리 방식은 MAX(FAVORITES)를 계산하기 위해 추가로 데이터를 그룹화 및 비교해야 하므로, 데이터가 클 경우 성능 차이가 날 수 있습니다.
- 기존 코드에서는 서브쿼리로 MAX(FAVORITES)를 구하고, 이를 메인 쿼리에서 비교해야 했는데, 개선된 코드는 윈도우 함수를 사용해 한 번의 연산으로 순위를 계산하며, 서브쿼리를 사용하지 않아 코드를 단순화할 수 있다.
