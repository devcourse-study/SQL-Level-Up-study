# 1. SELECT 구문

## 1-1. SELECT 구와 FROM 구

SELECT 구문은 검색을 위해 사용합니다.  
  

### FROM 구

FROM 구는 SELECT 구문을 실행하는 대상을 지정하기 위해 사용됩니다. FROM 구는 필수가 아닙니다.(SELECT 1과 같이 상수를 선택하는 경우)

## 1-2. WHERE 구

특정 조건의 데이터를 검색할 때 WHERE 구를 사용할 수 있습니다.

| 연산자 | 의미 |
| --- | --- |
| = | ~와 같음 |
| <> | ~와 같지 않음 |
| ≥ | ~ 이상 |
| > | ~ 보다 큼 |
| ≤ | ~ 이하 |
| < | ~ 보다 작음 |

- AND, OR
- IN절: 여러개의 OR 절을 간단히 작성
- NULL: NULL인 레코드를 선택하려면 특별한 키워드를 사용해야 함
    - IS NULL
    - IS NOT NULL

> SELECT 구문의 기능은 절차 지향형 언어의 함수와 같은 역할을 합니다. SELECT 구문의 경우에도 자료형이 정해져있는데, 그것은 바로 ‘테이블’(관계) 입니다. 따라서 입력도 출력도 모두 2차원 표입니다. 뷰와 서브쿼리를 함께 이해할 때 이 사실이 중요합니다.
> 

## 1-3. GROUP BY 구

GROUP BY 구를 사용하면, 테이블에서 단순하게 데이터를 선택하는 것뿐만 아니라 합계 또는 평균 등의 집계 연산을 SQL 구문으로 할 수 있습니다.

GROUP BY는 테이블의 데이터들을 ‘필드’를 기준으로 그룹화하는 구문입니다. 데이터를 그룹별로 나눴으면 그룹별로 집계가 가능합니다.

| 함수 이름 | 설명 |
| --- | --- |
| COUNT | 레코드 수를 계산 |
| SUM | 숫자를 더함 |
| AVG | 숫자의 평균을 구함 |
| MAX | 최댓값을 구함 |
| MIN | 최솟값을 구함 |

GROUP BY의 기준을 생략하여 작성할 수도 있습니다. 이때 테이블 전체를 하나의 큰 그룹으로 보게 됩니다. 보통은 `GROUP BY()` 구를 생략합니다.

```sql
SELECT COUNT(*)
FROM Address
GROUP BY ();

SELECT COUNT(*)
FROM ADDRESS;
```

## 1-4. HAVING 구

HAVING 구는 GROUP BY로 인해 얻어진 결과 집합에서 또다시 조건을 걸어 선택하는 기능입니다. HAVING 구는 SQL이 가진 굉장히 강력한 기능 중 하나로, 다양하게 응용할 수 있습니다.

```sql
// 그룹의 개수(COUNT)가 1인 레코드의 address, COUNT 조회
SELECT address, COUNT(*)
FROM Address
GROUP BY address
HAVING COUNT(*) = 1;
```

## 1-5. ORDER BY 구

SELECT 구문으로 출력되는 결과는 딱히 정해진 규칙 없이, 무작위 순서로 출력됩니다. DBMS에 따라 특정한 규칙이 있을 순 있지만, SQL의 일반적인 규칙에서는 정렬과 관련된 내용이 없습니다.  

SELECT 구문의 결과 순서를 보장하려면 명시적으로 순서를 지정해줘야 하는데, 이때 ORDER BY 구를 사용합니다.

```sql
SELECT name, phone_nbr, address, sex, age
FROM Address
ORDER BY age DESC, phone_nbr;
```

- ASC(default): 오름차순
- DECS: 내림차순

## 1-6. 뷰와 서브쿼리

뷰는 데이터베이스에 ‘SELECT 구문’을 저장하는 기능입니다. 뷰를 생성하면 데이터베이스에 ‘SELECT 구문’을 저장해놓고, 데이터를 조회하듯이, 구문을 조회하여 다른 SELECT 쿼리에 활용할 수 있습니다.

