# 8강 UNION을 사용한 쓸데없이 긴 표현 
---
## 조건 분기를 구현할 때 왜 UNION을 사용할까?
일반적으로 조건 분기는 복수의 조건에 일치하는 하나의 결과 집합을 얻고 싶을 때 사용합니다.

여러 개의 SELECT 문으로 **쪼개어** 각 조건을 표현할 수 있기 때문에 생각하기가 쉽습니다.

즉, `UNION`을 이용하는 방법은 조건 분기를 SQL 문으로 나타낼 때 가장 처음 나타내기 **쉬운** 방법입니다.


## UNION을 사용하면 어떤 점이 문제일까?
UNION을 사용하면 SQL 문을 짜기 쉬운데 왜 사용이 지양될까요?

바로 **성능**적인 측면에서 문제가 있기 때문입니다.

UNION을 이용하면 내부적으로 여러 개의 SELECT 구문을 실행하도록 해석됩니다.

이는 곧 **I/O** 횟수의 증가로, 성능적으로 손실이 발생합니다.

그렇다면 실제 예제를 통해 **UNION**과 **CASE** 구문을 비교해보겠습니다.

다음과 같은 상품 테이블이 있습니다.

|상품 ID|연도|상품 이름|세전 가격|세후 가격|
| ---- | ---- | ---- | ---- | ---- |
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

해당 테이블에서 **2001** 년 전까지는 **세전** 가격을 출력하고, **2002** 년부터는 **세후** 가격을 출력하는 **요구사항**이 있습니다.

**[UNION 사용 예제]**
```SQL
SELECT 상품_이름, 연도, 세전_가격 AS 가격
FROM Items
WHERE 연도 <= 2001
UNION ALL
SELECT 상품_이름, 연도, 세후_가격 AS 가격
FROM Items
WHERE 연도 >= 2002;
```

조건이 배타적이므로 중복되는 레코드가 발생하지 않습니다.
이 때문에 UNION ALL을 사용했습니다.

해당 코드의 **문제점**은 2가지 입니다.

1. **쓸데 없이 긴 쿼리**
해당 코드는 거의 비슷한 쿼리를 2번이나 반복하고 있습니다.
이는 해당 코드를 길고, 읽기 힘들게 합니다.

2. **성능**
위 쿼리의 실행계획을 살펴보면 다음과 같습니다. (MySQL 기준)

<img width="1391" alt="스크린샷 2023-10-20 오후 12 09 02" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/58359383/a4c93cae-60c9-4ea2-a7a7-19ebf047d43a">


