[6강 SELECT 구문](#6강-SELECT-구문)

[7강 조건 분기, 집합 연산, 윈도우 함수, 갱신](#7강-조건-분기,-집합-연산,-윈도우-함수,-갱신)

[궁금한점](궁금한점)


# 개요
---
이번 장에서는 SQL과 관련된 기초적인 설명을 합니다.

# 6강 SELECT 구문
---
SELECT 구문은 검색을 위해 사용합니다.

예시) 모든 사람 조회
```SQL
SELECT * FROM People;
```

## SELECT 구와 FROM 구

`SELECT` 구는 검색할 테이블의 `필드`들을 지정할 수 있습니다.
`FROM` 구는 검색할 대상이 되는 `테이블`을 지정할 수 있습니다.

## 특정 조건을 검색할 때는 어떻게 할까?

항상 모든 데이터가 필요한 것은 아닙니다. 특정 조건의 데이터를 검색할 때는 `WHERE` 구를 이용할 수 있습니다.

예시) 주소가 인천시인 사람 검색
```SQL
SELECT * FROM People WHERE address = '인천시';
```

WHERE 구는 '=' 이외에도 다양한 연산자를 통해 조건 지정을 할 수 있습니다. (`< >`, `>=` 등등)

또한, `AND`, `OR`을 통해 추가적인 조건을 붙일 수 있습니다.

예시) 주소가 인천시고 나이가 30 이상인 사람 검색
```SQL
SELECT * 
FROM People
WHERE address = '인천시' AND age >= 30;
```

`IN`을 통해 `OR` 조건을 간단하게 작성할 수도 있습니다.

`OR` 이용
```SQL
SELECT * 
FROM People
WHERE address = '인천시'
	OR address = '서울시'
	OR address = '부산시';
```

`IN` 이용
```SQL
SELECT * 
FROM People
WHERE address IN ('인천시', '서울시', '부산시');
```

### NULL 체크

NULL 을 체크할 때는 `IS NULL` 이라는 키워드를 사용해야 합니다.
NULL이 아닌 레코드를 체크할 때는 `IS NOT NULL`을 이용합니다.

```SQL
SELECT * 
FROM People
WHERE phone_number IS NULL;
```

## 집계 연산은 어떻게 사용할까?

`GROUP BY`를 이용하면 합계 또는 평균 등의 작업을 수행할 수 있습니다.

예시) 성별에 따른 사람수 조회
```SQL
SELECT sex, COUNT(*)
FROM People
GROUP BY sex;
```

### 그룹을 조회할 때 어떻게 조건을 걸까?

답은 `HAVING` 구를 사용하는 것입니다.

예시) 지역 별로 사람 수 조회
```SQL
SELECT address, COUNT(*)
FROM People
GROUP BY address;
```

위 예시에서 사람수가 1명인 지역만 조회할 때 다음과 같이 작성할 수 있습니다.

```SQL
SELECT address, COUNT(*)
FROM People
GROUP BY address
HAVING COUNT(*) = 1;
```

###  WHERE vs HAVING

두 구문 모두 `조건`을 설정하는 점에서 비슷합니다. 다만, `WHERE`의 경우 레코드에 조건을 지정하는 것이고 `HAVING`은 `집합`에 조건을 거는 것입니다.

## 조회한 결과는 어떻게 정렬할까?

원하는 순서를 지정하기 위해서는 `ORDER BY`를 이용할 수 있습니다.
기본 설정으로 `ASC`(오름차순)이고, `DESC`를 명시하면 내림차순으로 설정할 수 있습니다.

예시) 나이 오름차순으로 조회
```SQL
SELECT *
FROM People
ORDER BY age;
```

### 여러 개의 기준으로 정렬할 때는 어떻게 할까?

이 때는 우선 순위가 높은 순서대로 쉼표(,)를 통해 작성하면 됩니다.

예시) 나이 내림차순으로, 나이가 같으면 전화번호 오름차순으로 정렬
```SQL
SELECT *
FROM People
ORDER BY age DESC, phone_number;
```

## 매번 같은 SELECT 구문을 실행하지 않는 방법은 무엇일까?

SQL에는 `VIEW`라는 기능이 존재합니다. 
해당 기능을 통해 `SELECT` 구문을 데이터베이스에 저장할 수 있습니다.

