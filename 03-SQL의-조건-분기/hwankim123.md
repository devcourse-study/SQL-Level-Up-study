# 1. UNION을 사용한 쓸데없이 긴 표현

UNION 을 조건 분기에 사용하는 것은 WHERE 구만 조금씩 다른 여러 개의 SELECT 구문을 합쳐서, 복수의 조건에 일치하는 하나의 결과 집합을 얻고 싶을 때 사용합니다. 이런 방법은 큰 문제를 작은 문제로 나눌 수 있다는 점에서 생각하기 쉽다는 장점이 있지만, 성능척이 측면에서 굉장히 큰 단점이 있습니다. 외부적으로는 하나의 SQL 구문을 실행하는 것처럼 보이지만, 내부적으로는 여러 개의 SELECT 구문을 실행하는 실행 계쵝으로 해석되기 때문에 I/O 비용이 크게 증가합니다.

따라서 조건 분기를 할 때 UNION을 사용해도 좋을지 여부는 신중히 검토해야 합니다. UNION과 CASE를 사용한 조건 분기를 비교하면서, 어떤 경우에 어떤 것을 사용하는 것이 좋을지 알아보겠습니다.

<br>

## 1-1. UNION을 사용한 조건 분기와 관련된 간단한 예제

예를 들어 아래와 같은 테이블에 대해 2001년 이전은 세금이 포함되지 않은 가격을, 2002년부터는 세금이 포함된 가격을 price(가격) 필드로 표시하도록 출력한다고 생각해봅시다.

| item_id(상품 ID) | year(연도) | item_name(상품 이름) | price_tax_ex(세전 가격) | price_tax_id(세후 가격) |
| --- | --- | --- | --- | --- |
| 100 | 2000 | 머그컵 | 500 | 525 |
| 100 | 2001 | 머그컵 | 520 | 546 |
| 100 | 2002 | 머그컵 | 600 | 630 |
| 100 | 2003 | 머그컵 | 600 | 630 |
| 101 | 2000 | 티스푼 | 500 | 525 |
| 101 | 2001 | 티스푼 | 500 | 525 |
| 101 | 2002 | 티스푼 | 500 | 525 |
| 101 | 2003 | 티스푼 | 500 | 525 |
| 102 | 2000 | 나이프 | 600 | 630 |
| 102 | 2001 | 나이프 | 550 | 577 |
| 102 | 2002 | 나이프 | 550 | 577 |
| 102 | 2003 | 나이프 | 400 | 420 |

```jsx
// 예상 결과
item_name | year | price
------------------------
머그컵    | 2000 | 500
머그컵    | 2001 | 520
머그컵    | 2002 | 630
머그컵    | 2003 | 630
티스푼    | 2000 | 500
티스푼    | 2001 | 500
티스푼    | 2002 | 525
티스푼    | 2003 | 525
나이프    | 2000 | 600
나이프    | 2001 | 550
나이프    | 2002 | 577
나이프    | 2003 | 420
```

UNION을 사용하면 year을 조건으로 다음과 같은 쿼리를 작성할 수 있습니다.

조건이 배타적이므로 증복된 레코드가 발생하지 않습니다. 쓸데없는 정렬 등의 처리를 하지 않아도 되므로 UNION ALL을 사용했습니다.

```jsx
SELECT item_name, year, price_tax_ex AS price
FROM Items
WHERE year <= 2001
UNION ALL
SELECT item_name, year, price_tax_in AS price
FROM Items
WHERE year >= 2002;
```

하지만 문제는 다른 곳에 있습니다. 첫 번째는 쿼리가 쓸데없이 길다는 것입니다. 거의 같은 쿼리를 두 번이나 실행하고 있습니다. 두 번째 문제는 성능입니다.

<br>

### UNION을 사용했을 때의 실행 계획 문제

UNION으로 묶은 쿼리는 실행 계획 상으로는 Items 테이블에 2회 접근하게 됩니다. 그리고 그때마다 TABLE ACCESS FULL이 발생하므로, 읽어들이는 비용도 테이블 크기에 따라 선형적으로 증가합니다.

![KakaoTalk_Photo_2023-10-23-17-16-24](https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/f9943f9b-b1e2-4eae-8124-c9b4b46ab22d)

<br>

### 정확한 판단 없는 UNION 사용 회피

