# [SQL 레벨업] 7장 - 서브쿼리

# 1. 서브쿼리가 일으키는 폐해

서브쿼리는 SQL 내부에서 작성되는 일시적인 테이블입니다. RDB는 테이블과 서브쿼리를 기능적인 관점에서 동일한 것으로 취급합니다. 따라서 데이터베이스 사용자는 자신이 다루는 대상 테이블이 테이블인지, 서브쿼리인지, 혹은 뷰인지 구분하지 않고 자유롭게 사용할 수 있습니다.

이런 기능적인 유연함이 있지만, 서브쿼리는 성능이 나쁜 경향이 있습니다. 서브쿼리를 사용할 경우 일어나는 성능 문제 패턴을 확인하고, 어떤 점을 신경써야하는지 알아보겠습니다.

<br>

## 1-1. 서브쿼리의 문제점

서브쿼리의 문제점은 서브쿼리가 셀체적인 데이터를 저장하지 않는다는 점에서 기인합니다.

<br>

### 연산 비용 추가

실체적인 데이터를 저장하지 않는다는 것은 서브쿼리에 접근할 때 마다 SELECT 구문을 실행해서 데이터를 만들어야 한다는 뜻입니다. 이로 인해 SELECT 구문을 실행하는 비용이 추가됩니다. 서브쿼리의 내용이 복잡해질수록 실행 비용은 더 높아집니다.

<br>

### 데이터 I/O 비용 발생

서브쿼리의 연산 결과를 메모리에 저장해야 하는데, 데이터 양이 많다면 이를 디스크에 저장하게 됩니다. 이로 인해 접근 속도가 급격하게 떨어집니다.

<br>

### 최적화를 받을 수 없음

서브쿼리는 물리 테이블이 아니기 때문에 명시적인 제약 또는 인덱스가 작성되어있지 않습니다. 따라서 옵티마이저가 쿼리를 해석하기 위해 필요한 정보를 서브쿼리에서 얻을 수 없습니다.

> 이 문제를 해결하기 위해 Oracle에서는 뷰 병합이라는 방법을 사용한다고 합니다.
> 

서브쿼리는 이렇게 성능적인 문제점이 있습니다. 먼저 어느 경우에 서브쿼리를 사용하면 성능이 떨어지는지 알아보고, 어떤 경우에는 서브쿼리가 성능적으로 좋은지도 알아보겠습니다.

<br>

## 1-2. 서브쿼리 의존증

서브쿼리의 문제점을 보기 위해 예시를 살펴보겠습니다. Receipts 테이블에는 순번(seq) 필드를 갖는데 구입 시기가 오래될수록 작은 값을 갖습니다. 이때 고객별 최소 순번 레코드를 구하는 경우를 생각해봅시다.

```sql
CREATE TABLE Receipts
(cust_id CHAR(1) NOT NULL,
seq INTEGER NOT NULL,
price INTEGER NOT NULL,
PRIMARY KEY (cust_id, seq));

-- 생성된 테이블과 레코드의 예시는 INSERT 쿼리로 대신함
INSERT INTO Receipts VALUES ('A', 1, 500);
INSERT INTO Receipts VALUES ('A', 2, 1000);
INSERT INTO Receipts VALUES ('A', 3, 700);
INSERT INTO Receipts VALUES ('B', 5, 100);
INSERT INTO Receipts VALUES ('B', 6, 5000);
INSERT INTO Receipts VALUES ('B', 7, 300);
INSERT INTO Receipts VALUES ('B', 9, 200);
INSERT INTO Receipts VALUES ('B', 12, 1000);
INSERT INTO Receipts VALUES ('C', 10, 600);
INSERT INTO Receipts VALUES ('C', 20, 100);
INSERT INTO Receipts VALUES ('C', 45, 200);
INSERT INTO Receipts VALUES ('C', 70, 50);
INSERT INTO Receipts VALUES ('D', 3, 2000);

--구하고자 하는 답
cust_id | seq | price
---------------------
A       | 1   | 500
B       | 5   | 100
C       | 10  | 600
D       | 3   | 2000
```

간단하게 생각해보면, 고객들의 최소 순번 값을 저장하는 서브쿼리(R2)를 만들고, 기존 Receipts 테이블과 결합하는 방법이 있습니다.

```sql
SELECT R1.cust_id, R1.seq, R1.price
FROM Receipts R1 
INNER JOIN
    (SELECT cust_id, MIN(seq) AS min_seq
     FROM Receipts
     GROUP BY cust_id) R2
ON R1.cust_id = R2.cust_id AND R1.seq = R2.min_seq;
```

