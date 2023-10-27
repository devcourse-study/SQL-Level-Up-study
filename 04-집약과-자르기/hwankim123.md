# [SQL 레벨업] 4장 - 집약과 자르기

# 1. 집약

SQL에는 집약 함수라고 하는, 여러 개의 레코드를 한 개의 레코드로 집약하는 함수가 있습니다.

- COUNT
- SUM
- AVG
- MAX
- MIN

  <br>

## 1-1. 여러 개의 레코드를 한 개의 레코드로 집약

아래와 같은 테이블(NonAggTbl)이 있다고 가정하겠습니다.

| id | data_type | data_1 | data_2 | data_3 | data_4 | data_5 | data_6 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Jim | A | 100 | 10 | 34 | 346 | 54 |  |
| Jim | B | 45 | 2 | 167 | 77 | 90 | 157 |
| Jim | C |  | 3 | 687 | 1355 | 324 | 457 |
| Ken | A | 78 | 5 | 724 | 457 |  | 1 |
| Ken | B | 123 | 12 | 178 | 346 | 85 | 235 |
| Ken | C | 45 |  | 23 | 46 | 687  | 33 |
| Beth | A | 75 | 0 | 190 | 25 | 356 |  |
| Beth | B | 435 | 0 | 183 |  | 4 | 325 |
| Beth | C | 96 | 128 |  | 0 | 0 | 12 |
  
이 테이블의 특징은 data_type마다 해당 레코드에서 사용하고자 하는 데이터의 분류가 다르다는 것입니다. data_type 필드가 A라면 data_1 ~ data_2, B라면 data_3 ~ data_5, C라면 data_6를 사용합니다.

이런 비집약 테이블처럼 한 사람과 관련된 정보가 여러개의 레코드로 분산되어있는 테이블은, 단순 SELECT 구문을 사용하면 당연히 여러 레코드가 선택됩니다. 또한 A~C에서 사용하는 데이터가 필요하다면 3개의 쿼리를 작성해야 할 것입니다.

이런 데이터는 아래와 같이 레이아웃의 테이블(AggTbl)로 만드는 것이 바람직합니다. 즉, 한 사람을 한 개의 레코드로 집약한 테이블입니다.

| id | data_1 | data_2 | data_3 | data_4 | data_5 | data_6 |
| --- | --- | --- | --- | --- | --- | --- |
| Jim | 100 | 10 | 167 | 77 | 90 | 457 |
| Ken | 78 | 5 | 178 | 346 | 85 | 33 |
| Beth | 75 | 0 | 183 |  | 4 | 12 |

이렇게 테이블이 집약되어있으면 한 사람의 정보가 모두 같은 레코드에 들어있습니다. 따라서 한 사람의 정보를 얻을 때 쿼리 하나면 충분합니다. 모델링이라는 관점에서 보아도, 사람이라는 엔티티를 나타내는 테이블은 이렇게 되어있어야 합니다.

<br>

### CASE 식과 GROUP BY 응용

이제 NonAggTbl에서 AggTbl로 변환하려면 어떤 SQL을 사용해야할지 생각해봅시다. 사람 id를 기준으로 GROUP BY을 하고, 선택할 필드를 data_type 필드로 분기합니다. 여기에서 CASE 식을 사용합니다.

이런 내용을 사용하면 일단 아래와 같은 형태의 쿼리를 사용할 수 있습니다.

```sql
// 오류가 발생하는 쿼리
SELECT id, 
       CASE WHEN data_type = 'A' then data_1 ELSE NULL END AS data_1,
       CASE WHEN data_type = 'A' then data_2 ELSE NULL END AS data_2,
       CASE WHEN data_type = 'B' then data_3 ELSE NULL END AS data_3,
       CASE WHEN data_type = 'B' then data_4 ELSE NULL END AS data_4,
       CASE WHEN data_type = 'B' then data_5 ELSE NULL END AS data_5,
       CASE WHEN data_type = 'C' then data_6 ELSE NULL END AS data_6
FROM NonAggTbl
GROUP BY id;

// 발생한 오류
[42000][1055] Expression #2 of SELECT list is not in GROUP BY clause 
and contains nonaggregated column 'sql_levelup.NonAggTbl.data_type' 
which is not functionally dependent on columns in GROUP BY clause; 
this is incompatible with sql_mode=only_full_group_by
```

오류가 발생하는 이유는 GROUP BY로 집약했을 때 SELECT 구에 입력할 수 있는 것은 다음과 같은 세 가지뿐입니다.

- 상수
- GROUP BY 구에서 사용한 집약 키
- 집약 함수

 