간단하게 레코드 집합을 합칠 수 있다는 점에서 UNION은 굉장히 편리한 도구이지만, 정확한 판단 없이 SELECT 구문 전체를 여러 번 사용해서 코드를 길게 만드는 것은 쓸데없는 테이블 접근을 발생시키며 SQL의 성능을 나쁘게 만듭니다.

<br>

## 1-2. WHERE 구에서 조건 분기를 하는 사람은 초보자

이 문제를 SELECT 구만으로 조건 분기를 해서 최적화해보겠습니다.

```jsx
SELECT item_name, year,
			 CASE WHEN year <= 2001 THEN price_tax_ex
					  WHEN year >= 2002 THEN price_tax_in END AS price
FROM items;
```

CASE 식을 사용한 쿼리의 결과는 위의 UNION 쿼리와 동일하지만, CASE 식을 사용한 쿼리가 성능적으로 훨씬 좋습니다.

<br>

## 1-3. SELECT 구를 사용한 조건 분기의 실행 계획
![KakaoTalk_Photo_2023-10-23-17-16-29](https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/0d896c9c-9ad2-4922-ba81-6b1c6c980524)
Items 테이블에 대한 접근이 1회로 줄어든 것을 확인할 수 있습니다. 또한 SQL 쿼리의 가독성도 좋아졌습니다.

이처럼 SQL 구문의 성능이 좋은지 나쁜지는 반드시 실행 계획 레벨에서 판단해야 합니다.

<br>

# 2. 집계와 조건 분기

## 2-1. 집계 대상으로 조건 분기

집계를 수행하는 쿼리를 작성할 때, 쓸데없이 길어지는 경우를 자주 볼 수 있습니다.

예를 들어 지역별로 남녀 인구를 기록하는 Population 테이블을 생각해봅시다. 

| perfecture(지역 이름) | sex(성별) | pop(인구) |
| --- | --- | --- |
| 성남 | 1 | 60 |
| 성남 | 2 | 40 |
| 수원 | 1 | 90 |
| 수원 | 2 | 100 |
| 광명 | 1 | 100 |
| 광명 | 2 | 50 |
| 일산 | 1 | 100 |
| 일산 | 2 | 100 |
| 용인 | 1 | 20 |
| 용인 | 2 | 200 |

이 테이블을 아래와 같은 레이아웃으로 변경하는 방법을 생각해봅시다.

| prefecture | pop_men | pop_wom |
| --- | --- | --- |
| 수원 | 90 | 100 |
| 일산 | 100 | 100 |
| 성남 | 60 | 40 |
| 광명 | 100 | 50 |
| 용인 | 20 | 200 |

<br>

### UNION을 사용한 방법

이 문제를 절차지향적인 사고방식을 가진다면, 일단 남성의 인구를 지역별로 구하고, 여성의 인구를 지역별로 구한 뒤 머지(merge)하는 방법을 생각할 것입니다.

```sql
SELECT prefecture, SUM(pop_men) AS pop_men, SUM(pop_wom) AS pop_wom
FROM (SELECT prefecture, pop AS pop_men, null AS pop_wom
			FROM Population
			WHERE sex = '1'
			UNION
			SELECT prefecture, NULL AS pop_men, pop AS pop_wom
			FROM Population
			WHERE sex = '2')TMP
GROUP BY prefecture;

// 서브 쿼리 TMP의 결과는 아래처럼 남성과 여성의 인구가 별도의 레코드로 나옵니다.
prefecture | pop_men | pop_wom
------------------------------
성남        |60       |
성남        |         |40
수원        |90       |
수원        |         |100
...
```

이 함수의 실행 계획을 확인하면 Population **테이블에 풀 스캔이 2회 수행**되는 것을 확인할 수 있습니다.

<img width="1330" alt="스크린샷 2023-10-20 오후 6 11 54" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/a6e0305c-7e12-46f1-9352-fe02d5fea62c">

<br>

### 집계의 조건 분기도 CASE 식을 사용

이 문제는 CASE의 응용 방법으로 유명한 표측/표두 레이아웃 이동 문제입니다. 여기에서 표측은 좌측의 제목을 말하고, 표두는 이차원 표 상단 제목을 말합니다. CASE 식을 집약 함수 내부에 포함시켜서 ‘남성 인구’와 ‘여성 인구’ 필터를 만듭니다.

