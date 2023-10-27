# 집약
---
표준 SQL에는 다음과 같은 집약 함수가 존재합니다.

```SQL
COUNT
SUM
AVG
MAX
MIN
```

해당 함수들에 '집약'이라는 접두사가 붙은 이유는 여러 개의 레코드를 하나의 레코드로 **집약**하는 기능을 가지기 때문입니다.






## [여러 개의 레코드를 한 개의 레코드로 집약]

실제 예시를 통해 집약을 하면 어떻게 되는지 살펴보겠습니다.


다음과 같은 테이블이 존재합니다.

**NonAggTbl**

| id | data_type | data_1 | data_2 | data_3 | data_4 | data_5 | data_6 |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| Jim | A | 100 | 10 | 34 | 346 | 54 |  |
| Jim | B | 45 | 2 | 167 | 77 | 90 | 157 |
| Jim | C |  | 3 | 687 | 1355 | 324 | 457 |
| Ken | A | 78 | 5 | 724 | 457 |  | 1 |
| Ken | B | 123 | 12 | 178 | 346 | 85 | 235 |
| Ken | C | 45 |  | 23 | 46 | 687 | 33 |
| Beth | A | 75 | 0 | 190 | 25 | 356 |  |
| Beth | B | 435 | 0 | 183 |  | 4 | 325 |
| Beth | C | 96 | 128 |  | 0 | 0 | 12 |


다음과 같이 data_type에 따라 해당 레코드에서 사용하는 칼럼이 다릅니다.

```
A -> data_1, data_2 
B -> data_3, data_4, data_5 
C -> data_6
```


위와 같은 비집약 테이블에서는 한 사람의 정보를 접근할 때는 `Where id = 'Jim'` 과 같은 SELECT 구문이 사용되고, 3개의 레코드가 검색됩니다.


>한 사람에 대한 데이터는 한 개의 레코드로 얻으려면 어떻게 할까요?

**AggTbl**

| id | data_1 | data_2 | data_3 | data_4 | data_5 | data_6 |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| Jim | 100 | 10 | 167 | 77 | 90 | 457 |
| Ken | 78 | 5 | 178 | 346 | 85 | 33 |
| Beth | 75 | 0 | 183 |  | 4 | 12 |


1. **UNION** 이용

첫번째 방법은 id와 data_type으로 필요한 칼럼을 조회한 후 이를 UNION으로 합치는 것입니다.


하지만 이와 같은 방식은 전 장에서 살펴봤다 싶이 성능적으로 비효율적입니다.


2. CASE 및 GROUP BY 이용

다른 방법으로는, id로 GROUP BY를 한 후 data_type을 CASE 식을 통한 방법입니다.


```SQL
SELECT id,
		MAX(CASE WHEN data_type = 'A' THEN data_1 
			ELSE NULL END) AS data_1,
		MAX(CASE WHEN data_type = 'A' THEN data_2 
			ELSE NULL END) AS data_2,
		MAX(CASE WHEN data_type = 'B' THEN data_3 
			ELSE NULL END) AS data_3,
		MAX(CASE WHEN data_type = 'B' THEN data_4 
			ELSE NULL END) AS data_4,
		MAX(CASE WHEN data_type = 'B' THEN data_5 
			ELSE NULL END) AS data_5,
		MAX(CASE WHEN data_type = 'C' THEN data_6 
			ELSE NULL END) AS data_6
FROM NonAggTbl
GROUP BY id;
```


>cf) MAX를 붙인 이유는 GROUP BY를 적용했을 때 SELECT에 들어갈 수 있는 것은 다음 3가지 입니다.
>
>1.상수
>2. GROUP BY 구에서 사용한 집약 키
>3. 집약 함수
>
>data_n은 위의 3가지에 해당되지 않아 집약 함수를 이용하여 나타낸 것입니다.


해당 집약 쿼리의 **실행 계획**은 어떻게 될까요?

![[스크린샷 2023-10-27 오후 5.28.33.png]]


**Temporary Table**을 통해 집약을 하는 것을 볼 수 있습니다.
(책과는 상이, 책에서는 해시를 통해 집약하는 걸로 나옵니다.)


책을 기준으로 설명하면, GROUP BY에서 집약을 할 때는 해시 또는 정렬을 사용합니다.


해시의 경우, GROUP BY 구에 지정된 필드를 해시 함수의 키로 변환하고 같은 해시 키를 가진 그룹을 모아 집약하는 방식입니다.


정렬을 사용한 방법보다 더 빨라 많이 사용되고 있습니다.


해시의 특성상 GROUP BY 키의 유일성이 높으면 더 효율적으로 동작합니다.


GROUP BY와 관련된 성능 주의점은 **메모리**와 관련이 있습니다.


GROUP BY 연산을 위해 워킹 메모리가 확보되지 않으면 DISK로의 스왑이 발생해 굉장히 느려집니다.


이때문에 GROUP BY를 사용한 SQL 문에서는 충분한 성능 검증(실제 환경에서의 부하 검증)을 실행해야 합니다.







## [합쳐서 하나]