하지만 id 필드로 그룹화하고 CASE 식에 data_type을 지정하면 하나의 레코드만 선택됩니다. 따라서 집약 함수를 사용하지 않고 data_1~data_6을 그냥 입력해도 데이터베이스 엔진이 눈치가 있다면 새로운 레코드를 만들어낼 수 있을 것입니다. 하지만 이러한 발상은 SQL의 원리(집합론의 원리)를 위배한 것입니다. 따라서 귀찮더라도 집약 함수를 사용해서 아래와 같이 작성해야 합니다.

```sql
SELECT id, 
          MAX(CASE WHEN data_type = 'A' then data_1 ELSE NULL END) AS data_1,
          MAX(CASE WHEN data_type = 'A' then data_2 ELSE NULL END) AS data_2,
          MAX(CASE WHEN data_type = 'B' then data_3 ELSE NULL END) AS data_3,
          MAX(CASE WHEN data_type = 'B' then data_4 ELSE NULL END) AS data_4,
          MAX(CASE WHEN data_type = 'B' then data_5 ELSE NULL END) AS data_5,
          MAX(CASE WHEN data_type = 'C' then data_6 ELSE NULL END) AS data_6
FROM NonAggTbl
GROUP BY id;
```

GROUP BY로 데이터를 자르는 시점에는 각 집합에 3개의 요소가 있습니다.

- Jim 데이터 3개
- Ken 데이터 3개
- Beth 데이터 3개

그런데 여기에 집약 함수가 적용되면 NULL을 제외하고 하나의 요소만 있는 집합이 만들어집니다. MAX 함수를 사용하면 내부에 있는 하나의 요소를 선택할 수 있습니다. MIN,AVG, SUM 과 같은 함수를 사용해도 현재 예제에서는 상관없지만, 집약 기준이 문자 또는 날짜 등의 데이터일 때에도 사용할 수 있도록 MAX, MIN을 사용하는 습관을 들이는 것이 좋습니다.

<br>

### 집약, 해시, 정렬

그럼 이런 집약 쿼리의 실행 계획은 어떻게 될까요?

> 저는 MySQL을 주로 사용하기 때문에 MySQL의 실행계획을 살펴보았으나, 책의 실행계획과는 좀 다른 형태로 나오고 있습니다.
> 

<img width="846" alt="스크린샷 2023-10-24 오후 3 34 12" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/91baac25-5e44-4a1b-8e21-c4c8395c0502">

Oracle로 살펴본 실행계획은 다음과 같습니다.

(사진)

NonAggTbl 테이블을 모두 스캔하고 GROUP BY로 집약을 수행하는 단순한 실행 계획입니다. 중요한것은 GROUP BY의 집약 조작에 ‘해시’ 알고리즘을 사용했다는 것입니다. 경우에 따라서는 정렬을 사용하기도 하는데, 최근에는 집약에서 정렬보다 해시를 사용하는 경우가 많습니다. 이는 GROUP BY 구에 저장되어 있는 필드를 해시 함수를 사용해 해시 키로 변환하고, 같은 해시 키를 가진 그룹을 모아 집약하는 방법입니다. 정렬을 사용한 방법보다 빠르므로 많이 사용되고 있습니다. 특히 해시 성질상 GROUP BY의 유일성이 높으면 더 효율적으로 작동합니다.

GROUP BY와 관련되 성능 주의점을 짚어봅시다. 정렬과 해시 모두 메모리를 많이 사용하므로, 충분한 해시용(또는 정렬용) 워킹 메모리가 확보되지 않으면 스왑이 발생해 굉장히 느려집니다. 