```sql
// SELECT 구문을 사용한 조건 분기의 경우 쿼리가 굉장히 간단합니다.
SELECT prefecture,
			 SUM(CASE WHEN sex='1' THEN pop ELSE 0 END) AS pop_men,
			 SUM(CASE WHEN sex='2' THEN pop ELSE 0 END) AS pop_wom
FROM Population
GROUP BY prefecture;
```

CASE 식으로 작성한 쿼리는 실행계획 또한 간단해집니다. 즉 성능이 향상됩니다. 실행계획을 확인하면 풀 스캔이 1회밖에 수행되지 않습니다. UNION을 사용한 쿼리에 비해 I/O 접근 이 절반으로 줄어들게 되는 것입니다.

<img width="1309" alt="스크린샷 2023-10-20 오후 5 44 16" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/14473c71-3f08-43cd-9fec-b97650252ff3">

이렇게 CASE 식을 사용하면 UNION을 사용한 절차지향적인(복잡한) 쿼리가 단순해질 뿐만 아니라 성능또한 좋아지는 것을 알 수 있습니다.

<br>

## 2-2. 집약 결과로 조건 분기

집약에 조건 분기를 적용하는 또 하나의 패턴으로, 집약 결과에 조건 분기를 수행하는 경우가 있습니다.

<br>

### Employees 테이블

예를 들어 지역별로 남녀 인구를 기록하는 Employees 테이블을 생각해봅시다. 

| emp_id(직원ID) | team_id(팀 ID) | emp_name(직원 이름) | team(팀) |
| --- | --- | --- | --- |
| 201 | 1 | A | 상품기획 |
| 201 | 2 | A | 개발 |
| 201 | 3 | A | 영업 |
| 202 | 2 | B | 개발 |
| 203 | 3 | C | 영업 |
| 204 | 1 | D | 상품기획 |
| 204 | 2 | D | 개발 |
| 204 | 3 | D | 영업 |
| 204 | 4 | D | 관리 |
| 205 | 1 | E | 상품기획 |
| 205 | 2 | E | 개발 |

<br>

### 원하는 결과

여기에 다음과 같은 조건에 맞춰 결과를 만드는 것을 생각해봅시다.

1. 소속된 팀이 1개라면 해당 직원은 팀의 이름을 그대로 출력한다.
2. 소속된 팀이 2개라면 해당 직원은 ‘2개의 겸무’라는 문자열을 출력한다.
3. 소속된 팀이 3개 이상이라면 해당 직원은 ‘3개 이상을 겸무’라는 문자열을 출력한다.

| emp_name | team |
| --- | --- |
| A | 개발 |
| B | 3개 이상을 겸무 |
| C | 3개 이상을 겸무 |
| D | 영업 |
| E | 2개를 겸무 |

<br>

### UNION을 사용한 조건 분기

```sql
SELECT emp_name, MAX(team) AS team
	FROM Employees
	GROUP BY emp_name
	HAVING COUNT(*) = 1
UNION
SELECT emp_name, '2개를 겸무' AS team
	FROM Employees
	GROUP BY emp_name
	HAVING COUNT(*) = 2
UNION
SELECT emp_name, '3개 이상을 겸무' AS team
	FROM Employees
	GROUP BY emp_name
	HAVING COUNT(*) >= 3;
```

이 쿼리는 조건 분기가 레코드 값이 아닌 집합의 레코드 수에 적용되어있습니다. 그렇기 때문에 조건 분기가 WHERE 절이 아닌 HAVING에 지정되었습니다. 하지만 UNION으로 머지하고 있는 이상, 구문 레벨의 분기일 뿐입니다. 따라서 WHERE 구를 사용할 때와 크게 다르지 않습니다.

<br>

### UNION의 실행계획

3개의 쿼리를 머지하는 쿼리이므로 Employees에 대한 접근도 3번 발생하는 것을 확인할 수 있습니다.

<img width="1324" alt="스크린샷 2023-10-20 오후 5 44 53" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/b217f3b4-79f1-4fd5-bb73-bb4f3608771f">

<br>

### CASE 식을 사용한 조건 분기

```sql
SELECT emp_name,
	CASE WHEN COUNT(*) = 1 THEN MAX(team) // 의문)왜 MAX함수?
		   WHEN COUNT(*) = 2 THEN '2개를 겸무'
			 WHEN COUNT(*) >= 3 THEN '3개 이상을 겸무'
	END AS team
FROM employees
GROUP BY emp_name;
```