> 데이터는 보관 X, 구문만 저장

뷰 생성 예시는 다음과 같습니다.

```SQL
CREATE VIEW CountAddress (v_address, cnt)
AS
SELECT address, COUNT(*)
FROM People
GROUP BY address;
```

이렇게 생성한 뷰는 다음과 같이 사용할 수 있습니다.

```SQL
SELECT v_address, cnt
FROM CountAddress;
```

### 익명 뷰란 무엇일까?

뷰는 테이블처럼 데이터를 저장하는 것이 아닌 구문만 저장합니다.
따라서, 뷰를 조회할 때 내부적으로는 `SELECT` 구문이 추가적으로 호출되게 됩니다.
아래의 조회 구문은 서로 동일한 결과를 도출합니다.

```SQL
SELECT v_address, cnt
FROM CountAddress;
```

```SQL
SELECT v_address, cnt
FROM (SELECT address, COUNT(*)
	FROM People
	GROUP BY address) AS CountAddress;
```

위와 같이 뷰 대신 `FROM` 구에 직접 `SELECT` 구문을 지정할 수 있습니다.
이런 SELECT 구문을 `서브쿼리`(subquery)라고 부릅니다.

### 서브쿼리는 언제 이용할까?

서브쿼리를 `WHERE` 구에 조건으로 활용할 수 있습니다. 

예를 들어 필드는 같고, 데이터만 다른 테이블 People1, People2 가 있다고 가정하겠습니다. (다만, 공통된 데이터도 존재합니다.)
이 상황에서 People1과 People2의 공통된 데이터를 어떻게 검색할 수 있을까요?

서브쿼리를 통해 다음과 같이 작성할 수 있습니다.

```SQL
SELECT *
FROM People1
WHERE name IN (SELECT name FROM People2);
```

SQL은 서브쿼리가 먼저 실행합니다.
이에 따라 People2에 있는 사람 이름의 리스트 중 People1에 해당하는 이름이 있다면, 결과 테이블에 출력됩니다.

이 방식의 장점은 `동적`으로 결과를 조회할 수 있다는 점입니다.
People2의 데이터가 바뀌더라도 `SELECT` 구문이 다시 실행되어 변경된 데이터가 반영되어 결과가 계산됩니다.

# 7강 조건 분기, 집합 연산, 윈도우 함수, 갱신
---

## 조건 분기

SQL에서는 조건 분기를 `CASE` 식으로 제공하고 있습니다.
문법은 다음과 같습니다.

```SQL
CASE WHEN [평가식] THEN [식]
	WHEN [평가식] THEN [식]
	WHEN [평가식] THEN [식]
		생략
	ELSE [식]
END
```

식에는 `필드=값` 과 같은 조건이 들어갑니다.

CASE 식은 if-else나 switch 문 과 비슷한 역할을 수행합니다. 
하지만 값을 `리턴`값이 존재하는 점에서 차이점이 있습니다.

다음의 예시로 살펴보겠습니다.

```
SELECT name, address,
	CASE WHEN address = '서울시' THEN '경기'
	CASE WHEN address = '인천시' THEN '경기'
	CASE WHEN address = '부산시' THEN '영남'
	CASE WHEN address = '속초시' THEN '관동'
	ELSE NULL END AS distinct
FROM People
```

위의 예시는 주소를 더 큰 분류로 조회하는 쿼리입니다.
결과로 다음과 같이 나오게 됩니다.

|name|address|distinct|
|----|----|----|
|인성|서울시|경기|
|하진|부산시|영남|
|준|인천시|경기|
|민|속초시|관동|

`CASE` 식은 `SELECT`, `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY` 구와 같은 곳 어디에나 사용할 수 있습니다.

### SQL 집합 연산

SQL에는 집합 연산이 존재합니다. 
바로 `UNION`과 `INTERSECT`, `EXCEPT`인데요.
각각 합집합, 교집합, 차집합에 대응합니다.

#### UNION

각 SELECT의 연산 결과를 하나로 합치는 연산을 수행합니다. 
이 때 중복되는 값이 있다면 하나만 결과에 반영됩니다.
중복을 제외하고 싶지 않다면 `UNION ALL`을 사용하면 됩니다.

```SQL
SELECT *
FROM People
UNION
SELECT *
FROM People
```

#### INTERSECT