```sql
CREATE VIEW [뷰 이름] ([필드 이름1], [필드 이름2] ... ) AS [SELECT 구문]
```

```sql
CREATE VIEW CountAddress (v_address, cnt)
AS
SELECT address, COUNT(*)
FROM Address
GROUP BY address;
```

만들어진 뷰는 SELECT 구문에서 테이블처럼 사용할 수 있습니다. 뷰에서 데이터를 선택하는 SELECT 구문은 실제로는 내부적으로 ‘추가적인 SELECT 구문’을 실행하는 중첩 구조가 되는 것입니다. 두 번째 구문과 같이 FROM 구에 직접 지정하는 SELECT 구문을 서브쿼리라고 부릅니다.

```sql
// 뷰에서 데이터를 선택
SELECT v_address, cnt
FROM CountAddress;

// 뷰는 실행할 때 SELECT 구문으로 전개
SELECT v_address, cnt
FROM (SELECT address AS v_address, COUNT(*) AS cnt
			FROM Address
			GROUP BY address) AS CountAddress;
```

서브쿼리는 WHERE 절의 조건에 활용될 수 있습니다. IN절은 상수를 매개변수로 받는 것 뿐만 아니라, 서브쿼리를 매개변수로 받을 수 있습니다.

```jsx
// Address 테이블에서 Address2 테이블에 있는 사람을 선택
SELECT name
FROM Address
WHERE name IN (SELECT name FROM Address2);
```

SQL은 서브쿼리부터 순서대로 실행합니다. 따라서 앞의 SELECT 구문을 받은 DBMS는 서브쿼리를 상수로 전개해서 아래와 같이 바꿉니다.

```jsx
SELECT name
FROM Address
WHERE name IN ('인성','민', '준서', '지연', '서준', '중진');
```

서브쿼리를 조건절에 활용하면 데이터가 변경되어도 쿼리를 따로 수정할 필요가 없다는 점에서 굉장히 편리합니다.

# 2. 조건 분기, 집합 연산, 윈도우 함수, 갱신

## 2-1. SQL과 조건 분기

SQL은 조건 분기를 ‘식’을 기준으로 합니다. 그리고 이런 식의 분기를 실현하는 기능이 바로 CASE 식입니다.

```jsx
//검색 CASE 식의 구문. 검색 CASE 식만 기억해도 충분
CASE WHEN [평가식] THEN [식]
		 WHEN [평가식] THEN [식]
		 WHEN [평가식] THEN [식]
		 ...
		ELSE [식]
END
```

여기에서 평가식이란 ‘필드 = 값’ 처럼 조건을 지정하는 식을 말합니다. WHERE 구에 조건을 작성하는 방법과 같습니다.

CASE 식은 절차지향 언어의 switch 문과 거의 비슷합니다. CASE 식의 첫번째 평가식부터 평가되고 조건이 맞으면 THEN 구에서 지정한 식이 리턴되며 CASE 식 전체가 종료됩니다.

```jsx
// 구하고자 하는 실행 결과
name  | address | district
--------------------------
인성  | 서울시   | 경기
하진  | 서울시   | 경기
준    | 서울시   | 경기
민    | 부산시   | 영남
하린  | 부산시   | 영남
빛나래| 인천시   | 경기
인아  | 인천시   | 경기
아린  | 속초시   | 관동
가주  | 서귀포시 | 호남

//시도의 이름을 클 지역으로 구분하는 CASE 식
SELECT name, address,
			CASE WHEN address = '서울시' THEN '경기'
			CASE WHEN address = '인천시' THEN '경기'
			CASE WHEN address = '부산시' THEN '영남'
			CASE WHEN address = '속초시' THEN '관동'
			CASE WHEN address = '서귀포시' THEN '호남'
			ELSE NULL END AS district
FROM Address;
```

일반적으로 CASE 식은 위와 같이 ‘서울시 → 경기’, ‘부산시 → 영남’ 과 같이 값의 교환 규칙을 정의하는데 많이 사용합니다.