> MAX 집계를 사용하지 않을 때 발생하는 오류:
[42000][1055] Expression #2 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'sql_levelup.employees.team' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
> 

<br>

### CASE 식을 사용한 조건 분기의 실행 계획

<img width="1345" alt="스크린샷 2023-10-20 오후 6 12 37" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/1134f08c-69a0-4649-ad99-7ac8eebca524">

이렇게 CASE 식을 사용하면 테이블에 접근 비용을 1/3으로 줄일 수 있습니다. 또한 GROUP BY의 HASH 연산도 3회에서 1회로 줄었습니다. 이를 가능하게 하는 것은 집약 결과를 CASE 식의 입력으로 사용했기 떄문입니다.

> WHERE 구에서 조건 분기를 하는 사람은 초보자
HAVING 구에서 조건 분기를 하는 사람은 초보자
> 

<br>

# 3. 그래도 UNION이 필요한 경우

지금까지 UNION이 나쁘다는 식으로 설명했습니다. 하지면 UNION을 사용하지 않으면 안되는 경우도 있습니다. 또한 UNION을 사용하는 것이 오히려 성능적으로 좋은 경우도 있습니다.

<br>

## 3-1. UNION을 사용할 수밖에 없는 경우

여러 테이블을 머지하는 경우 UNION을 사용할 수 밖에 없습니다. CASE를 사용하는 방법도 있지만, FROM구에서 테이블을 결합해야 하기 때문에 성능적으로 악영향이 발생합니다(UNION을 사용한다면 발생하지 않습니다). 따라서 실행 계획 등을 확인해 어떤 것이 더 좋은지 명확하게 확인해줘야 합니다.

> 의문: UNION과 JOIN의 차이
> 

<br>

## 3-2. UNION을 사용하는 것이 성능적으로 더 좋은 경우

UNION 이외의 다른 방법으로도 풀 수 있지만, UNION을 사용하는 것이 더 성능이 좋을 수 있는 경우입니다.

바로 인덱스와 관련된 경우입니다. UNION을 사용했을 때 좋은 인덱스(압축을 잘 하는 인덱스)를 사용하지만, 이외의 경우에는 테이블 풀 스캔이 발생하는 경우입니다.

> 의문: 압축을 잘 하는 인덱스? 분포가 넓은. PK같은거
> 

예를 들어 3개의 날짜 필드 date_1 ~ date_3과, 그것과 짝을 이루는 플래그 필드 flg_1 ~ flg_3 을 가진 테이블 TreeElements을 생각해봅시다.

| key | name | date_1 | flg_1 | date_2 | flg_2 | date_3 | flg_3 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | a | 2013-11-01 | T |  |  |  |  |
| 2 | b |  |  | 2013-11-01 | T |  |  |
| 3 | c |  |  | 2013-11-01 | F |  |  |
| 4 | d |  |  | 2013-12-30 | T |  |  |
| 5 | e |  |  |  |  | 2013-11-01 | T |
| 6 | f |  |  |  |  | 2013-12-01 | F |

이 테이블에서 date_1 ~ date_2이 특정 날짜(예를 들어 2013년 11월 1일)을 값으로 갖고 있고 대칭되는 플래그 필드의 값이 ‘T’인 레코드를 선택한다고 합시다.

```sql
//예시 결과
key | name | date_1     | flg_1 | date_2     | flg_2 | date_3     | flg_3
---------------------------------------------------------------------------
1   | a    | 2013-11-01 | T     |            |       |            |
2   | b    |            |       | 2013-11-01 | T     |            |
5   | e    |            |       |            |       | 2013-11-01 | T     
```

이 결과를 만들어내는 쿼리에는 무엇이 있는지, 성능은 어떻게 되는지 알아보겠습니다.

<br>

### UNION을 사용한 방법

세 개의 SELECT 구문을 UNION으로 머지하면 됩니다.

```sql
SELECT key, name, date_1, flg_1, date_2, flg_2, date_3, flg_3
FROM TreeElements
WHERE date_1 = '2013-11-01' AND flg_1 = 'T'
UNION
SELECT key, name, date_1, flg_1, date_2, flg_2, date_3, flg_3
FROM TreeElements
WHERE date_2 = '2013-11-01' AND flg_2 = 'T'
UNION
SELECT key, name, date_1, flg_1, date_2, flg_2, date_3, flg_3
FROM TreeElements
WHERE date_3 = '2013-11-01' AND flg_3 = 'T';
```