각 결과를 교집합한 것을 조회합니다.
마찬가지로 중복된 것은 하나만 반영되므로 `ALL` 키워드를 통해 제외시키지 않을 수 있습니다. (`EXCEPT`도 동일)

```SQL
SELECT *
FROM People1
UNION
SELECT *
FROM People2
```

#### EXCEPT

각 결과에서 차집합을 수행하는 연산입니다.
`UNION`과 `INTERSECT`와는 달리 순서가 결과에 영향을 줍니다.

```SQL
SELECT *
FROM People1
EXCEPT
SELECT *
FROM People2
```

### 윈도우 함수

윈도우 함수는 '집약 기능이 없는 `GROUP BY` 구'라고 할 수 있습니다.
즉, 각 데이터를 분류하는 작업만 수행합니다.
이러한 특성 때문에 출력 결과가 레코드의 수와 동일하게 나타납니다.

기본 문접은 집약 함수 또는 윈도우 전용 함수 뒤에 `OVER` 구를 작성하고, 키를 지정하기 위한 `PARTITION BY` 또는 `ORDER BY`를 입력하는 것입니다. (둘 다 입력 가능)

예시) 주소 별 사람 수
```SQL
SELECT address, COUNT(*) OVER PARTITION BY address
FROM People
```

위의 SQL문을 실행하면 결과의 레코드 개수가 테이블의 레코드 개수와 동일하게 나옵니다.

### 트랜잭션과 갱신

#### 테이블에 데이터 삽입

테이블에 데이터를 삽입하기 위해서 `INSERT`구문을 사용할 수 있습니다.

예시)
```SQL
INSERT INTO People(name, phone_number, address, sex, age) 
	VALUES ('인성', '010-1111-2222', '서울시', '남', 30);
```

여러 개의 레코드를 한번에 삽입할 수는 없을까요?
한번에 삽입할 수 있는 SQL 구문을 지원하는 DBMS가 존재합니다. 다음과 같이 작성할 수 있습니다. (모든 DBMS가 지원하는 것은 아닙니다.)

```SQL
INSERT INTO People(name, phone_number, address, sex, age) 
	VALUES ('인성', '010-1111-2222', '서울시', '남', 30), VALUES ('하진', '010-3333-4444', '인천시', '', 21), VALUES ('준', '010-5555-6666', '부산시', '남', 45), VALUES ('민', '010-7777-8888', '속초시', '남', 32);
```

#### 데이터 제거

데이터를 삭제할 떄는 `DELETE` 구문을 이용합니다.
해당 구문을 통해 여러 개의 데이터를 한번에 삭제할 수 있습니다.

또한, `WHERE`절을 통해 원하는 조건의 데이터만 삭제할 수도 있습니다.

예시) 주소가 인천시인 데이터 삭제
```SQL
DELETE FROM People
WHERE address = '인천시';
```

`DELETE` 구문은 테이블 자체를 삭제할 수는 없습니다.
또한, 테이블의 필드를 삭제하는 것도 불가능합니다.

#### 데이터 갱신

등록된 데이터를 변경할 때는 `UPDATE` 구문을 이용합니다.

`DELETE`와 마찬가지로 `WHERE` 구문을 통해 조건에 맞는 데이터만 수정할 수 있습니다.

예시) 인성의 전화번호 갱신
```SQL
UPDATE People
SET phone_number = '010-1010-2020'
WHERE name = '인성';
```

한번에 여러 개의 필드도 수정할 수 있습니다.
방법으로는 다음과 같은 두가지 방법이 존재합니다.

```SQL
# 방법 1 (모든 DBMS에서 지원)
UPDATE People
SET phone_number = '010-1010-2020', age = 20
WHERE name = '인성';

# 방법 2 (일부 DBMS에서 지원)
UPDATE People
SET (phone_number, age) = ('010-1010-2020', 20)
WHERE name = '인성';
```

# 궁금한점
---
## 윈도우 함수에 대해 더 공부해보자
> 행과 행간의 관계를 쉽게 정의하기 위해 만든 함수


### WINDOW 함수 종류 ([출처](!http://www.gurubee.net/lecture/2382))