CASE의 강력한 점은 **식이라는 것**입니다. 따라서 식을 적을 수 있는 곳이라면 어디든 적을 수 있습니다. SELECT, WHERE, GROUP BY, HAVING, ORDER BY 구와 같은 곳 어디에나 적을 수 있으므로 다양한 기법으로 활용할 수 있습니다. CASE 식은 SQL의 성능과도 굉장히 큰 관련이 있습니다.

## 2-2. SQL의 집합 연산

집합 연산은 서로 다른 두 테이블의 공통 필드에 대해 합집합, 교집합, 차집합 등의 연산을 할 수 있는 기능입니다.

UNION 연산자는 테이블에 대한 합집합을 구하는 연산자입니다. 만약 대상 테이블 양쪽에 중복되는 레코드가 있다면 UNION은 중복을 제거하여 결과 레코드를 반환합니다. 이는 UNION 뿐만 아니라 INTERSECT, EXCEPT 등에서도 같습니다. 만약 중복을 제외하고 싶지 않다면 ‘UNION ALL’ 처럼 ALL 옵션을 붙이면 됩니다.

```jsx
SELECT * FROM Address
UNION
SELECT * FROM Address2;
```

교집합을 구하는 연산자는 INTERSECT 입니다.

```jsx
SELECT * FROM Address
INTERSECT
SELECT * FROM Address2;
```

차집합을 구하는 연산자는 EXCEPT 입니다. 아래 구문의 결과는 ‘Address - Address2’입니다.

```jsx
SELECT * FROM Address
EXCEPT
SELECT * FROM Address2;
```

## 2-3. 윈도우 함수

윈도우 함수는 데이터를 가공하게 해준다는 점에서도 중요하지만, 성능과 큰 관계가 있어 굉장히 중요한 기능입니다.

윈도우 함수의 특징은 ‘집약 기능이 없는 GROUP BY구’입니다. GROUP BY는 자르기와 집약 기능이 있는 구이지만, 윈도우 함수는 자르기 기능만 있는 구입니다.

아래 구문은 addresss 를 기준으로 자르고, COUNT를 통해 레코드 수를 집약하는 구문입니다.

```jsx
SELECT address, COUNT(*)
FROM Address
GROUP BY address;

// 실행 결과
address   | count
-----------------
서울시    | 3
인천시    | 2
부산시    | 2
속초시    | 1
서귀포시  | 1
```

윈도우 함수는 ‘PARTITION BY’라는 구로 수행합니다. 위의 GROUP BY와 차이가 있다면 자른 후에 집약하지 않으므로 출력 결과의 레코드 수가 입력으로 주어지는 테이블의 레코드 수와 같다는 것입니다.

윈도우 함수의 기본적인 구문은 집약 함수 뒤에 OVER 구를 작성하고, 내부에 자를 키를 지정하는 PARTITION BY 또는 ORDER BY를 입력하는 것입니다. 작성하는 위치는 SELECT 구라고만 생각해도 문제없습니다.

 

```jsx
SELECT address, COUNT(*) OVER(PARTITION BY address)
FROM Address;

// 실행 결과
address   | count
-----------------
속초시    | 1
인천시    | 2
인천시    | 2
서울시    | 3
서울시    | 3
서울시    | 3
부산시    | 2
부산시    | 2
서귀포시  | 1
```

윈도우 함수에서 사용할 수 있는 함수로는 기본적인 COUNT, SUM 외에도 윈도우 함수 전용으로 제공되는 RANK 또는 ROW_NUMBER 등의 순서 함수가 있습니다.

RANK 함수는 지정된 키로 레코드에 순위를 붙이는 함수입니다. RANK 함수는 숫자가 같으면 같은 순위를 표시하고 건너뛰는데, 건너뛰고싶지 않을 때는 DENSE_RANK 함수를 사용합니다.