문제는 바로 성능과 실행 계획입니다.

이 쿼리를 최적의 성능으로 수행하려면 다음과 같은 필드 조합에 인덱스가 필요합니다.

```sql
CREATE INDEX IDX_1 ON TreeElements (date_1, flg_1);
CREATE INDEX IDX_2 ON TreeElements (date_2, flg_2);
CREATE INDEX IDX_3 ON TreeElements (date_3, flg_3);
```

인덱스가 있으면 UNION을 사용했을 때 실행계획은 다음과 같습니다.(테이블 레코드 수가 적다면 옵티마이저가 인덱스 스캔을 선택하지 않고 테이블 풀 스캔을 선택할 가능성이 큽니다. 테이블 레코드가 커지고, 인덱스로 레코드 압축이 잘 된다면, 인덱스 스캔이 훨씬 좋은 성능을 냅니다.)

<img width="1350" alt="스크린샷 2023-10-20 오후 8 03 06" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/791252c0-8dee-4bdc-a3fc-6ccf5943bd1c">

3개의 SELECT 구문 모두 인덱스를 사용합니다. 이렇게 되면 테이블의 레코드 수가 많고, 각각의 WHERE 구의 검색 조건에서 레코드 수를 많이 압축할수록, 테이블의 풀 스캔보다도 훨씬 빠른 접근 속도를 기대할 수 있습니다.

<br>

### OR을 사용한 방법

```sql
SELECT key, name, date_1, flg_1, date_2, flg_2, date_3, flg_3
FROM TreeElements
WHERE (date_1 = '2013-11-01' AND flg_1 = 'T') OR
			(date_2 = '2013-11-01' AND flg_2 = 'T') OR
			(date_3 = '2013-11-01' AND flg_3 = 'T');
```

결과는 UNION을 사용했을 때와 같습니다. 하지만 실행 계획이 크게 다릅니다.

<img width="1351" alt="스크린샷 2023-10-20 오후 8 05 41" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/6ffb993b-6144-4dfb-94a2-74dc50ef714a">

SELECT 구문이 하나로 줄었기 때문에 테이블에 대한 접근도 1회로 줄었습니다. 하지만 이때 인덱스가 사용되지 않고, 그냥 테이블 풀 스캔이 사용됩니다. 이렇게 WHERE 구문에서 OR을 사용하면 해당 필드에 부여된 인덱스를 사용할 수 없습니다.

> 의문: WHERE 구문에서 OR을 사용하면 해당 필드에 부여된 인덱스를 사용할 수 없습니다. → 이건 그냥 이렇다 라고 외우는건가? 이게 규칙인가?
https://hyeyul-k.tistory.com/12
> 

따라서 이러한 경우에 UNION과 OR의 성능 비교는 결국 **3회의 인덱스 스캔과 1회의 테이블 풀 스캔 중에서 어떤 것이 더 빠른지**에 대한 문제가 됩니다. 이는 테이블 크기와 검색 조건에 따른 선택 비율에 따라 답이 달라집니다. 하지만 테이블이 크고, WHERE 조건으로 선택되는 레코드의 수가 충분히 작다면 UNION이 더 빠릅니다.

<br>

### IN, CASE를 사용한 방법

IN의 매개변수로는 단순한 스칼라뿐만 아니라 이렇게 (a, b, c)와 같은 값의 리스트(배열)을 입력할 수도 있습니다.

```sql
explain SELECT _key, name, date_1, flg_1, date_2, flg_2, date_3, flg_3
FROM TreeElements
WHERE ('2013-11-01', 'T') IN((date_1, flg_1), (date_2, flg_2), (date_3, flg_3));
```

이 쿼리는 OR을 사용할 때와 실행 계획이 같습니다. 

<img width="1348" alt="스크린샷 2023-10-20 오후 8 07 08" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/72458b23-c06f-4beb-ab15-a3b3638296e1">

한편, CASE를 사용하여 풀 수도 있는데, CASE 쿼리도 같은 결과를 만들어냅니다. 하지만 실행 계획은 OR, IN을 사용할 때와 같습니다.