테이블에 2회 접근하는 것을 알 수 있고, 그 때마다 [**FULL TABLE SCAN**(type: ALL)](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#jointype_all)을 하는 것을 알 수 있습니다.

**FULL TABLE SCAN**을 하게 되면 읽는 비용이 테이블의 크기에 따라 **선형적**으로 증가합니다.

위의 2가지 문제로 인해 쉬운 사용법에도 UNION을 통한 조건 분기는 **지양**됩니다.

그렇다면 **CASE** 구문으로는 어떻게 작성할까요?

**[CASE 사용 예제]**
```SQL
SELECT 상품_이름, 연도
	CASE WHEN 연도 <= 2001 THEN 세전_금액
		WHEN 연도 >= 2002 THEN 세후_금액
	END AS 가격
FROM Items;
```

위의 쿼리도 UNION 쿼리와 같은 결과를 출력합니다.

하지만, 해당 쿼리가 성능적으로 훨씬 좋습니다.
그 이유는 뭘까요? **실행 계획**을 통해 살펴보겠습니다.

<img width="1395" alt="스크린샷 2023-10-20 오후 12 21 14" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/58359383/ba0f3e3e-e9c6-462a-aa8d-e8e41ea84976">

위의 실행 계획을 보면 테이블에 대한 접근이 **1회**로 줄어든 것을 확인할 수 있습니다.

테이블 접근 횟수가 줄어 성능적으로 큰 개선이 있습니다.
물론, 가독성도 좋아졌습니다.

이처럼 **실행 계획**을 통해 비교하면 성능이 좋은지 나쁜지 판단할 수 있습니다.

**[구문적인 관점에서 비교]**
책에서는 UNION과 CASE의 쿼리를 구문적인 관점에서 비교하는 부분이 있습니다.

UNION을 이용한 분기는 SELECT **구문**을 기본 단위로 분기하고 있습니다.

책에서는 구문을 기본 단위로 사용하는 점에서 **절차 지향형**의 발상을 벗어나지 못한 방법이라고 소개하고 있습니다.

반면, CASE의 경우 식을 바탕으로 분기를 처리하고 있습니다.

책의 저자는 절차 지향 관점의 **구문**에서 **식**으로 사고를 변경하는 것의 중요성을 강조했습니다.

저는 구문 단위로 작성함으로써 I/O를 하는 횟수가 늘기 때문에 저자가 위와 같이 강조하는 것이라고 생각했습니다.

SELECT와 같은 큰 단위로 작성하는 것보다 I/O를 하지 않는 **식**의 단위로 작성하는 중요성을 깨달았습니다.

# 집계와 조건 분기

**집계**를 수행하는 쿼리에서도 조건 분기를 하는 경우가 있습니다.

이 경우에도 UNION과 CASE 두 가지 방법으로 쿼리를 작성할 수 있습니다.
예제를 통해 각각의 케이스를 살펴보겠습니다.


### 1. 집계 대상으로 조건 분기

다음과 같은 테이블이 있습니다.

Population

|지역_이름|성별|인구|
| ---- | ---- | ---- |
| 성남 | 남 | 60 |
| 성남 | 여 | 40 |
| 수원 | 남 | 90 |
| 수원 | 여 | 100 |
| 광명 | 남 | 100 |
| 광명 | 여 | 50 |
| 일산 | 남 | 100 |
| 일산 | 여 | 100 |
| 용인 | 남 | 20 |
| 용인 | 여 | 200 |

지역에 따라 남성과 여성의 인구수를 출력하고자 할 때 어떻게 작성할 수 있을까요?

**[UNION 활용]**
절차 지향적으로 풀고자 한다면, 남성과 여성의 지역별 인구 수를 각각 구하고 합치는 것으로 생각할 수 있습니다.

```SQL
SELECT 지역_이름, SUM(남성_인구) AS 남성_인구, SUM(여성_인구) AS 여성_인구  
FROM (SELECT 지역_이름, 인구 AS 남성_인구, NULL AS 여성_인구  
      FROM Population  
      WHERE 성별 = '남'  
      UNION  
      SELECT 지역_이름, NULL AS 남성_인구, 인구 AS 여성_인구  
      FROM Population  
      WHERE 성별 = '여') AS Temp  
GROUP BY 지역_이름;
```

서브 쿼리 Temp 는 남성과 여성의 인구가 다음과 같이 별도의 레코드에 나옵니다.

|지역_이름|남성 인구|여성 인구|
| ---- | ---- | ---- |
| 성남 | 60 | |   
| 수원 | 90 | |   
| 광명 | 100 | |   
| 일산 | 100 | |   
| 용인 | 20 | |   
| 성남 |  | 40 |  
| 수원 |  | 100 |  
| 광명 |  | 50 |  
| 일산 |  | 100 |  
| 용인 |  | 200 |

따라서 GROUP BY를 통해 하나의 레코드로 집약하였습니다.
그러면 다음과 같이 정상적으로 출력됩니다.

|지역_이름|남성 인구|여성 인구|
| ---- | ---- | ---- |
| 성남 | 60 | 40 |  
| 수원 | 90 | 100 |  
| 광명 | 100 | 50 |  
| 일산 | 100 | 100 |  
| 용인 | 20 | 200 |

위와 같이 쿼리를 작성하면 절차지향적으로 작성하는데에 문제가 있습니다.

위의 쿼리의 실행 계획을 살펴보면 다음과 같습니다.
<img width="1366" alt="스크린샷 2023-10-20 오후 2 11 39" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/58359383/49e3c728-e6d9-4a10-a9fe-529c9d4cf472">

Population 테이블에 Full Table Scan을 **2**회 한 것을 볼 수 있습니다.

그렇가면 CASE 문을 이용하면 어떻게 작성할 수 있을까요?

**[CASE 활용]**
이 문제는 CASE 식의 응용 방법으로 유명한 표측/표두 레이아웃 이동 문제입니다.

다음과 같이 CASE 식을 집약 함수 내부에 포함시켜 남성 인구와 여성 인구 필터를 만듭니다.

```SQL
SELECT 지역_이름,  
    SUM(CASE WHEN 성별 = '남' THEN 인구 ELSE 0 END) AS 남성_인구,  
    SUM(CASE WHEN 성별 = '여' THEN 인구 ELSE 0 END) AS 여성_인구  
FROM Population  
GROUP BY 지역_이름;
```

위의 쿼리의 실행 계획을 살피면 다음과 같습니다.

<img width="913" alt="스크린샷 2023-10-20 오후 2 11 08" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/58359383/efeff7d6-12fc-4f7f-8c7e-93373bbe0ab1">

Population 테이블로의 풀 스캔이 1회로 감소한 것을 확인할 수 있습니다.

UNION 방식에 비해 I/O 비용이 절반으로 감소되었습니다. (캐시 등을 고려하지 않았을 때)

### 2. 집약 결과로 조건 분기

집계 함수의 결과로 조건 분기를 수행하는 경우가 존재합니다.

예를 들어 다음과 같이 직원과 직원이 소속된 팀을 관리하는 테이블이 있다고 가정하겠습니다.

**Employees**

| emp_id(직원ID) | team_id(팀ID) | emp_name(직원 이름) | team(팀) |
| ---- | ---- | ---- | ---- |
| 201 | 1 | Joe | 상품기획 |
| 201 | 2 | Joe | 개발 |
| 201 | 3 | Joe | 영업 |
| 202 | 2 | Jim | 개발 |
| 203 | 3 | Carl | 영업 |
| 204 | 1 | Bree | 상품기획 |
| 204 | 2 | Bree | 개발 |
| 204 | 3 | Bree | 영업 |
| 204 | 4 | Bree | 관리 |
| 205 | 1 | Kim | 상품기획 |
| 205 | 2 | Kim | 개발 |

다음과 같은 조건에 맞춰 쿼리를 작성하려면 어떻게 해야 할까요?

1. 소속된 팀이 1개라면 해당 직원은 팀의 이름을 그대로 출력한다.
2. 소속된 팀이 2개라면 해당 직원은 '2개를 겸무'라는 문자열을 출력한다.
3. 소속된 팀이 3개 이상이라면 해당 직원은 '3개 이상을 겸무'라는 문자열을 출력한다.

**[UNION 활용]**

```SQL
SELECT emp_name,  
       MAX(team) AS team  
FROM Employees  
GROUP BY emp_name  
HAVING COUNT(*) = 1  
UNION  
SELECT emp_name,  
       '2개를 겸무' AS team  
FROM Employees  
GROUP BY emp_name  
HAVING COUNT(*) = 2  
UNION  
SELECT emp_name,  
       '3개를 겸무' AS team  
FROM Employees  
GROUP BY emp_name  
HAVING COUNT(*) >= 3;
```

위의 쿼리의 실행 계획을 살펴보겠습니다.

<img width="1279" alt="스크린샷 2023-10-20 오후 2 39 52" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/58359383/9cbe2864-acd1-4e87-b249-1c8adaa57ba7">

테이블에 대한 접근이 3번 발생하는 것을 확인할 수 있습니다.

**[CASE 활용]**

위의 문제를 CASE로 풀면 다음과 같습니다.

```SQL
SELECT emp_name,  
    CASE WHEN COUNT(*) = 1 THEN MAX(team)  
	    WHEN COUNT(*) = 2 THEN '2개를 겸무'  
        WHEN COUNT(*) >= 3 THEN '3개를 겸무'  
        END AS team  
FROM Employees  
GROUP BY emp_name;
```

위의 쿼리의 실행 계획을 살펴보면 다음과 같습니다.

<img width="920" alt="스크린샷 2023-10-20 오후 2 43 47" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/58359383/01580569-f264-45d7-b1ae-5676165fa0d2">

테이블에 접근하는 횟수가 1로 줄어들었습니다.
추가로, GROUP BY에 의해 임시 테이블이 생성되는 것도 1회로 줄어들었습니다.

집계 함수의 결과를 1개의 레코드로 압축함으로써 CASE 식의 매개 변수에 넣을 수 있습니다.

이와 같이 조건 분기를 할 때는 식의 형태로 만들어서 CASE 구문을 활용하는 것이 성능상으로 중요한 포인트입니다.

이 책의 저자는 'WHERE 구에서 조건 분기를 하는 사람은 초보자'라는 말에 덧붙여, 'HAVING 구에서 조건 분기하는 사람도 초보자'라는 말을 전하고 있습니다.


# 그래도 UNION이 필요한 경우
---
위의 내용까지는 UNION을 사용하는 것이 나쁘다라는 메세지를 전했는데요.

'그렇다면 항상 나쁜가'에 대한 의문을 남길 수 있습니다.

물론 항상 나쁜 것은 아닙니다. 
그 경우를 이제 소개해겠습니다.

## 1. UNION을 사용할 수밖에 없는 경우

> 서로 다른 여러 테이블에 대해 SELECT한 결과를 합칠 때

여러 개의 테이블에서 검색한 결과를 합칠 때는 UNION이 필요합니다.

```SQL
SELECT name  
	FROM People  
	WHERE address = '서울시'  
UNION ALL  
SELECT emp_name  
	FROM Employees  
	WHERE team = '상품기획';
```

위와 같이 경우를 예시로 들 수 있습니다.

물론, **CASE** 식을 사용하여 이를 구현할 수 있습니다. 
FROM 구에서 테이블을 결합하면 CASE 식을 사용해 원하는 결과를 구할 수 있습니다.

하지만 **필요 없는 결합**이 발생해서 성능적으로 악영향이 발생합니다.

이 경우에는 실행 계획 등을 확인하여 어떤 방식이 더 좋은지 확인해야 합니다.

## 2. UNION을 사용하는 것이 성능적으로 더 좋은 경우

혹시 UNION을 사용하는 것이 성능상으로 이득인 경우가 있을까요?

바로 인덱스와 연관이 있는 경우입니다.

만약 UNION을 이용했을 때는 **좋은 인덱스**(적게 조회해도 레코드를 찾을 수 있는 인덱스, 압축을 잘하는 인덱스)를 사용하지만, 이외의 경우에는 테이블 풀 스캔을 한다면 UNION을 사용하는 것이 성능상으로 더 좋을 수 있습니다.

책에서 다음과 같은 예시를 소개하고 있습니다.

ThreeElements

| key | name | date_1 | flg_1 | date_2 | flg_2 | date_3 | flg_3 | 
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 1 | a | 2023-10-20 | 1 |  |  |  | |   
| 2 | b |  |  | 2023-10-20 | 1 |  | |   
| 3 | c |  |  | 2023-10-20 | 0 |  | |   
| 4 | d |  |  | 2023-11-20 | 1 |  | |   
| 5 | e |  |  |  |  | 2023-10-20 | 1 |  
| 6 | f |  |  |  |  | 2023-11-20 | 0 |

위와 같은 테이블이 있을 때 다음 결과를 출력하고자 합니다.

| key | name | date_1 | flg_1 | date_2 | flg_2 | date_3 | flg_3 | 
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 1 | a | 2023-10-20 | 1 |  |  |  | |   
| 2 | b |  |  | 2023-10-20 | 1 |  | |   
| 5 | e |  |  |  |  | 2023-10-20 | 1 |

(date_1, flg_1), (date_2, flg_2), (date_2, flg_2) 이렇게 3개의 조합에 **인덱스**를 걸었을 때 쿼리와 그에 대한 실행 계획은 어떻게 될까요?

**[UNION 이용]**
```SQL
SELECT `key`, name,  
    date_1, flg_1,  
    date_2, flg_2,  
    date_3, flg_3  
FROM ThreeElements  
WHERE date_1 = '2023-10-20'  
AND flg_1 = 1  
UNION  
SELECT `key`, name,  
    date_1, flg_1,  
    date_2, flg_2,  
    date_3, flg_3  
FROM ThreeElements  
WHERE date_2 = '2023-10-20'  
AND flg_2 = 1  
UNION  
SELECT `key`, name,  
    date_1, flg_1,  
    date_2, flg_2,  
    date_3, flg_3  
FROM ThreeElements  
WHERE date_3 = '2023-10-20'  
AND flg_3 = 1
```

![Pasted image 20231020160113](https://github.com/devcourse-study/SQL-Level-Up-study/assets/58359383/092252fb-3b95-4334-9fcd-78d82510e365)

**[OR 이용]**
```SQL
SELECT `key`, name,  
       date_1, flg_1,  
       date_2, flg_2,  
       date_3, flg_3  
FROM ThreeElements  
WHERE (date_1 = '2023-10-20' AND flg_1 = 1)  
   OR (date_2 = '2023-10-20' AND flg_2 = 1)  
   OR (date_3 = '2023-10-20' AND flg_3 = 1)
```

<img width="1360" alt="스크린샷 2023-10-20 오후 4 02 12" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/58359383/fd8d962f-cc74-448b-b39a-24a707c4fea8">

**[IN 이용]**
```SQL
SELECT `key`, name,  
       date_1, flg_1,  
       date_2, flg_2,  
       date_3, flg_3  
FROM ThreeElements  
WHERE ('2023-10-20', 1)  
	  IN ((date_1, flg_1),  
		  (date_2, flg_2),  
		  (date_3, flg_3));
```

<img width="1362" alt="스크린샷 2023-10-20 오후 4 02 41" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/58359383/98b05688-9774-43ee-ba6e-cf7f38f3fd22">

**[CASE 이용]**
```SQL
SELECT `key`, name,  
       date_1, flg_1,  
       date_2, flg_2,  
       date_3, flg_3  
FROM ThreeElements  
WHERE CASE WHEN date_1 = '2023-10-20' THEN flg_1  
           WHEN date_2 = '2023-10-20' THEN flg_2  
           WHEN date_3 = '2023-10-20' THEN flg_3  
           ELSE NULL END = 1;
```

<img width="1354" alt="스크린샷 2023-10-20 오후 4 03 13" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/58359383/9df5b990-3a3d-406f-8602-4af0dd060abb">

사실 책에서의 실행 계획과 제가 실습한 실행 계획이 **다르게** 나왔습니다.

책에서는 UNION이 인덱스 스캔을, 나머지 경우가 테이블 풀 스캔으로 나왔습니다.

저의 경우에는 나머지는 동일하지만 OR의 경우가 조금 다르게 나옵니다.

하지만 결국 책에서 제시하는 의미는 똑같이 전달됩니다.

위의 실행 계획의 결과를 요약하면 다음과 같습니다.

> 3회의 INDEX SCAN vs 1회의 TABLE FULL SCAN

전자의 경우 테이블의 크기가 커지고 WHERE 조건으로 선택되는 레코드 수가 작을 수록 더 빠릅니다.

따라서 **테이블의 크기**와 **검색 조건에 따른 선택 비율**(레코드 히트율)에 따라 어떤 방식을 적용할지 선택해야 합니다.


# 11강 절차 지향형과 선언형
---

대부분의 경우에는 UNION 보다는 CASE를 이용하는 것이 성능상으로 좋고, 가독성도 뛰어납니다.

SQL을 작성할 때는 '구문' 단위로 생각하지 말고 '식' 단위로 생각하는 것이 중요합니다.

이는 SQL의 기본적인 체계가 선언형이기 때문입니다.

따라서 절차 지향형의 구문에서 선언형의 식으로 도약하는 것이 SQL 능력 향상의 핵심이라고 볼 수 있습니다.


# 궁금한 점
---
### UNION ALL 은 UNION에 비해 어떤 처리를 하는걸까?

[출처](https://intomysql.blogspot.com/2011/01/union-union-all.html)

> **[UNION 및 UNION ALL 처리 과정]**
> 1. 최종 UNION [ALL | DISTINCT] 결과에 적합한 임시 테이블(Temporary table)을 메모리 테이블로 생성
> 2. UNION 또는 UNION DISTINCT 의 경우, 임시 테이블의 모든 컬럼으로 Unique Hash 인덱스 생성
> 3. 서브쿼리1 실행 후 결과를 임시 테이블에 복사
> 4. 서브쿼리2 실행 후 결과를 임시 테이블에 복사
> 5. 만약 3,4번 과정에서 임시 테이블이 특정 사이즈 이상으로 커지면 임시 테이블을 Disk 임시 테이블로 변경
> 	(이 때 Unique Hash 인덱스는 Unique B-Tree 인덱스로 변경)
> 6. 임시 테이블을 읽어서 Client에 결과 전송
> 7. 임시 테이블 삭제

RealMySQL의 저자에 의하면, MySQL에서는 임시 테이블을 생성하여 UNION을 처리하고 있습니다.

다만 UNION은 UNION ALL과는 다르게 컬럼에 대해 **해시 인덱스**를 생성하여 중복을 제거하는 과정이 추가됩니다.

따라서 UNION ALL은 해당 과정을 하지 않고 처리하기 때문에 UNION에 비해 빠른 처리 속도를 가질 수 있는 것입니다.

### UNION을 사용할 수 밖에 없는 경우에서 CASE로 어떻게 구현할까?

**[UNION 사용]**
```SQL
SELECT name  
   FROM People  
   WHERE address = '서울시'  
UNION ALL  
SELECT emp_name  
   FROM Employees  
   WHERE team = '상품기획';
```
<img width="1386" alt="스크린샷 2023-10-23 오후 5 42 23" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/58359383/00678bbe-c27b-4642-b9f8-3917a923fa8d">

**[CASE 사용]**
```SQL
SELECT DISTINCT  
  CASE    WHEN address = '서울시' THEN name  
    WHEN team = '상품기획' THEN emp_name  
  END AS result  
FROM People,Employees  
WHERE address = '서울시' OR team = '상품기획';
```
<img width="1422" alt="스크린샷 2023-10-23 오후 5 42 06" src="https://github.com/devcourse-study/SQL-Level-Up-study/assets/58359383/de7341ae-46c5-4d46-83fe-9f62a49d3900">


CASE의 경우가 조회하는 열이 더 많아 비효율적이라고 볼 수 있습니다.


# 피드백 및 느낀점
---
UNION을 조건 분기에 사용하는 것을 처음 알았습니다.

이번 장을 통해서 UNION으로 조건 분기를 처리하는 것을 지양해야 한다는 것을 알았습니다.

아직 식을 기준으로 생각하는 것이 어렵다고 느꼈습니다.

다음에 연습 문제를 풀어오면 좋을 것 같습니다.

[블로그 포스팅](https://taek-coding.tistory.com/30)