이 방식은 간단하지만, 가독성이 떨어진다는 문제점과 성능상이 문제점이 있습니다. 이러한 쿼리의 성능이 나쁜 이유로 다음과 같은 4가지를 꼽을 수 있습니다.

1. 서브쿼리는 대부분 일시적인 영역(메모리, 디스크)에 확보되므로 오버헤드가 생긴다.
2. 서브쿼리는 인덱스 또는 제약 정보를 가지지 않기 때문에 최적화되지 못한다.
3. 이 쿼리는 결합을 필요로 하기 때문에 비용이 높고 실행 계획 변동 리스크가 발생한다.
4. Receipts 테이블에 스캔이 두 번 필요하다.

이러한 문제는 실행 계획에서도 살펴볼 수 있습니다. Oracle의 실행계획을 보겠습니다.
![KakaoTalk_Image_2023-11-07-19-36-32](https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/4538d6c2-0565-4a64-bec7-14bd6f8b3946)


테이블에 두번 접근하는 것과 결합을 위해 결합 알고리즘을 사용한다는 것을 확인할 수 있습니다. 그렇다면 성능이 더 좋으면서도 간단해서 읽기 쉬운 코드는 어떻게 작성해야 하는 것일까요?

한편, MySQL 5.6 이상의 버전에서는 동일한 쿼리의 실행계획을 살펴보면 Materialize라는 최적화가 적용된 것을 볼 수 있습니다.
<img width="1213" alt="스크린샷 2023-11-07 오후 7 35 23" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/6af72dba-a443-4dc9-8bde-29c55438009b">

> Materialize : MySQL 5.6 버전에 추가된 셀렉트 타입입니다. **Semi-join Materializaion 최적화**가 되었다는 것을 의미합니다.
> 

<br>

### 상관 서브쿼리는 답이 될 수 없다

먼저 정답이 아닌 쿼리부터 보겠습니다. 바로 상관 서브쿼리를 사용한 동치 변환입니다.

```sql
SELECT cust_id, seq, price
FROM Receipts R1
WHERE seq = (SELECT MIN(seq)
    FROM Receipts R2
    WHERE R1.cust_id = R2.cust_id);
```

Oracle의 실행 계획을 보면, 상관 서브쿼리를 사용해도 테이블 접근이 두 번 발생하는 것을 볼 수 있습니다. 상관 서브쿼리에도 결합이 사용되기 때문입니다.

![KakaoTalk_Image_2023-11-07-19-39-20](https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/a92dfdcd-01ae-4313-bc5a-f3ab48f0b147)


R2에 대한 접근이 인덱스 온리 스캔이라고 하더라도 테이블 2회 접근에 근본적인 성능 개선은 일어나지 않습니다.

<br>

### 윈도우 함수로 결합을 제거

일단 개선해야 하는 부분은 Receipts 테이블에 대한 접근을 1회로 줄이는 것입니다. SQL 튜닝에서 가장 중요한 것이 I/O를 줄이는 것입니다.

```sql
SELECT cust_id, seq, price
FROM (SELECT cust_id, seq, price,
	ROW_NUMBER() OVER(PARTITION BY cust_id ORDER BY seq) AS row_seq
	FROM Receipts) WORK
WHERE WORK.row_seq = 1;
```
![KakaoTalk_Image_2023-11-07-19-42-20](https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/c7498e6a-9720-4f2f-a31c-e91cd2ebcb7f)

실행 계획을 보면 윈도우 함수에서 정렬을 사용하는것이 추가되었긴 했지만, 테이블 접근이 1회로 감소한 것을 알 수 있습니다. 하지만 이전에도 MIN 함수를 위한 임시 영역을 사용했으므로 큰 차이는 없습니다.

<br>

## 1-3. 장기적 관점에서의 리스크 관리

최초의 쿼리에 비해 윈도우 함수를 사용하는 것이 성능상으로 얼마나 개선되었는지 쉽게 단언할 수 없습니다. 하지만 확실한 것은 I/O을 줄이는 것이 SQL 튜닝의 기본입니다.

어쨋거나 튜닝을 통해 기존의 결합을 제거했는데요, 이로 인해 성능 향상뿐만 아니라 성능의 안정성 확보도 기대할 수 있습니다. 

- 결합 알고리즘의 변동 리스크
- 환경 요인에 의한 지연 리스크(인덱스, 메모리, 매개변수 등)

<br>

### 알고리즘 변동 리스크