> MySQL이 정렬과 해시를 위해 사용하는 메모리는 무엇인지, 작동 원리는 무엇인지  
> [https://velog.io/@rnjsrntkd95/MySQL-Group-By-성능-개선-with-Distinct](https://velog.io/@rnjsrntkd95/MySQL-Group-By-%EC%84%B1%EB%8A%A5-%EA%B0%9C%EC%84%A0-with-Distinct)


따라서 연산 대상 레코드 수가 많은 GROUP BY 구(또는 집약 함수)를 사용하는 SQL에서는 충분한 성능 검증(실제 환경에서 어떻게 되는지 부하 검증)을 실행해줘야 합니다.

<br>

## 1-2. 합쳐서 하나

집약의 이해를 위해 간단한 문제를 풀어봅시다. 제품의 대상 연령별 가격을 관리하는 테이블(PriceByAge)이 있습니다. 같은 제품이라도 나이에 따라 가격이 다를 수 있고, 한 개의 제품에서 연령 범위가 중복되는 경우는 없습니다.

| product_id(제품 ID) | low_age(대상 연령의 하한) | high_age(대상 연령의 상한) | price(가격) |
| --- | --- | --- | --- |
| 제품1 | 0 | 50 | 2000 |
| 제품1 | 51 | 100 | 3000 |
| 제품2 | 0 | 100 | 4200 |
| 제품3 | 0 | 20 | 500 |
| 제품3 | 31 | 70 | 800 |
| 제품3 | 71 | 100 | 1000 |
| 제품4 | 0 | 99 | 8900 |

이제 이런 제품 중에서 0~100세까지 모든 연령이 가지고 놀 수 있는 제품을 구하는 문제를 풀여야 합니다. 예를 들어 제품 1의 경우 2개의 레코드를 합치면 0~100세까지 모든 연령이 가지고 놀 수 있어 조건에 만족합니다. 이렇게 1개의 레코드로 전체를 커버하지 못해도 여러 개의 코드를 조합해 커버할 수 있다면 ‘합쳐서 하나’라고 하는 것이 문제의 주제입니다.

```sql
SELECT product_id
FROM PriceByAge
GROUP BY product_id
HAVING SUM(high_age - low_age + 1) = 101;
```

지금 예제에서는 ‘연령’이라는 숫자 자료형의 데이터를 사용했습니다. 하지만 확장하면 날짜 또는 시각에도 적용할 수 있습니다. 호텔방마다 도착일과 출발일을 기록하는 테이블(HotelRooms)이 있습니다.

| room_nbr(방 번호) | start_date(도착일) | end_date(출발일) |
| --- | --- | --- |
| 101 | 2008-02-01 | 2008-02-06 |
| 101 | 2008-02-06 | 2008-02-08 |
| 101 | 2008-02-10 | 2008-02-13 |
| 202 | 2008-02-05 | 2008-02-08 |
| 202 | 2008-02-08 | 2008-02-11 |
| 202 | 2008-02-11 | 2008-02-12 |
| 303 | 2008-02-03 | 2008-02-17 |

이 테이블에서 사람들이 숙박한 날이 10일 이상인 방을 선택합니다. 숙박한 날의 수는 도착일이 2월 1일, 출발일이 2월 6일이라면 5박이므로 5일입니다.

```sql
SELECT room_nbr
FROM HotelRooms
GROUP BY room_nbr
HAVING SUM(end_date - start_date) >= 10;
```

<br>

# 2. 자르기

GROUP BY는 ‘집약’ 이외에도 ‘자르기’라는 기능이 있습니다. 이는 원래 모 집합인 테이블을 작은 부분 집합들로 분리하는 것입니다.

<br>

## 2-1. 자르기와 파티션

예제로 아래와 같이 개인 신체정보를 저장하고 있는 테이블이 있다고 생각합니다.

```sql
CREATE TABLE Persons
(name VARCHAR(8) NOT NULL,
age INTEGER NOT NULL,
height FLOAT NOT NULL,
weight FLOAT NOT NULL,
PRIMARY KEY (name));
```

| name(이름) | age(나이) | jeight(키(cm)) | weight(몸무게(kg)) |
| --- | --- | --- | --- |
| Anderson | 30 | 188 | 90 |
| Adela | 21 | 167 | 55 |
| Bates | 87 | 158 | 48 |
| Becky | 54 | 187 | 70 |
| Bill | 39 | 177 | 120 |
| Chris | 90 | 175 | 48 |
| Darwin | 12 | 160 | 55 |
| Dawson | 25 | 182 | 90 |
| Donald | 30 | 176 | 53 |

이 테이블로부터 이름의 앞 알파벳을 기준으로 몇 명이 존재하는지 집계해봅시다.

```sql
SELECT SUBSTRING(name, 1, 1) AS label,
				COUNT(*)
FROM Persons
GROUP BY SUBSTRING(name, 1, 1);

//실행 결과
label | COUNT(*)
-----------------
A     | 2
B     | 3
C     | 1
D     | 3
```

<br>

### 파티션

파티션은 서로 중복되는 요소를 가지지 않는 부분 집합입니다. 같은 모집합이라도 파티션을 만드는 방법은 굉장히 많습니다. 예를 들어 나이 기준으로 어린이(20세 미만), 성인(20~69세), 노인(70세 이상) 으로 나눈다면 쿼리는 다음과 같이 됩니다.

```sql
SELECT CASE WHEN age < 20 THEN '어린이'
				WHEN age BETWEEN 20 AND 69 THEN '성인'
				WHEN age >= 70 THEN '노인'
				ELSE NULL END AS age_class,
				COUNT(*)
FROM Persons
GROUP BY CASE WHEN age < 20 THEN '어린이'
				WHEN age BETWEEN 20 AND 69 THEN '성인'
				WHEN age >= 70 THEN '노인'
				ELSE NULL END;

//실행 결과
age_class | COUNT(*)
--------------------
어린이      | 1
성인        | 6
노인        | 2
```

자르기의 기준이 되는 키를 GROUP BY 구와 SELECT 구 모두에 입력하는 것이 포인트입니다. PostgreSQL과 MySQL는 뒤에 붙은 별칭을 사용하여 ‘GROU BY age_class’와 같이 작성할 수 있지만, 표준은 아닙니다.

그럼 GROUP BY 구에서 CASE 식을 사용하면 실행 계획은 어떻게 될까요? MySQ 기준으로 살펴봅시다.

<img width="823" alt="스크린샷 2023-10-27 오후 6 34 21" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/da949fbc-82b1-40cf-9822-ed96e79bc794">

GROUP BY 구에서 CASE 식 또는 함수를 사용해도 실행계획에는 영향이 없다는 것을 알 수 있습니다. 필드에 연산을 추가한 식을 키로 한다면 CPU 오버헤드가 걸릴 것이지만, 이또한 데이터 접근과는 무관합니다. 사실 GROUP BY의 실행 계획은 성능적인 측면에서, 해시(또는 정렬)에 사용되는 워킹 메모리의 용량에 주의하라는 것 외에는 따로 할 말이 없습니다.

 <br>

### BMI로 자르기

> BMI = w / t^2 (w: 몸무게(kg), t: 키(m))
> 

이 BMI 수치를 기준으로 18.5 미만을 저체중, 18.5 이상 25 미만을 정상, 25 이상을 과체중으로 합니다. 이러한 기준을 바탕으로 Persons 테이블의 사람들의 체중을 분류하고 몇 명이 해당되는지 알아봅시다.

```sql
SELECT CASE WHEN weight / POWER(height / 100, 2) < 18.5 THEN '저체중'
						WHEN 18.5 <= weight / POWER(height / 100, 2)
								AND weight / POWER(height / 100, 2) < 25 THEN '정상'
						WHEN 25 <= weight / POWER(height / 100, 2) THEN '비만' 
						ELSE NULL END AS bmi,
FROM persons
GROUP BY CASE WHEN weight / POWER(height / 100, 2) < 18.5 THEN '저체중'
						WHEN 18.5 <= weight / POWER(height / 100, 2)
								AND weight / POWER(height / 100, 2) < 25 THEN '정상'
						WHEN 25 <= weight / POWER(height / 100, 2) THEN '비만' 
						ELSE NULL END;
```

이렇게 GROUP BY에는 복잡한 수식을 기준으로도 자를 수 있습니다.

이 쿼리에 대한 MySQL 기준 실행 계획입니다.
<img width="860" alt="스크린샷 2023-10-27 오후 7 31 40" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/bca62bc5-e1d1-4a39-96f2-eeb6d2491172">

## 2-2. PARTITION BY 구를 사용한 자르기

PARTITION BY 구는 GROUP BY의 집약 기능만 없는 것을 제외하면 실질적인 기능에는 차이가 없습니다.

한마디로 PARTITION BY에도 필드 이름 뿐만 아니라 CASE 식, 계산 식을 사용한 복잡한 기준을 사용할 수 있다는 말입니다.
```sql
SELECT name, age,
				CASE WHEN age < 20 THEN '어린이'
				WHEN age BETWEEN 20 AND 69 THEN '성인'
				WHEN age >= 70 THEN '노인'
				ELSE NULL END AS age_class,
				RANK() OVER(PARTITION BY CASE WHEN age < 20 THEN '어린이'
																			WHEN age BETWEEN 20 AND 69 THEN '성인'
																			WHEN age >= 70 THEN '노인'
																			ELSE NULL END
										ORDER BY age) AS age_rank_in_clas
FROM Persons
ORDER BY age_class,age_rank_in_classk;

//실행 결과
name      | age | age_class | age_rank_in_class
------------------------------------------
Darwin    | 12  | 어린이      | 1
Adela     | 21  | 성인       | 1
Dawson    | 25  | 성인       | 2
Anderson  | 30  | 성인       | 3
Donald    | 30  | 성인       | 4
Bill      | 39  | 성인       | 5
Becky     | 54  | 성인       | 6 
Bates     | 87  | 노인       | 1 
Chris     | 90  | 노인       | 2
```

#### 블로그 링크
https://habefast.tistory.com/23
