# Overview of the Optimizer
오라클 데이터베이스가 SQL 질의를 어떻게 처리 하는지 이해하기 위해서는 optimizer를 이해하는 것이 필요하다.

## Use of the Optimizer
옵티마이저는 가능한 실행 방법을 묘사하는 실행 계획을 생성한다. 

옵티마이저는 어떤 실행 계획이 가장 효율적인지 결정하는데, 이 때 고려 사항으로는 쿼리 조건과 가능한 access path, 시스템에서 수집된 통계, 힌트를 포함한다. 오라클에 의해 처리된 SQL 질의를 위해 옵티마이저는 다음과 같은 작업을 수행한다 :
- expression과 condition의 평가
- 데이터에 대해 자세히 알아보고 이 메타데이터를 기반으로 최적화하기 위한 무결성 제약 조건 검사
- 질의 변경
- 옵티마이저 목표 선택
- 접근 경로 선택
- 조인 순서 선택

옵티마이저는 쿼리를 처리할 수 있는 가능성이 높은 방법들을 생성하고 각각의 코스트를 계산한다. 가장 낮은 코스를 가진 계획을 실행한다.

> SQL을 실행시키지 않고 실행 계획을 얻을 수 있다. 실제로 실행에 사용되는 실행 계획을 query plan이라고 한다.