결합 알고리즘은 크게 3가지가 있고, 테이블의 크기 등을 고려해 옵티마이저가 자동으로 결정합니다. 때문에 시간이 감에 따라 알고리즘이 변동될 리스크가 있고, 성능이 좋아질수도 있지만 반대로 악화되는 경우도 있습니다. 또한 같은 알고리즘이 계속해서 선택되는 경우에도, 테이블의 크기가 커짐에 따라 TEMP 탈락 현상이 발생할 위험이 있습니다.

<br>

### 환경 요인에 의한 지연 리스크

Nested Loops는 내부 테이블 결합 키에 인덱스가 걸려있다면, Hash의 경우 워킹 메모리를 늘려주면 성능이 개선될 수 있습니다. 하지만 이러한 튜닝이 불가능한 환경이 있을 수 있고, 트레이드 오프를 발생시킬 수도 있습니다.

다시 말해, 결합을 사용한다는 것은 곧 장기적 관점에서 고려해야 할 리스크를 늘리게 된다는 뜻입니다.

따라서 엔지니어는 비기능적(성능적) 요인도 생각하며 다음을 기억해야합니다.

- 실행 계획이 단순할수록 성능이 안정적이다.
- 엔지니어는 기능(결과)뿐만 아니라 비기능적인 부분(성능)도 보장할 책임이 있다.

<br>

## 1-4. 서브쿼리 의존증 - 응용편

이번에는 앞서 사용한 Receipts 테이블에 대해 순번의 최솟값, 최댓값을 가지는 레코드끼리 price 필드의 차이를 구해봅시다. 원하는 결과는 다음과 같습니다.

```sql
cust_id | diff
---------------
A       | -200
B       | -900
C       |  550
D       |  0
```

<br>

### 다시 서브쿼리 의존증

서브쿼리를 사용한다면 다음과 같이 최솟값의 집합과 최댓값의 집합을 찾아 고객 ID를 키로 결합할 수 있습니다.

```sql
SELECT TMP_MIN.cust_id, TMP_MIN.price - TMP_MAX.price AS diff
FROM (SELECT R1.cust_id, R1.seq, R1.price
	FROM Receipts R1
	INNER JOIN
		(SELECT cust_id, MIN(seq) AS min_seq
		FROM Receipts
		GROUP BY cust_id) R2
		ON R1.cust_id = R2.cust_id
		AND R1.seq = R2.min_seq) TMP_MIN
INNER JOIN (SELECT R3.cust_id, R3.seq, R3.price
	FROM Receipts R3
	INNER JOIN
		(SELECT cust_id, MAX(seq) AS min_seq
		FROM Receipts
		GROUP BY cust_id) R4
		ON R3.cust_id = R4.cust_id
		AND R3.seq = R4.min_seq) TMP_MAX
ON TMP_MIN.cust_id = TMP_MAX.cust_id;
```

이전에 비해 쿼리가 상당히 복잡해지고, 서브 쿼리도 4개로 늘어 테이블 접근이 4회로 늘어난 것을 볼 수 있습니다. MySQL의 실행 계획을 살펴보겠습니다.
<img width="1215" alt="스크린샷 2023-11-07 오후 7 45 24" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/c49600f2-32ac-4bd9-960a-4d247df055ab">

<br>

### 레코드 간 비교에서도 결합은 불필요

이 쿼리의 개선 포인트는 앞에서와 마찬가지로 ‘테이블 접근과 결합을 얼마나 줄일 수 있는지’입니다. 이번에는 윈도우 함수에 추가로 CASE 식도 함께 사용합니다.

```sql
SELECT cust_id,
	SUM(CASE WHEN min_seq = 1 THEN price ELS 0 END)
	- SUM(CASE WHEN max_seq = 1 THEN price ELSE 0 END) AS diff
FROM (SELECT cust_id, price,
		ROW_NUMBER() OVER(PARTITION BY cust_id ORDER BY seq) AS min_seq,
		ROW_NUMBER() OVER(PARTITION BY cust_id ORDER BY seq DESC) AS max_seq
	FROM Receipts) WORK
WHERE WORK.min_seq = 1 OR WORK.max_seq = 1
GROUP BY cust_id;
```

이렇게 하면 결합이 사라지고, 서브쿼리도 하나로 줄었습니다. 이때 한 가지 트릭으로 CASE 식을 씁니다. 다른 레코드에 있는 값끼리는 뺄셈할 수 없습니다. 따라서 GROUP BY cust_id로 한 개의 레코드로 집약했습니다. 이 때 최솟값과 최댓값을 다른 필드에 할당해주는 부분이 CASE 식입니다.