```sql
SELECT _key, name, date_1, flg_1, date_2, flg_2, date_3, flg_3
FROM TreeElements
WHERE CASE
          WHEN date_1 = '2013-11-01' THEN flg_1
          WHEN date_2 = '2013-11-01' THEN flg_2
          WHEN date_3 = '2013-11-01' THEN flg_3
          ELSE NULL END = 'T';
```

<img width="1354" alt="스크린샷 2023-10-20 오후 8 12 19" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/35731532/5ff03356-ae11-4428-bff9-9b121709f68e">

<br>

# 4. 절차 지향형과 선언형

지금까지 UNION을 사용한 조건 분기와 그 이외의 방법을 비교해서 살펴보았습니다. 결론은 예외적인 몇 가지 상황을 제외하면 UNION을 사용하지 않는 것이 성능적으로도 좋고 가독성도 좋다는 것입니다. 원래 UNION은 조건 분기를 위해 만들어진 것이 아니므로 당연한 결과입니다. 반대로 CASE 식은 조건 분기를 위해 만들어졌으므로 , CASE 식을 사용하는 것이 훨씬 자연스럽습니다.

<br>

## 4-1. 구문 기반과 식 기반

SQL을 절차지향적으로 작성하면 구문(statement)을 단위로 생각하게 됩니다. 반대로 SQL을 선언형으로 작성하면 식(expression)을 단위로 생각합니다.

하지만 SQL의 기본적인 체계는 선언형입니다. 이 세계의 주역은 ‘구문’이 아니라 ‘식’입니다. 

SQL 구문의 각 부분(SELECT, FROM, WHERE, GROUP BY, HAVING, OEDER BY)에 작성하는 것은 식입니다. SQL 구문 내부에는 식을 작성하지, 구문을 작성하지는 않습니다.

<br>

## 4-2. 선언형의 세계로 도약

절차 지향형 세계에서 선언형 세계로 도약하는 것이 곧 SQL 능력 향상의 핵심입니다. 이후의 내용에서 그러한 도약을 경험할 수 있게 됩니다.

<br>

## 궁금한 점

- UNION ALL을 사용하면 왜 최적화가 되는지(8강)(https://brownbears.tistory.com/126)
```
UNION은 결과 테이블을 합칠 때 중복을 제거하지만, UNION ALL은 중복을 제거하지 않습니다.
여기에서 UNION은 이미 SELECT된 결과를 가지고 UNION하기 때문에 SELECT되기 전의 테이블이나 레코드에 대한 정보는 알 수 없습니다.
그래서, 중복 여부의 판단은 SELECT된 튜플들에 속해있는 모든 컬럼의 값들 자체가 중복 체크의 기준이 되는 것입니다.
```
- UNION, UNION ALL의 동작 과정
1. 최종 UNION [ALL | DISTINCT] 결과에 적합한 임시 테이블을 메모리 테이블로 생성
2. UNION 또는 UNION DISTINCT 의 경우, 임시 테이블의 모든 컬럼으로 Unique Hash 인덱스 생성
3. 서브쿼리 1 실행 후 결과를 임시테이블에 복사 
4. 서브쿼리 2 실행 후 결과를 임시테이블에 복사 
5. 3,4 번 과정에서 임시 테이블이 특정 사이즈 이상으로 커지면 임시 테이블을 디스크 임시 테이블로 변경 
6. 임시 테이블을 읽어서 클라이언트에 결과 전송 
7. 임시 테이블 삭제

GROUP BY의 연산 원리(임시 테이블을 가지고 언제는 해시, 정렬 등등)(https://willseungh0.tistory.com/162)

왜 OR절을 활용하는 쿼리는 인덱스로 잘 조회하는지
```
인덱스를 이용해 쿼리를 실행할 때, 대부분 옵티마이저는 테이블별로 하나의 인덱스만 사용하는 실행 계획을 수립합니다.
인덱스 머지 실행 계획을 사용하면 하나의 테이블에 대해 2개 이상의 인덱스를 이용해 쿼리를 처리합니다.
쿼리에 사용된 각각의 조건이 서로 다른 인덱스를 사용할 수 있고, 그 조건을 만족하는 레코드 건수가 많을 것으로 예상될 때 MySQL 서버는 인덱스 머지 실행 계획을 선택합니다.
```

- UNION과 JOIN의 차이
> UNION은 여러 개의 SELECT 쿼리 결과를 합치는 기능
> JOIN은 여러 테이블을 조합하여 하나의 결과로 보여주는 기능
