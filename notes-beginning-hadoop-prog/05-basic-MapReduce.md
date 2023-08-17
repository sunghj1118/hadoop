# 05: 맵리듀스 기초 다지기

[04: 맵리듀스 시작하기](https://www.notion.so/04-318dc0f65289432e9c515254c37331cb?pvs=21)

4단원에서는 간단한 wordcount 프로그램을 작성했다면, 5단원에서는 실무에서 사용하는 훨씬 더 복잡하고 큰 파일들을 처리해봤습니다.

# 5.1 분석용 데이터 준비

ASA(미국 규격 협회)에서 공개한 미국 항공편 운항 통계 데이터를 사용합니다. TB는 아닌 11GB의 크기이지만 학습용으로 적절합니다. 해당 데이터는 연도, 월, 일, 항공편 번호 등 29개의 칼럼이 있으며 csv 형식입니다.

# 5.2 항공 출발 지연 데이터 분석

첫 번째 맵리듀스 프로그램은 연도별로 얼마나 많은 항공기가 출발이 지연됐는지 계산하는 프로그램입니다. 

| 클래스 | 입출력 구분 | 키 | 값 |
| --- | --- | --- | --- |
| 매퍼 | 입력 | offset | 항공 운항 통계 데이터 |
|  | 출력 | 운항연도, 운항월 | 출발 지연 건수 |
| 리듀서 | 입력 | 운항연도, 운항월 | 출발 지연 건수 |
|  | 출력 | 운항연도, 운항월 | 출발 지연 건수 합계 |

## 5.2.1 매퍼 구현

항공 출발 지연 건수를 계산하는 매퍼를 구현합니다. 즉, 연도와 월을 조합해서 키를 설정하고 지연시간이 0보다 크면 데이터를 1씩 출력합니다.

## 5.2.2 리듀서 구현

매퍼의 출력 데이터를 순회하며, 연도와 월별로 지연 횟수를 합산합니다. 

## 5.2.3 드라이버 클래스 구현

매퍼와 리듀서를 실행하는 클래스를 정의해줍니다.

## 결과

드라이버 클래스를 실행해본 결과 6~8월과 11~12월인 성수기에 운항 지연이 가장 크게 증가하는것을 확인할 수 있습니다.

# 5.3 항공 도착 지연 데이터 분석

첫 번째 맵리듀스 프로그램은 연도별로 얼마나 많은 항공기가 출발이 지연됐는지 계산하는 프로그램입니다. 

| 클래스 | 입출력 구분 | 키 | 값 |
| --- | --- | --- | --- |
| 매퍼 | 입력 | offset | 항공 운항 통계 데이터 |
|  | 출력 | 운항연도, 운항월 | 도착 지연 건수 |
| 리듀서 | 입력 | 운항연도, 운항월 | 도착 지연 건수 |
|  | 출력 | 운항연도, 운항월 | 도착 지연 건수 합계 |

## 5.3.1 매퍼 구현

항공 출발 지연 건수를 계산하는 매퍼를 구현합니다. 즉, 연도와 월을 조합해서 키를 설정하고 지연시간이 0보다 크면 데이터를 1씩 출력합니다.

## 5.3.2 리듀서 구현

값만 합산하면 되기 때문에 위 예제랑 똑같은 코드를 썼습니다.

## 5.3.3 드라이버 클래스 구현

매퍼와 리듀서를 실행하는 클래스를 정의해줍니다.

## 결과

출발 지연 추세랑 유사한 결과를 살펴 볼 수 있습니다. 즉, 드라이버 클래스를 실행해본 결과 6~8월과 11~12월인 성수기에 운항 지연이 가장 크게 증가하는것을 확인할 수 있습니다.

# 5.4 사용자 정의 옵션 사용

위에서는 코드가 반복되는 느낌이 있어서 사용자가 정의한 파라미터를 입력받아서 처리하도록 하겠습니다.

그러기 위해서는 하둡에서 제공하는 사용자 정의 옵션에 대한 다양한 API를 이해해야 합니다.

## 5.4.1 사용자 정의 옵션의 이해

하둡에서는 개발자가 편하게 맵리듀스 프로그램을 개발할 수 있게 여러 헬퍼 클래스를 제공합니다.

****************GenericOptionsParser****************

하둡 콘솔 명령어에서 입력한 옵션을 분석합니다. 

**Tool**

GenericOptionsParser의 콘솔 설정 옵션을 지원하기 위한 인터페이스입니다. run 메서드가 정의돼 있습니다.

**ToolRunner**

Tool 인터페이스의 실행을 도와주는 헬퍼 클래스입니다. ToolRunner는 GenericOptionsParser를 사용해 사용자가 콘솔 명령어에서 설정한 옵션을 분석하고, Configuration 객체에 설정합니다. Configuration 객체를 Tool 인터페이스에 전달한 후, Tool 인터페이스의 run 메서드를 실행합니다.

## 5.4.1 매퍼 구현

사용자가 맵리듀스 잡을 실행할 때 명령어로 입력한 workType이라는 파라미터를 확인합니다. 그래서 workType이 departure일 때는 출발 지연 칼럼을, arrival일 때는 도착 지연 칼럼을 확인합니다.

```bash
./bin/hadoop jar wikibooks-hadoop-examples.jar wikibooks.hadoop.chapter05.DelayCount -D workType=departure input departure_delay_count
```

## 결과

위 코드를 실행되고 나서 HDFS에 저장된 데이터를 확인해보면 항공 출발 지연 데이터만을 처리하던 드라이버 클래스를 실행했을 때와 동일한 결과가 나온 것을 확인할 수 있습니다.

# 5.5 카운터 사용

하둡은 맵리듀스의 job의 진행 상황을 모니터링할 수 있게 카운터 (Counter)라는 API를 제공하며, 모든 잡은 다수의 내장 카운터를 가지고 있습니다.

해당 카운터는 맵, 리듀스, 콤바이너의 입출력 레코드 건수와 바이트, 몇 개의 테스크가 성공하고 실패 했는지, 파일 시스템에서는 얼마나 많은 바이트를 읽고 썼는가에 대한 정보를 제공합니다.

## 5.5.1 사용자 정의 카운터 구현

enum 클래스를 사용하여 DelayCounters라는 카운터 그룹에 여섯개의 카운터를 정의했습니다.

## 5.5.2 매퍼 구현

항공 도착 지연 데이터를 분석하는 매퍼에 카운터를 적용 했습니다. 즉, 지연 시간이 있는지 확인만 하는것이 아니라, 일찍 도착하거나 제시간에 도착한 건수도 기록하고 카운터로 셌습니다.

## 결과

하둡에서 제공하는 맵리듀스 잡 관리자용 웹 화면에서도 delaycounters가 콘솔 화면에 출력 되었습니다.