> MySQL의 실행계획을 보면 Rows 수가 0인 Full Scan이 여럿 보이는데, 책에서 Oracle의 실행계획을 보며 설명하는 "테이블 스캔 횟수가 1"로 감소한 것을 유츄해볼 수 있습니다.
> 

<img width="1212" alt="스크린샷 2023-11-07 오후 7 48 56" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/7b53dfad-5d00-4a77-81f3-4f628bc0394d">

테이블 스캔 횟수가 1회로 감소했습니다. 윈도우 함수로 정렬이 2회 발생하지만, 결합이 반복하는 것보다는 실행 계획의 안정성도 확보할 수 있으므로 좋은 거래입니다.

<br>

## 1-5. 서브쿼리는 정말 나쁠까?

그렇다고 서브쿼리가 항상 나쁜 것은 아닙니다. 오히려 서브쿼리를 사용하지 않으면 해결할 수 없는 상황도 많습니다. 그리고 처음 쿼리를 고민할 때는 먼저 서브쿼리를 사용하는 쪽으로 풀어보는 게 이해하기 쉽습니다.

하지만 서브쿼리로 해결하는 바텀업 사고방식은 SQL과 어울리지 않기 때문에, 실제로 효율적인 코드가 되지는 않습니다.

<br>

# 2. 서브쿼리 사용이 더 나은 경우

이번에는 서브쿼리를 사용하는 편이 성능 측면에서 더 나은 경우를 살펴보겠습니다. 바로 결합과 관련된 쿼리입니다. 결합은 결합 대상 레코드 수를 줄이는 것이 중요한데, 옵티마이저가 이러한 것을 잘 판별하지 못할 수 있습니다. 이럴 때 서브쿼리로 직접 연산 순서를 명시해주면 성능적으로 좋은 결과를 얻을 수 있습니다.

<br>

## 2-1. 결합과 집약 순서

예시로 회사와 사업소를 관리하는 테이블을 사용하겠습니다.

```sql
CREATE TABLE Companies
(co_cd CHAR(3) NOT NULL,
district CHAR(1) NOT NULL,
CONSTRAINT pk_Companies PRIMARY KEY (co_cd));

-- 생성된 테이블과 레코드의 예시는 INSERT 쿼리로 대신함
INSERT INTO Companies VALUES('001', 'A');
INSERT INTO Companies VALUES('002', 'B');
INSERT INTO Companies VALUES('003', 'C');
INSERT INTO Companies VALUES('004', 'D');
```

```sql
CREATE TABLE Shops
(co_cd CHAR(3) NOT NULL,
shop_id CHAR(3) NOT NULL,
emp_nbr INTEGER NOT NULL,
main_flg CHAR(1) NOT NULL,
CONSTRAINT pk_Shops PRIMARY KEY (co_cd, shop_id));

-- 생성된 테이블과 레코드의 예시는 INSERT 쿼리로 대신함
INSERT INTO Shops VALUES('001', '1', 300, 'Y');
INSERT INTO Shops VALUES('001', '2', 400, 'N');
INSERT INTO Shops VALUES('001', '3', 250, 'Y');
INSERT INTO Shops VALUES('002', '1', 100, 'Y');
INSERT INTO Shops VALUES('002', '2',  20, 'N');
INSERT INTO Shops VALUES('003', '1', 400, 'Y');
INSERT INTO Shops VALUES('003', '2', 500, 'Y');
INSERT INTO Shops VALUES('003', '3', 300, 'N');
INSERT INTO Shops VALUES('003', '4', 200, 'Y');
INSERT INTO Shops VALUES('004', '1', 999, 'Y');
```

이 두개의 테이블은 1:N 관계를 나타냅니다. 문제는 이러한 두 개의 테이블을 사용해, 회사마다 주요 사업소(main_flg 필드가 Y인 사업소)의 직원 수를 구해 다음과 같은 결과를 얻는 것입니다.

```jsx
co_cd | district | sum_emp
--------------------------
001   | A        | 550
002   | B        | 100
003   | C        | 1100
004   | D        | 999
```

<br>

### 두 가지 방법

첫 번째는 결합부터 하고 집약을 하는 방법입니다.

```sql
SELECT C.co_cd, MAX(C.district), SUM(emp_nbr) AS sum_emp
FROM Companies C
INNER JOIN Shops S
ON C.co_cd = S.co_cd
WHERE main_flg = 'Y'
GROUP BY C.co_cd;
```