![![Resource/SQL Level Up/#^Table1]]
### WINDOW 구문 사용 방법
1. OVER 문구 필수 포함
2. 다음과 같이 사용
	```SQL
	SELCT WINDOW_FUNCTION (ARGUMENTS) 
	   OVER ( [PARTITION BY 칼럼] [ORDER BY 절] )
	FROM 테이블 명;
	```

- ARGUMENT: WINDOW 함수가 적용되는 칼럼
- PARTITION BY: 전체 데이터를 여러 그룹으로 나눌 때의 기준(칼럼)
- ORDER BY: 각 그룹의 정렬할 기준(칼럼)

> 실제 한번 사용해보자

원본 테이블

| name | phone_number | address | sex | age |
| ---- | ---- | ---- | ---- | ---- |
| 인성| 010-1111-1111 | 서울시 | 남| 30|
| 하진| 010-2222-2222 | 서울시 | 여| 21|
| 준| 010-3333-3333 | 서울시 | 남| 45|
| 민| 010-4444-4444 | 부산시 | 남| 32|
| 하린| 010-5555-5555 | 부산시 | 여| 55|
| 빛나래| 010-6666-6666 | 인천시 | 여| 19|
| 인아| 010-7777-7777 | 인천시 | 여| 20|
| 아린| 010-8888-8888 | 속초시 | 여| 25|
| 기주| 010-9999-9999 | 서귀포시 | 남| 32|


### 예시1) PARTITION BY만 사용

```SQL
SELECT name,  
       age,  
       address,  
       COUNT(*) OVER (PARTITION BY address) as count  
FROM People;
```

결과

| name | age | address | count |
| ---- | ---- | ---- | ---- |
| 민 | 32 | 부산시 | 2 |  
| 하린 | 55 | 부산시 | 2 |  
| 기주 | 32 | 서귀포시 | 1 |  
| 인성 | 30 | 서울시 | 3 |  
| 하진 | 21 | 서울시 | 3 |  
| 준 | 45 | 서울시 | 3 |  
| 아린 | 25 | 속초시 | 1 |  
| 빛나래 | 19 | 인천시 | 2 |  
| 인아 | 20 | 인천시 | 2 |

결과 해석

PARTITION BY를 통해 지역에 대해 전체 데이터를 분류했고, 각 그룹에서의 숫자를 검색한 것입니다.
ORDER BY를 명시하지 않았기 때문에 각 그룹 내에서 정렬되어 출력되지는 않았습니다.

### 예시2) ORDER BY만 사용

```SQL
SELECT name,  
        age,  
        RANK() over (ORDER BY age DESC) AS `rank`  
FROM People;
```

결과

| name | age | rank |
| ---- | ---- | ---- |
|하린 | 55 | 1|
|준 | 45 | 2|
|민 | 32 | 3|
|기주 | 32 | 3|
|인성 | 30 | 5|
|아린 | 25 | 6|
|하진 | 21 | 7|
|인아 | 20 | 8|
|빛나래 | 19 | 9|


결과 해석

PARTITION BY가 사용되지 않았으므로 모든 데이터에 대해 WINDOW 함수를 적용한 것으로 보입니다.
ORDER BY가 사용되었으로 각 분류에 대해서 age로 내림차순 정렬되야 하지만, 분류가 없으므로 전체 데이터에 대해 age로 내림차순 된 것을 볼 수 있습니다.

### 예시3) PARTITION BY, ORDER BY 둘 다 사용

```SQL
SELECT name,  
       age,  
       address,  
       COUNT(*) OVER (PARTITION BY address ORDER BY age DESC) as count  
FROM People;
```

결과

| name | age | address | count |
| ---- | ---- | ---- | ---- |
| 하린 | 55 | 부산시 | 1 |  
| 민 | 32 | 부산시 | 2 |  
| 기주 | 32 | 서귀포시 | 1 |  
| 준 | 45 | 서울시 | 1 |  
| 인성 | 30 | 서울시 | 2 |  
| 하진 | 21 | 서울시 | 3 |  
| 아린 | 25 | 속초시 | 1 |  
| 인아 | 20 | 인천시 | 1 |  
| 빛나래 | 19 | 인천시 | 2 |

결과 해석

예제 1에서 `ORDER BY age DESC`를 적용하여 각 그룹에서 나이 기준으로 정렬된 것을 볼 수 있습니다.

## 윈도우 함수에서 OVER 뒤에 오는 ORDER BY와 일반 ORDER BY를 같이 적용하면 어떻게 될까?

위의 예제 3 쿼리에서 다음과 같이 쿼리를 변경해보았습니다.

```SQL
SELECT name,  
       age,  
       address,  
       COUNT(*) OVER (PARTITION BY address ORDER BY age DESC) as count  
FROM People  
ORDER BY name;
```

결과

| name | age | address | count |
| ---- | ---- | ---- | ---- |
|기주 | 32 | 서귀포시 | 1 |  
|민 | 32 | 부산시 | 2 |  
|빛나래 | 19 | 인천시 | 2 |  
|아린 | 25 | 속초시 | 1 |  
|인성 | 30 | 서울시 | 2 |  
|인아 | 20 | 인천시 | 1 |  
|준 | 45 | 서울시 | 1 |  
|하린 | 55 | 부산시 | 1 |  
|하진 | 21 | 서울시 | 3 |

결과 해석

`OVER` 뒤에 오는 ORDER BY는 무시되었고 단순히 `name` 칼럼에 대해서만 정렬된 것을 볼 수 있습니다.

MySQL(의 InnoDB)에서 실험하여 다른 DBMS에서는 다른 결과가 나올 수 있겠지만, `ORDER BY를 중복`해서 사용할 때는 주의해야 할 것 같습니다.

## UNION은 필드가 다른 테이블에 대해서도 사용할 수 있을까?

UNION의 기본 조건은 다음과 같습니다.
1. 모든 SELECT 문의 column 수가 같아야 한다.
2. 각 column의 데이터 타입이 같아야 합니다.
3. column은 동일한 순서여야 합니다.

그렇다면 필드의 이름이 다른 테이블에 대해서 UNION을 했을 때 형식만 맞추면 어떻게 될까요?

테이블은 다음이 age만 빼고 이름은 모두 다르며, 타입은 모두 동일합니다.

table 1

| surname | given_name | age |
| ---- | ---- | ---- |
| varchar(255) |  varchar(255) | int |

| surname | given_name | age |
| ---- | ---- | ---- |
| Kelby| Weymouth | 24 |  
| Prentice| Traite | 7 |  
| Sauncho| Cuerda | 87 |  
| Glenda| Collacombe | 13 |

table 2

| lastname | firstname | age |
| ---- | ---- | ---- |
| varchar(255) |  varchar(255) | int |

| lastname | firstname | age |
| ---- | ---- | ---- |
| Marlyn | Houseman | 1 |  
| Pearline | Pasek | 2 |  
| Olwen | Portwain | 3 |  
| Tamqrah | Astbury | 4 |

다음 SQL문을 실행하여 테스트를 진행했습니다.

```SQL
SELECT given_name, surname, age  
FROM table1  
UNION  
SELECT firstname, lastname, age  
FROM table2;
```

| surname | given_name | age |
| ---- | ---- | ---- |
| Kelby| Weymouth | 24 |  
| Prentice| Traite | 7 |  
| Sauncho| Cuerda | 87 |  
| Glenda| Collacombe | 13 |
| Marlyn | Houseman | 1 |  
| Pearline | Pasek | 2 |  
| Olwen | Portwain | 3 |  
| Tamqrah | Astbury | 4 |

결과는 위와 같이 처음 제시된 칼럼 명으로 출력 테이블이 생성된 것을 볼 수 있습니다.

제가 궁금했던 `칼럼 명이 달라도` UNION이 잘 동작할까에 대한 의문은 `동작`하는 것으로 해결되었습니다. 

## "`CASE` 식은 `SELECT`, `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY` 구와 같은 곳 어디에나 사용할 수 있습니다. "-> 요거 실제 예시가 궁금하다.

[WHERE](!https://www.mssqltips.com/sqlservertip/7703/sql-case-in-where-clause/)

[GROUP BY](!https://learnsql.com/blog/group-by-case-when/)

[ORDER BY](!https://learnsql.com/blog/case-in-sql-order-by/)

[HAVING](!https://learnsql.com/blog/case-sql/)

# 피드백
---
윈도우 함수를 제대로 이해하지 못했다. (PARTITION BY의 역할)

윈도우 함수의 효용성에 대해 조사하면 좋을 것 같다.

프로그래머스 SQL 문제 5문제 정도 풀고 공유하기