집약을 이해하기 위해 간단한 문제를 하나 풀어보겠습니다.
(SQL Puzzles 2판, Morgan Kaufmann의 퍼즐 65 제품 대상 연럼 범위)


다음과 같이 제품의 대상 연령별 가격을 관리하는 테이블이 있습니다.
(product_id와 low_age에 의해 식별됩니다.)

**PriceByAge**

|product_id(예약 ID)|low_age(대상 연령의 하한)|high_age(대상 연령의 상한)|price(가격)|
| ---- | ---- | ---- | ---- |
|제품1|0|50|2000|
|제품1|51|100|3000|
|제품2|0|100|4200|
|제품3|0|20|500|
|제품3|31|70|800|
|제품3|71|100|1000|
|제품4|0|99|8900|


다음과 같은 요구 사항에 대해 쿼리를 어떻게 작성할 수 있을까요?

>0~100세까지 모든 연령이 가지고 놀 수 있는 제품을 구하라


제품 1의 경우에는 2개의 레코드로 0~100까지의 정수 범위를 커버할 수 있습니다.


반면, 제품 3의 경우에는 중간에 끊긴 부분이 존재합니다.


이와 같이 하나의 레코드로는 커버하지 못해도 여러 개의 레코드의 조합으로 커버할 수 있다면 '**합쳐서 하나**'라고 하는 것이 문제의 주제입니다.


코드는 다음과 같습니다.

```SQL
SELECT product_id  
FROM PriceByAge  
GROUP BY product_id  
HAVING SUM(high_age - low_age + 1) = 101;
```






# 자르기
---

지금까지는 GROUP BY의 '집약'이라는 측면을 강조해서 살펴봤습니다.


그렇다면 다른 중요한 기능인 이제 '자르기'에 대해 알아보겠습니다.


## 1. 자르기와 파티션

다음과 같이 개인 신체정보를 저장하고 있는 테이블이 있다고 하겠습니다.

|name|age|height|weight|
| ---- | ---- | ---- | ---- |
|Anderson|30|188|90|
|Adela|21|167|55|
|Bates|87|158|48|
|Becky|54|187|70|
|Bill|39|177|120|
|Chris|90|175|48|
|Darwin|12|160|55|
|Dawson|25|182|90|
|Donald|30|176|53|

다음과 같은 요구사항이 있습니다.

>이름 첫 글자를 사용해 특정 알파벳으로 시작하는 이름을 가진 사람이 몇 명인지 집계하자


다음과 같이 작성할 수 있습니다.

```SQL
SELECT SUBSTRING(name, 1, 1) AS lable,  
       COUNT(*)  
FROM Persons  
GROUP BY SUBSTRING(name, 1, 1);
```

|label|COUNT(\*)|
| --- | --- |
|A|2|
|B|3|
|C|1|
|D|3|

위와 같이 4개의 부분 집합으로 나뉘게 됩니다.


이와 같이 GROUP BY 구로 잘라 만든 하나하나의 부분 집합을 수학적으로 '**파티션**'이라고 부릅니다.


파티션은 서로 중복되는 요소를 가지지 않는 집합입니다.


위의 테이블을 나이 기준으로 나눈다면 다른 종류의 파티션을 생성할 수 있습니다.

```SQL
SELECT CASE WHEN age < 20 THEN '어린이'  
		    WHEN age BETWEEN 20 AND 69 THEN '성인'  
		    WHEN age > 69 THEN '노인'  
    ELSE NULL END AS age_class  
FROM Persons  
GROUP BY CASE WHEN age < 20 THEN '어린이'  
		      WHEN age BETWEEN 20 AND 69 THEN '성인'
		      WHEN age > 69 THEN '노인'
    ELSE NULL END;
```

위와 같이 SELECT 구와 GROUP BY 구에 자르기에 기준이 되는 키가 모두 입력되는 것을 볼 수 있습니다.


위의 쿼리의 실행 계획은 어떻게 될까요?

![[스크린샷 2023-10-27 오후 6.07.46.png]]


앞에서 본 것처럼 임시 테이블을 생성해서 GROUP BY를 처리하고 있습니다.


다만, GROUP BY 구에서 CASE 식 또는 함수를 사용해도 실행 계획에는 **영향이 없는 것**을 볼 수 있습니다.


CASE 식 또는 함수는 데이터를 뽑아온 뒤에 행해지기 때문에 데이터 접근 경로에는 영향을 주지 않습니다.


따라서 GROUP BY의 실행 계획은 성능적인 측면에서, 임시 테이블의 용량에 주의하는 것 이외에 따로 할 말은 없습니다.


위의 테이블에서 **BMI**를 기준으로 구분하려면 어떻게 할까요?
(BMI = 몸무게 / (키)^2)