첫 번째 경우의 PostgreSQL의 실행 계획을 살펴보겠습니다.
![KakaoTalk_Image_2023-11-07-19-55-13_002](https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/93720da6-d7ea-4205-9a45-fa3daed1495a)

두 번째 방법은 집약을 먼저 하고 결합하는 방법입니다.

```sql
SELECT C.co_cd, C.district, sum_emp
FROM Companies C
INNER JOIN 
	(SELECT co_cd, SUM(emp_nbr) AS sum_emp
	FROM Shops
	WHERE main_flg = 'Y'
	GROUP BY co_cd) CSUM
ON C.co_cd = CSUM.co_cd;
```

두 번째 방법의 PostgreSQL의 실행 계획을 살펴보겠습니다.

![KakaoTalk_Image_2023-11-07-19-55-12_001](https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/57f94e99-6cf6-4e01-98a3-c67b4034b595)

첫 번째 방법은 회사 테이블과 사업소 테이블의 결합을 먼저 수행하고, 결과에 GROUP BY를 적용해서 집약했습니다. 반면 두 번째 방법은 먼저 사업소 테이블을 집약해서 직원 수를 구하고, 회사 테이블과 결합했습니다.

두 방식은 같은 결과를 만들어내므로 기능적 관점에서는 같지만, 성능적인 측면에서는 서로 다릅니다.

<br>

### 결합 대상 레코드 수

두 가지 방법은 성능적으로 큰 차이를 보일 수 있습니다. 판단 기준은 바로 결합 대상 레코드 수 입니다.

일단 첫 번째 방법의 경우 결합 대상 레코드 수는 다음과 같습니다.

- 회사 테이블: 레코드 4개
- 사업소 테이블: 레코드 10개

두 번째 방법의 경우 결합 대상 레코드 수는 다음과 같습니다.

- 회사 테이블: 레코드 4개
- 사업소 테이블(CSUM): 레코드 4개

중요한 것은 CSUM 뷰가 회사 코드로 집약되어 4개로 압축되었다는 것입니다. 따라서 결합 비용을 낮출 수 있습니다. 

예시의 레코드 수가 작아서 실행 속도에는 큰 차이가 보이지 않습니다. 하지만 테이블 규모가 다음과 같다면 어떨까요?

- 회사 테이블: 레코드 1,000개
- 사업소 테이블(main_flg = ‘Y’ 조건을 만족하는 레코드): 레코드 500만 개
- 사업소 테이블(CSUM으로 집약된 뷰의 레코드): 레코드 1,000개

이렇게 회사 테이블에 비해 사업소 테이블의 규모가 매우 크다면, 일단 결합 대상 레코드 수를 집약하는 편이 I/O 비용을 줄일 수 있습니다. 물론 이 상황에선 집약 비용이 더 커지겠지만, TEMP 탈락이 발생하지 않으면 괜찮은 트레이드 오프입니다.

한편 첫 번째 방법과 두 번째 방법의 성능 차이는 환경에도 의존합니다. 여기서 환경이란 테이블 레코드 수 뿐만 아니라 하드웨어, 미들웨어, 결합 알고리즘 등의 요소를 모두 포함합니다. 따라서 실제 개발을 할 때는 이러한 요인들을 모두 고려해서 성능 테스트를 하는 것이 좋습니다. 

그렇지만 튜닝 선택지 중 하나로 ‘사전에 결합 레코드 수를 압축한다’라는 방법을 알아둔다고 손해는 없겠죠?

<br>

# 3. 추가로 알아볼 것

## 3-1. 뷰 병합이란?

뷰 병합이란, Oracle에서 서브쿼리를 최적화하는 기법으로, 서브쿼리를 실행할 때 서브쿼리 내부 로직과 외부 로직을 결합해서 하나의 실행 계획을 만드는 것입니다.

<br>

## 3-2. MySQL의 서브쿼리 최적화

https://jojoldu.tistory.com/520

Materialize란 **Semi-join Materializaion 최적화**가 되었다는 것을 의미합니다.

그 이전의 버전에서는 IN 절 내에 서브쿼리가 존재할 경우 매 레코드마다 서브쿼리를 실행시키는 형태로 수행되었기 때문에 매우 비효율적이었습니다. 5.6 에서부터 추가된 MATERIALIZED는 IN 절 내의 서브쿼리를 임시테이블로 만들어 조인을 하는 형태로 최적화를 해줍니다.

이로 인해 WHERE … IN (서브쿼리) 형태의 쿼리가 성능 향상되었습니다.
