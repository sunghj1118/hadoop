# 12 하이브 활용하기

> 하이브란 데이터 분석가들이 쉽게 데이터를 분석할 수 있도록 페이스북에서 개발한 데이터 웨어하우스 패키지다. 하둡에 저장된 데이터를 쉽게 처리할 수 있게 한다.
> 

# 12.1 하이브 아키텍처

하이브 클라이언트는 다음으로 구성된다.

-CLI

-Hive Server

-Web Interface

![!\[Untitled\](12%20%E1%84%92%E1%85%A1%E1%84%8B%E1%85%B5%E1%84%87%E1%85%B3%20%E1%84%92%E1%85%AA%E1%86%AF%E1%84%8B%E1%85%AD%E1%86%BC%E1%84%92%E1%85%A1%E1%84%80%E1%85%B5%20642ff1c37bf34a8e8f0025f0d7ded442/Untitled.png)](images/ch12-hive_arch.png)

하이브의 서버의 경우 JDBC, ODBC,  Thrift로 개발된 클라이언트가 하이브 서비스를 활용할 수 있게 Thrift (쓰리프트) 서비스를 제공합니다. 

또한, 하이브는 Metastore라는 저장소를 만들어 하둡에서 처리된 메타데이터의 구조를 메타스토어에 저장합니다. 

# 12.2 하이브 설치

하이브 공식 사이트에서 원하는 패키지 파일을 내려받으면 된다.

# 12.3 HiveQL

하이브는 하이브QL라고 SQL문과 매우 유사한 언어를 제공한다. 

차이점은 다음과 같다:

1. 하이브에서 사용하는 데이터가 HDFS에 저장되며, 한번 저장된 파일은 수정이 안된다. 따라서 UDPATE와 DELETE를 사용할 수 없으며, INSERT도 비어있는 곳에 쓰거나 덮어 쓰는것만 가능하기 때문에 INSERT OVERWRITE라는 키워드를 사용한다.
2. SQL에선 어느 절에서도 서브쿼리를 사용할 수 있지만, 하이브QL은 FROM 절에서만 서브쿼리 사용 가능.
3. SQL의 뷰는 업데이트할 수 있고, 구체화된 뷰 또한 비구체화된 뷰를 지원합니다. 그러나, 하이브QL의 뷰는 읽기 전용이며, 비구체화된 뷰만 지원합니다.
4. SELECT 문을 사용할 때 HAVING 절을 사용할 수 없습니다.
5. 저장 프로시저 (stored procedure)를 지원하지 않습니다. 대신 맵리듀스 스크립트를 실행할 수 있습니다. 

해당 장에서는 ‘미국 항공 운항 지연 데이터’를 분석하기 위한 하이브QL 쿼리문을 작성한다.

## 12.3.1 테이블 생성

하둡은 HDFS에 저장된 파일에 직접 접근해서 처리하지만, 하이브는 메타스토어에 저장된 테이블을 분석한다. 

따라서 데이터를 조회하기 전에 먼저 테이블을 생성해야 한다. 

```sql
CREATE TABLE airline delay(Year INT, Month INT, … )
COMMENT 'The table consists of ....'
PARTITIONED BY (delayYear INT)
```

테이블 생성

설명 추가

파티션 설정

위에 해당하는 HiveQL 문들이다.

## 12.3.2 데이터 업로드

2008.csv를 업로드

## 12.3.3 집계 함수

하이브의 내장 집계 함수

| 함수 | 내용 |
| --- | --- |
| COUNT(1), COUNT(*) | 전체 데이터 건수를 반환 |
| COUNT(DISTINCT COLUMN) | 유일한 칼럼값의 건수를 반환 |
| SUM(COLUMN) | 칼럼값의 합계를 반환 |
| SUM(DISTINCT COLUMN) | 유일한 칼럼값의 합계를 반환 |
| AVG(COLUMN) | 칼럼값의 평균을 반환 |
| AVG(DISTINCT COLUMN) | 유일한칼럼값의 평균을 반환 |
| MAX(COLUMN) | 칼럼의 최댓값을 반환 |
| MIN(COLUMN) | 칼럼의 최솟값을 반환 |

> 미국 항공 운항 지연 데이터 가운데 2008년도의 지연 건수를 조회
> 

```sql
SELECT COUNT(1)
FROM airline_delay
WHERE delayYear=2008;
```

> 연도와 월별로 도착지연 건수를 조회
> 

```sql
SELECT Year, Month, COUNT(*) AS arrive_delay_count
FROM airline_delay
WHERE delayYear = 2008
AND ArrDelay > 0
GROUP BY Year, Month;
```

> 2008년도의 평균 지연 시간을 연도와 월별로 계산
> 

```sql
SELECT Year, Month, AVG(ArrDelay) AS avg_arrive_delay_time, AVG(DepDelay) AS avg_departure_delay_time
FROM airline_delay
WHERE delayYear = 2008
AND ArrDelay > 0
GROUP BY Year, Month;
```

## 12.3.4 JOIN

맵리듀스로 조인을 구현하려면 수십 줄 이상의 클래스를 작성해야 합니다. 하지만 하이브를 이용하면 간단하게 조인을 수행할 수 있다. 다만, 다음 제약사항이 존재한다.

- 하이브는 EQ 조인만 지원.

> EQ 조인이란? 두 테이블 대상으로 동일성을 비교한 후, 그 결과를 기준으로 조인하는 것.
> 
- 하이브는 FROM 절에 테이블 하나만 지정할 수 있고, ON 키워드를 사용해 조인을 처리해야 한다.

### 1. INNER JOIN (내부 조인)

> 출발 공항 코드(Origin)와 도착 공항 코드(Dest), airport_code 테이블을 조인해 공항별 지연 건수를 계산하는 쿼리문입니다.
> 

```sql
SELECT A.Year, A.Origin, B.AirPort, A.Dest, C.AirPort, COUNT(*)
FROM airline_delay A
JOIN airport_code B ON (A.Origin = B.Iata)
JOIN airport_code C ON (A.Dest = C.Iata)
WHERE A.ArrDelay > 0
GROUP BY A.Year, A.Origin, B.AirPort, A.Dest, C.AirPort;
```

### 2. OUTER JOIN (외부 조인)

> 왼쪽 외부 조인 쿼리
> 

```sql
SELECT A.Year, A.UniqueCarrier, B.Code, B.Description
FROM airline_delay A
LEFT OUTER JOIN carrier_code2 B ON (A.UniqueCarrier = B.Code)
WHERE A.UniqueCarrier = 'WN'
LIMIT 10;
```

## 12.3.5 버킷 활용

하이브는 전체 데이터를 조회하지 않고 일부 데이터만 무작위로 샘플링할 수 있는 BUCKETS라는 모델을 제공합니다. 버킷은 테이블을 생성할 때 다음과 같은 형태로 선언한다:

```sql
CLUSTERED BY (칼럼) INTO 버킷 개수 BUCKETS;
```

> 20개의 버킷을 생성하여 테이블을 생성합니다.
> 

```sql
CREATE TABLE airline_delay2(Year INT, Month INT, UniqueCarrier STRING, ArrDelay INT, DepDelay INT)
CLUSTERED BY (UniqueCarrier) INTO 20 BUCKETS;
```