## Optimizer Components
옵티마이저는 세 가지 주요 컴포넌트가 있다.
![Description of Figure 7-2 follows](https://docs.oracle.com/cd/E11882_01/server.112/e40540/img/cncpt287.gif)

옵티마이저에게 parsed query를 인풋으로 넣으면 옵티마이저는 다음과 같은 작업을 수행한다 :
1. 옵티마이저는 parsed query를 받고 가용한 접근 경로와 힌트에 기반한 잠재적인 계획의 집합을 생성한다.
2. 옵티마이저는 data dictionary의 통계에 기반한 각각의 플랜의 cost를 예측한다. Cost는 특정 계획으로 질의를 실행하는데 필요한 예상 리소스 사용에 비례하는 추정 값이다.
3. 옵티마이저는 플랜의 cost를 비교하여 가장 적은 cost의 플랜을 선택하고 row source generator에게 전달한다.

### Query Transformer
Query transformer는 옵티마이저가 더 좋은 실행 계획을 생성할 수 있게 쿼리의 form을 변경한다.
Query transformer의 인풋은 쿼리 블록의 집합인 parsed query이다.

### Estimator
Estimator는 주어진 실행 계획의 전체 cost를 결정한다. Estimator는 다음과 같은 세 가지 다른 측정 방법으로 목표를 달성한다 :
- Selectivity : 유일한 row의 수의 수치
- Cardinality : row set의 row의 수
- Cost : 이 측정은 work나 리소스 사용의 단위를 대표한다. 쿼리 옵티마이저는 disk I/O, CPU 사용, 메모리 사용을 사용한다.

통계가 가용하다면 estimator는 측정을 위해 그 값을 사용한다.

### Plan Generator
Plan generator는 제공된 쿼리를 위해 서로 다른 플랜을 생성하고 가장 적은 cost의 플랜을 선택한다. Optimizer는 분리된 쿼리 블록으로 대표되는 nested subqueries와 unmerged views 각각의 서브 플랜을 생성한다. Plan generator는 서로 다른 access path, join method, join order를 시도해보며 쿼리 블록을 위한 다양한 플랜을 생성한다.

Diagnostic tools such as the `EXPLAIN PLAN` statement enable you to view execution plans chosen by the optimizer. `EXPLAIN PLAN` shows the query plan for the specified SQL query if it were executed now in the current session. Other diagnostic tools are Oracle Enterprise Manager and the SQL*Plus `AUTOTRACE` command. [Example 7-6](https://docs.oracle.com/cd/E11882_01/server.112/e40540/sqllangu.htm#CHDHGDCG) shows the execution plan of a query when `AUTOTRACE` is enabled.

## Access Path
Access path는 데이터베이스로부터 데이터를 가져오는 방법이다. 예를 들어, 인덱스를 사용하는 쿼리는 그렇지 않은 쿼리와 다른 접근 경로를 가진다. 일반적으로, 인덱스 접근 경로는 작은 집합의 테이블 row를 회수하는데 최고의 접근 경로이다. 풀 스캔은 더 큰 부분에 접근하는데 효율적이다.

데이터베이스는 테이블로부터 데이터를 가져오기 위한 여러 접근 경로가 있다.
- Full table scan : 테이블의 모든 row를 읽으며 선택 기준에 맞지 않는 데이터는 필터링한다. 데이터베이스는 연속적으로 세그먼트의 모든 데이터 블록을 스캔하는데, 사용되는 공간과 사용되지 않는 공간을 나누는 high water mark 아래의 데이터까지 스캔한다.
- Rowid scans : Row의 rowid는 데이터파일과 row를 포함하는 데이터 블록을 특정하고, 블록 내부에서 row의 위치를 특정한다.
- Index scans : 이 스캔은 SQL 질의에 의해 접근되는 인덱싱된 컬럼 값의 인덱스를 서칭한다. 만약 질의가 인덱스의 컬럼만 조회한다면, 오라클 데이터베이스는 인덱스로부터 직접 데이터를 가져온다.
- Cluster scans : 클러스터 스캔은 indexed table cluster에 저장되어있는 데이터를 가져온다. 데이터베이스는 먼저 cluster index를 스캔하여 rowid를 가져온다. 오라클 데이터베이스는 rowid에 기반하여 row를 찾는다.
- Hash scans : Hash scan은 같은 hash 값을 가진 row끼리 같은 데이터 블록에 저장되는 hash cluster에 row를 위치시키기 위해 사용된다. 먼저 hash 함수를 통해 클러스터 키를 가져온 다음 같은 값을 가지는 데이터 블록을 스캔한다.

## Optimizer Statistics
Optimizer 통계는 데이터베이스와 데이터베이스 내의 오브젝트에 대한 상세 정보를 묘사하는 데이터의 집합이다. 

Optimizer statistics는 다음을 포함한다 :
- 테이블 통계 : row의 수, block의 수, row length 평균
- 컬럼 통계 : distinct value의 수, 데이터베이스의 분포, null의 수
- 인덱스 통계 : leaf block의 수와 index level
- 시스템 통계 : CPU와 I/O 성능

오라클 데이터베이스는 자동으로 모든 데이터베이스 오브젝트에 대한 통계를 수집한다. 그리고 자동화된 유지관리 작업으로 이 통계를 관리한다. 수동으로 DBMS_STATS 패키지를 사용하여 통계를 모을 수 있다.

Optimizer statistics는 쿼리 최적화를 위해 생성되고 데이터 딕셔너리에 저장된다. 이 통계는 동적 성능 뷰에서 보이는 성능 통계와 혼동해서는 안된다.

## Optimizer Hints
Hint는 옵티마이저에게 지시하는 것 처럼 행동하는 주석이다. 

예를 들어, 쿼리가 50 row를 반환한다고 가정하자. 어플리케이션은 초기에 오직 25개의 row만 fetch할 것이고 유저에게 전달할 것이다. 이는 `first_row(25)` 힌트를 통해 최적화 될 수 있다.

# Overview of SQL Processing
이 섹션은 오라클 데이터베이스가 SQL 질의를 처리하는 방법을 설명한다.

## Stages of SQL Processing
![Description of Figure 7-3 follows](https://docs.oracle.com/cd/E11882_01/server.112/e40540/img/cncpt250.gif)

### SQL Parsing
SQL 프로세스의 첫번째 단계는 파싱이다. 이 단계는 다른 루틴에서 사용할 수 있도록 SQL 질의를 쪼개 데이터 구조로 만드는 것을 포함한다. 

어플리케이션이 SQL 질의를 실행시키면, 어플리케이션은 parse call을 데이터베이스에 보내 질의 실행을 준비한다. Parse call은 세션의 priave SQL area를 다루는 cursor를 열거나 생성한다. cursor와 private SQL area는 PGA 안에 있다.

parse call  동안, 데이터베이스는 다음과 같은 확인을 한다의 해시 값이 동일하더라도 semantic 검사를 통해 의미가 동일한지 여부를 확인한다.

```
CREATE TABLE my_table ( some_col INTEGER );
SELECT * FROM my_table;
```

다음과 같은 SQL을 서로 다른 유저가 실행시키는 상황을 가정하자. 동일한 syntax를 실행하더라도 다른 유저가 실행시킨 create table은 서로 다른 object를 생성한다.

```
ALTER SYSTEM FLUSH SHARED_POOL;
SELECT * FROM my_table;

ALTER SESSION SET OPTIMIZER_MODE=FIRST_ROWS;
SELECT * FROM my_table;

ALTER SESSION SET SQL_TRACE=TRUE;
SELECT * FROM my_table;
```
다음 질의와 같이 서로 다른 옵티마이저 환경에서 실행된 같은 `SELECT`문이라도 Hard Parsing이 강제된다.


### SQL Optimization
Overview of the Optimizer에서 설명했듯이, 쿼리 최적화는 SQL 질의를 실행시키는 수단 중 가장 효과적인 방법을 선택하는 과정이다.

데이터베이스는 모든 유일한 DML 질의에 대해 최소 한번은 하드 파싱을 진행하고, 파싱하는 동안 최적화를 수행한다. DDL은 subquery에 DML을 포함하지 않는 한 최적화되지 않는다.

### SQL Row Source Generation
Row source generator은 옵티마이저로부터 실행 계획을 받거나 데이터베이스의 나머지 부분에서 사용할 수 있는 쿼리 계획이라는 반복 계획을 생성하는 소프트웨어다. 반복 계획은 SQL 가상 시스템에서 실행될 때 결과 집합을 생성하는 바이너리 프로그램이다.

쿼리 플랜은 단계의 조합으로 이루어져 있다. 각각의 단계는 row 집합을 리턴한다. 집합 내의 Row는 다음 단계나 마지막 단계에 사용되고, 어플리케이션으로 반환된다.

Row source는 반복적으로 row를 처리할 수 있는 통제 구조와 함께 실행 계획의 단계에 따라 반환되는 row 집합이다. Row source는 테이블, 뷰, 조인의 결과나 그루핑의 결과가 될 수 있다.

Row source generator는 row source의 집합인 row source tree를 생성한다. Row source tree는 다음과 같은 정보를 보여준다 :
- 질의에 의한 테이블 정렬
- 질의에 언급된 각각의 테이블을 위한 접근 방법
- 질의의 조인 작업에 의해 영향을 받은 테이블 조인 방법
- 필터, 정렬, 집계같은 데이터 작업


### SQL Excution
SQL 실행하는 동안, SQL engine은 각각의 row source generator에서 생성된 트리의 각각의 row source를 실행한다. 이 단계는 DML 처리에서는 필수적이다.

![[cncpt001.gif]]

위 그림은 parse tree라고 불리는 excution tree이다. 단계 단계마다의 row source의 흐름을 보여준다. 일반적으로 실행의 순서는 plan의 반대 순서인데, 따라서 plan은 밑에서부터 읽어야한다.  작업 column의 초기 공백은 계층 관계를 나타낸다. 예를 들어 작업 이름 앞에 공백 두 개가 있으면 이 연산은 공백 한 개가 선생하는 연산의 자식이다.

위 그림에서 각각의 트리의 노드는 row source 처럼 행동한다. 이는 각각의 실행 계획의 단계가 데이터베이스로부터 row를 가져오거나 row의 집합을 input으로 수용한다는 뜻이다. SQL 엔진은 각각의 row source를 다음과 같이 수행한다 :

- 검은 박스로부터 지시된 단계는 데이터베이스로부터 물리적으로 데이터를 가져온다. 이 단계는 데이터베이스로부터 데이터를 가져오기 위한 access path이거나 기술이다.
- 하얀 박스로부터 지시된 단계는 row source에서 작동된다.

실행 중에, 데이터베이스는 메모리에 데이터가 없다면 디스크로부터 데이터를 읽어 메모리에 적재한다. 데이터베이스는 또한 SQL 수행 동안 lock이나 latch같은 데이터 무결성을 보장하는데 필요한 잠금 및 래치를 제거하고 SQL 실행 중에 변경된 내용을 기록한다. SQL 질의의 마지막 처리 단계는 커서를 닫는 것이다.

## Oracle Database가 DML을 처리하는 방법

대부분의 DML 문에는 쿼리 구성 요소가 있습니다. 쿼리에서, 커서 실행은 쿼리의 결과를 결과 집합이라고 하는 행 집합에 배치합니다.  
  
결과 집합 행은 한 번에 또는 그룹으로 가져올 수 있습니다. 페치 단계에서 데이터베이스는 행을 선택하고 쿼리에 의해 요청되면 해당 행을 순서화합니다. 각 연속적인 페치는 마지막 행이 페치될 때까지 결과의 다른 행을 검색합니다.  
  
일반적으로 데이터베이스는 마지막 행을 가져올 때까지 쿼리에 의해 검색될 행의 특정 수를 결정할 수 없습니다. 오라클 데이터베이스는 가져오기 호출에 응답하여 데이터를 검색하므로 데이터베이스가 더 많은 행을 읽을수록 더 많은 작업을 수행합니다. 일부 쿼리의 경우 데이터베이스는 첫 번째 행을 가능한 빨리 반환하지만 다른 쿼리의 경우 첫 번째 행을 반환하기 전에 전체 결과 집합을 만듭니다.
## Oracle Database가 DDL을 처리하는 방법

Oracle Database는 DML과 다르게 DDL을 처리합니다. 예를 들어, 테이블을 만들 때 데이터베이스는 CREATE TABLE 문을 최적화하지 않습니다. 대신 Oracle Database는 DDL 문을 구문 분석하고 명령을 수행합니다.

데이터베이스는 데이터 사전에서 개체를 정의하는 수단이기 때문에 DDL을 다르게 처리합니다. 일반적으로 Oracle Database는 DDL 명령을 실행하기 위해 많은 재귀적 SQL 문을 구문 분석하고 실행해야 합니다. 다음과 같이 테이블을 생성한다고 가정합니다:
```
CREATE TABLE mytable (mycolumn INTEGER);
```
일반적으로 데이터베이스는 수십 개의 재귀적 문을 실행하여 이전 문을 실행합니다. 재귀적 SQL은 다음과 같은 작업을 수행합니다:
- CREATE TABLE 문을 실행하기 전에 commit을 실행한다.
- 테이블을 생성하기에 충분한 유저 권한을 가지고 있는지 확인한다.
- 테이블이 어떤 테이블스페이스에 저장되어야하는지 결정
- 테이블 스페이스 할당량을 초과하지 않았는지 확인
- 같은 이름의 스키마가 없는지 확인
- 테이블을 정의하는 row를 data dictionary에 삽입
- DDL문이 성공한 경우 commit을 실행하고 실패한 경우 rollback을 실행