다음과 같이 작성할 수 있습니다.
```SQL
SELECT CASE WHEN weight / POWER(height/100, 2) < 18.5 THEN '저체중'  
    WHEN 18.5 <= weight / POWER(height/100, 2) AND weight / POWER(height/100, 2) < 25 THEN '정상'  
    WHEN weight / POWER(height/100, 2) >= 25 THEN '과체중'  
    ELSE NULL END AS bmi,  
    COUNT(*)  
FROM Persons  
GROUP BY CASE WHEN weight / POWER(height/100, 2) < 18.5 THEN '저체중'  
    WHEN 18.5 <= weight / POWER(height/100, 2) AND weight / POWER(height/100, 2) < 25 THEN '정상'  
    WHEN weight / POWER(height/100, 2) >= 25 THEN '과체중'  
    ELSE NULL END;
```


위와 같이 단순 필드 이름 이외에 복잡한 수식을 기준으로 자를 수 있습니다.






## PARTITION BY 구를 사용한 자르기

PARTITION BY는 GROUP BY에서 집약 기능을 제외하고 자르기 기능만 남긴 것이라고 2장에서 소개했습니다.


PARTITION BY 구를 사용해도 CASE 식, 계산 식을 사용한 복잡한 기준을 사용할 수 있습니다.


위에서 살펴봤던 연령 범위 테이블에서 파티션 자르기를 해보겠습니다.


연령 등급(어린이, 성인, 노인)에서 어린 순서로 순위를 매기는 코드를 작성하면 다음과 같습니다.

```SQL
SELECT name,  
       age,  
       CASE WHEN age < 20 THEN '어린이'  
            WHEN age BETWEEN 20 AND 69 THEN '성인'  
            WHEN age > 69 THEN '노인'  
            ELSE NULL END AS age_class,  
       RANK() OVER (PARTITION BY 
				       CASE WHEN age < 20 THEN '어린이'  
							WHEN age BETWEEN 20 AND 69 THEN '성인'  
                            WHEN age > 69 THEN '노인'  
                            ELSE NULL END  
		           ORDER BY age) AS age_rank_in_class  
FROM Persons  
ORDER BY age_class, age_rank_in_class;
```






# 궁금한 점
---

## MySQL에서는 GROUP BY에 어떤 연산을 사용할까?

MySQL에서는 GROUP BY를 3가지 방식으로 처리할 수 있습니다.

1. Temporary Table
2. Tight Index Scan
3. Loose Index Scan


**[Temporary Table]**

위에서 저의 쿼리에서 나온 방식으로, 다음과 같은 방법으로 처리됩니다.

1. 테이블을 Full Table Scan 방식으로 읽는다.
2. 조회한 결과를 임시 테이블에 저장한다. 이 때 사용되는 임시 테이블은 GROUP BY 절에 사용된 컬럼과 SELECT 하는 칼럼만 저장한다.
	- 임시 테이블에서는 GROUP BY에 사용된 칼럼으로 **유니크 키**를 생성한다.
3. 임시 테이블의 유니크 키 순서대로 읽어 클라이언트로 전송된다.
	- 만약 쿼리의 ORDER BY 절에 명시된 칼럼과 GROUP BY 절에 명시된 칼럼이 다르면 Filesort를 통해 다시 한번 정렬 작업을 수행한다.


[Tight Index Scan]

임시 테이블을 이용하지 않고 **인덱스 스캔**을 이용하여 GROUP BY를 처리하는 방식입니다.


WHERE 조건에 명시된 범위 조건이 있다면 해당 조건을 만족하는 키 값만 읽고, 그렇지 않으면 인덱스 스캔을 수행하는 방식입니다.


즉, WHERE 절에 있는 모든 범위의 키를 읽거나 전체 인덱스를 스캔하기 때문에 Tight Index Scan으로 불리는 것입니다.


GROUP BY가 해당 방식으로 처리되더라도, 그룹 함수를 처리하기 위한 임시 테이블을 생성할 수 있습니다.


해당 방식을 이용할 때 실행 계획에서는 Extra 칼럼에 별로도 GROUP BY 관련 코멘트나 정렬 관련 코멘트가 표시되지 않습니다.


[Loose Index Scan]

가장 효율적으로 처리하는 방법으로, 인덱스의 레코드를 건너뛰면서 필요한 부분만 가져오는 방법입니다.


WHERE 절이 없는 경우 그룹의 수만큼 레코드를 읽어 전체를 읽는 것에 비해 효율적입니다.


만약 WHERE 절에 범위 조건이 있는 경우 범위 조건을 충족하는 각 그룹의 첫번째 레코드를 조회하고 충족하는지 판별합니다.


해당 방식을 사용했을 때 실행 계획에서는 `Using index for group-by`라고 Extra 칼럼에 표시됩니다.


마지막으로 다음 쿼리는 해당 방식을 이용할 수 없는 쿼리 패턴입니다.

1. MIN()과 MAX() 이외의 집합 함수 사용
	```SQL
	SELECT col1, SUM(col2)
	FROM test
	GROUP BY col1;
	```
2. GROUP BY에 사용된 칼럼이 인덱스 구성 칼럼의 왼쪽부터 일치하지 않는 경우
	```SQL
	SELECT col1, col2
	FROM test
	GROUP BY col2, col3;
	```
3. SELECT 절의 칼럼이 GROUP BY와 일치하는 않는 경우
	```SQL
	SELECT col1, col3
	FROM test
	GROUP BY col1, col2;
	```