```jsx
SELECT name, age, RANK() OVER(ORDER BY age DESC) AS rnk
FROM Address;

// 실행 결과
address   | count | rnk
-----------------------
속초시    | 1     | 1
인천시    | 2     | 2
인천시    | 2     | 3
서울시    | 3     | 4
서울시    | 3     | 5
서울시    | 3     | 6
부산시    | 2     | 7
부산시    | 2     | 8
서귀포시  | 1     | 9
```

## 윈도우 함수 추가 정리(Real MySQL)

윈도우 함수와 GROUP BY는 모두 하나의 레코드 결과를 연산하는데 다른 레코드들을 참초(집계를 위해서)합니다. 윈도우 함수가 GROUP BY와 다른 특성을 갖는 것은 윈도우 함수는 **결과 집합이 전혀 바뀌지 않으면서** 하나의 레코드가 다른 레코드들을 참조할 수 있다는 점입니다.  

### 쿼리 각 절의 실행 순서

쿼리의 각 절의 실행 순서를 정확히 숙지해야 정확한 쿼리를 작성할 수 있습니다. 아래 두 쿼리는 LIMIT 절의 위치가 달라 서로 다른 결과를 만듭니다.
![쿼리 실행 순서](https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/64d864c5-2f26-40dd-8654-db198106961f)

```sql
//윈도우 함수보다 LIMIT 절이 더 나중에 실행되기 때문에, 
//AVG의 결과값은 5개의 레코드에 대한 평균값이 아니라 WHERE 절을 만족하는 모든 레코드에 대한 평균값이다.
SELECT emp_no, from_date, salary,
			AVG(salary) OVER() AS avg_salary
FROM salaries
WHERE emp_no = 10001
LIMIT 5;

//윈도우 함수보다 LIMIT 절이 먼저 실행되길 원한다면 FROM 절에 서브쿼리를 작성해야 한다.
//서브쿼리로 인해 WHERE절을 만족하는 5개의 레코드가 나오고, 5개의 레코드에 대한 AVG가 집계된다.
SELECT emp_no, fromt_date, salary,
			AVG(salary) OVER() AS avg_salary
FROM (SELECT * FROM salaries WHERE emp_no = 10001 LIMIT 5) s2;
```

### 기본 사용법

```sql
AGGREAGTE_FUNC() OVER(<partition> <order>) AS window_func_col
```

- PARTITION BY [칼럼]: [칼럼]을 기준으로 소그룹 파티션을 나누겠다. 즉, 집계 함수가 [칼럼]별로 따로따로 집계됨.
    - AVG(salary) OVER(PARTITION BY dept_no) → dept_no=1인 소그룹의 salary 평균, dept_no=2인 소그룹의 salary 평균 …
- ORDER BY [칼럼]: 집계 결과를 [칼럼]을 기준으로 정렬하겠다.
    - AVG(salary) OVER(ORDER BY dept_no) → dept_no=1인 소그룹 먼저 출력. 단 AVG 값은 전체 레코드의 salary 평균으로 동일함

### 성능 이슈

MySQL의 윈도우 함수는 8.0에서 처음 도입되었기 때문에, 아직 인덱스를 이용한 최적화가 부족합니다.

배치 프로그램 같은 경우에는 사용해도 무방하겠지만, 온라인 트랜잭션 처리에서는많은 레코드에 대해 윈도우 함수를 적용하는 것은 피해야 합니다.(레코드 수가 적다면 메모리에서 빠르게 처리될 것이므로 성능에 대한 고민 X)

## 2-4. 트랜잭션과 갱신

```jsx
INSERT INTO [테이블 이름] ([필드1], [필드2], [필드3] ... )
VALUES ([값1], [값2], [값3] ... ) 

// WHERE 구는 필수 아님
DELETE FROM Address
WHERE [조건문];

UPDATE [테이블 이름]
SET [필드1] = [값1], [필드2] = [값2];
WHERE [조건문];
```

## 3. 추가로 알아볼 것

- [x]  윈도우 함수의 자세한 개념
- [ ]  서브 쿼리 문제 5개 풀어보기
