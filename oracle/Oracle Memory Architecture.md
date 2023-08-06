## Oracle Memory Structure 소개
인스턴스가 시작되면, 오라클 데이터베이스는 메모리 공간을 할당하고 백그라운드 프로세스를 시작한다.

메모리 공간은 다음과 같은 정보를 저장한다 :
- 프로그램 코드
- 활성화 되지 않은 세션을 포함한 연결된 세션에 대한 정보.
- 프로그램 실행에 필요한 정보.
- 프로세스 사이에 교환되고 공유되어지는 잠금 데이터
- 데이터 블록과 리두 레코드 같은 캐싱된 데이터

## 기본적인 메모리 구조
오라클 데이터베이스는 다수의 메모리 공간을 가지고 있다.

기본적인 메모리 공간은 다음과 같다 :
- System global area (SGA)
SGA는 공유 메모리 구조 그룹이다. SGA 라고 알려져 있는데, 오라클 데이터베이스 인스턴스 하나의 데이터와 컨트롤 정보를 포함한다. 모든 서버와 백그라운드 프로세스는 SGA를 공유한다. 

- Program global area (PGA)
PGA는 오라클 프로세스에서 사용되는 컨트롤 정보와 데이터를 저장하는 비공유 메모리다.
오라클 데이터베이스는 오라클 프로세스가 시작됐을 때 PGA 영역을 생성한다.

서버 프로세스와 백그라운드 프로세스는 각각 하나의 PGA 영역을 가지고 있다. 각각의 PGA를 instance PGA라고 총칭한다. 데이터베이스 초기화 파라미터는 instance PGA의 크기를 설정하지만 각각의 PGA의 크기를 설정하지 않는다.

- User global area(UGA)
UGA는 유저 세션과 연관된 메모리다

- Software code areas
소프트웨어 코드 영역은 실행되거나 실행될 수 있는 코드를 저장하는 공간이다. 오라클 데이터베이스 코드는 소프트웨어 영역에 저장되어 있고, 배제적이고 안전한 영역에 저장되어 있다.

![Description of Figure 14-1 follows](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/img/cncpt217.gif "Description of Figure 14-1 follows)

## Oracle Database Memory Management
메모리 관리는 데이터베이스 변경에 따른 인스턴스 메모리 의 적절한 크기를 유지시키는 것을 포함한다. 오라클 데이터베이스는 메모리 관련된 초기화 파라미터 설정에 의해 관리된다.

메모리 관리의 기본 옵션은 다음과 같다: 
- 자동 메모리 관리
데이터베이스 인스턴스 메모리 사이즈를 정하면, 인스턴스는 자동으로 설정한 메모리 사이즈로 튜닝되고 SGA와 인스턴스 PGA에 재분배된다.
- 자동 공유 메모리 관리
이 관리 모드는 부분적으로 자동화됐다. SGA의 사이즈를 정하면 PGA의 전체 사이즈를 정하거나 PGA 각각의 work area를 결정할 수 있다.
- 매뉴얼 메모리 관리
전체 메모리 사이즈를 정하는 대신, 초기화 파라미터를 설정할 수 있다

만약 데이터베이스르르 Database Configuration Assistant(DBCA)를 통해 생성하고 기본 설치 옵션을 선택한다면 자동 메모리 관리 모드가 기본적으로 선택된다.

## User Global Area 개요
UGA는 로그온 정보나 데이터베이스 세션에서 요구되는 정보 같은 세션 변수의 할당을 위한 세션 메모리이다. 필수적으로, UGA는 세션 상태를 저장한다.

다음 그림은 UGA를 묘사한다.
![Description of Figure 14-2 follows](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/img/cncpt222.gif "Description of Figure 14-2 follows")

만약 세션이 PL/SQL 패키지를 메모리로 로드하면, UGA는 특정 시간의 모든 패키지 변수를 저장하는 변수 집합인 package 상태를 저장하고 있다. 패키지 상태는 패키지 서브 프로그램이 변수를 변경하면 바뀐다. 기본적으로, 패키지 변수는 세션의 생애주기동안 유일하고 지속된다.

OLAP page pool은 또한 UGA에 저장된다. Pool은 OLAP 데이터 페이지(데이터 블록)을 관리한다. 페이지 풀은 OLAP 세션이 시작할 때 할당되고 세션이 끝날 때 반납된다. OLAP 세션은 cube같은 다차원 객체를 질의할 때 자동적으로 열린다.

UGA는 세션 생명주기 동안 사용 가능해야한다. 그렇기 때문에 shared server connection을 사용하는 동안 UGA는 PGA에 저장될 수 없는데, 이는 PGA가 하나의 프로세스에 특정되기 때문이다. 게다가 UGA는 공유 서버 프로세스가 shared server connection 에 접근 가능하도록 SGA에 저장되어 있다. Dedicated server connection을 사용할 때에는 UGA는 PGA에 저장된다.


## Program Global Area(PGA) 개요
PGA는 시스템의 다른 프로세스나 스레드에 의해 공유되지 않는 운영 프로세스나 스레드에 특정된 메모리 공간이다. 왜냐하면 PGA 는 프로세스에 특정되어 SGA 에 절대 할당되지 않기 때문이다.

PGA 는 dedicated 나 shared server 프로세스에 의해 요구되어지는 세션 의존적인 변수를 포함하는 메모리 힙 공간이다. 서버 프로세스는 PGA에서 요구하는 메모리 구조를 할당한다.

## PGA의 내부
PGA는 서로 다른 목적에 의해 분리된다.

다음 그림은 dedicated server session의 PGA 내부이다

![[cncpt219.gif]]

### Private SQL Area
Private SQL Area는 파싱된 SQL 질의와 프로세스를 위한 다른 세션별 정보를 저장한다.

서버 프로세스가 SQL과 PL/SQL 코드를 실행시키면, 프로세스는 private SQL area를 바인드 변수값을 저장하거나 쿼리 수행 상태 정보나 쿼리 수행 공간을 저장하는데 사용한다.

PGA를 shared SQL area와 혼동하지 말자. 같은 쿼리를 여러 번 수행한다면, 쿼리 플랜은 같지만 PGA 공간은 다를 것이다.

Cursor는 private SQL area를 핸들링한다. 다음 그림과 같이, cursor가 클라이언트 측이나 서버 측의 포인터라고 생각할 수 있다. 왜냐하면 커서는 private SQL area와 밀접하게 연관되어있으며, 용어가 가끔 교차사용되기 때문이다.

![[cncpt324.gif]]

private SQL area는 다음의 공간으로 나눠져 있다:
- run-time area
이 영역은 쿼리 수행 상태 정보를 담고 있다. 예를 들어 run-time area는 풀 테이블 스캔에서 반환된 row의 수를 추적한다.

- persistance area
이 영역은 바인드 변수를 담고 있다. 바인드 변수는 질의가 수행될 때 질의문에게 공급된다. persistent area는 cursor가 닫혔을 때 해제된다.

클라이언트 프로세스는 private SQL area를 관리하는 책임이 있다. Private SQL area의 할당과 반환은 어플리케이션에 크게 의존한다.

대부분의 유저가 자동 커서 핸들링에 의존함에도 불구하고 오라클 데이터베이스 programmatic interfaces는 개발자에게 커서에 대한 통제권을 많이 제공해주고 있다. 일반적으로, 어플리케이션은 다시 사용되지 않을 커서를 닫아야 한다. 

### SQL Work Area
Work area는 메모리 집약적인 작업에 사용되는 PGA 메모리의 private 할당이다.

예를 들어, 정렬 작업이 정렬 공간을  사용한다. 비슷하게 해시 조인 작업이 해시 테이블을 만들기 위해 해시 공간을 사용한다. 비트맵 머지 또한 비트맵 머지 공간을 사용한다.

만약 작업에 의해 처리된 데이터의 양이 work area에 맞지 않으면, 오라클 데이터베이스는 데이터를 작은 조각으로 넣는다. 이 방법으로, 데이터베이스는 메모리에 올라가지 않은 데이터는 /tmp에 저장한다.

Automatic PGA memory management가 활성화 됐을 때, 데이터베이스는 자동적으로 작업 영역의 크기를 튜닝한다. 

일반적으로 큰 work area를 가지면 성능을 향상시킬 수 있다. 최적으로는, work area의 크기는 관련 SQL 연산자가 할당한 입력 데이터 및 보조 메모리 구조를 수용하기에 충분하다. 만약 그렇지 않으면, 인풋 데이터의 일부를 디스크에 캐싱해야되기 때문에 응답 시간이 늘어난다. 극단적인 사례에선, 만약 work area가 너무 작으면, 데이터베이스는 데이터 조각을 여러번 디스크에 옮겨야 하고, 응답 시간이 기하급수적으로 늘어난다.


### Shared server 와 Dedicated Server 에서의 PGA 사용
| Memory Area                    | Dedicated Server | Shared Server |
| ------------------------------ | ---------------- | ------------- |
| 세션 메모리의 특성             | Private          | Shared        |
| persistent area의 위치         | PGA              | SGA           |
| DML과 DDL의 run-time area 위치 | PGA              | PGA              |


# SGA
## Database Buffer Cache
Database Buffer Cache는 데이터 파일로부터 읽어온 데이터 블록의 복사본이 저장된다.

동시적으로 데이터베이스 인스턴스에 연결된 모든 유저는 버퍼 캐시에 접근할 수 있다.

### 버퍼 캐시의 목적
- 물리적 I/O의 최적화
데이터베이스는 캐시의 데이터 블록을 변경시키고, 변경에 대한 메타데이터를 리두 로그 버퍼에 저장한다. commit 이후, 데이터베이스는 데이터 블록을 데이터 파일에 바로 쓰는 것이 아니라 리두 버퍼에서 online redo log에 쓰게 된다. 대신 database writer는 백그라운드에서 lazy write를 한다.

- 자주 액세스하는 블을 버퍼 캐시에 보관하고 자주 액세스하지 않는 블록을 디스크에 기록
Database Smart Flash Cache가 활성화 될 때, 버퍼 캐시의 일부가 flash cache에 있을 수 있다. 이 버퍼 캐시 확장은 1개 이상의 플래시 메모리를 사용하는 SSD인  flash disk 기기에 저장되어 있다. 데이터베이스는 HDD 를 사용하기 보다 버퍼 캐시를 넣은 SSD를 사용하여 성능 향상을 꾀할 수 있다.


### 버퍼 상태
- Unused
사용할 수 있는 버퍼
- Clean
이 버퍼는 이전에 사용되었으며 이제 특정 시점의 읽기 일관성 있는 블록 버전을 포함한다. 데이터를 포함하고 있지만 checkpoint될 필요는 없다.
- Dirty
디스크에 기록되지 않은 변경된 데이터를 저장하고 있다. 데이터베이스는 block을 체크포인트 해야한다.

모든 버퍼는 access mode가 있다
- pinned
버퍼가 pinned 된 것은 유저 세션이 접근하고 있는 동안 age out 되지 않는 것을 말함
- free


### 버퍼 모드
- Current mode
Current mode get은 db block get으로 불리고, 버퍼 캐시에서 데이터를 검색한 것을 말한다. 예를 들어 커밋 되지 않은 트랜잭션이 블록의 두 row를 업데이트 했다면, current mode get은 이 uncommitted row를 검색한다. 데이터베이스는 db block get을 변경 질의에서 가장 많이 사용하는데, 현재 버전의 블록만 변경해야 하기 때문이다.

- Consistent mode
consistent read get은 블록의 읽기 일관성 버전 검색이다. 이 검색은 undo data를 사용한다.

### 버퍼 I/O
논리적 I/O는 버퍼 I/O로 알려져 있는데, 버퍼 캐시의 버퍼에 쓰기나 읽기 작업을 하는 것을 말한다.

#### 버퍼 교체 알고리즘
버퍼에 대한 접근을 효율적으로 하기 위해, 데이터베이스는 어떤 데이터를 메모리에 캐시하고 어떤 데이터를 디스크로부터 가져올지 결정해야 한다.

데이터베이스는 다음과 같은 알고리즘을 사용한다 :
- LRU-based, block-level replacement algorithm
개념적으로 하나의 LRU만 사용하지만 데이터 동시성을 위해 데이터베이스는 많은 LRU들을 활용한다.

- Temperature-based, object-level replacement algorithm
오라클 12c release 1에서 시작된 Automatic big table caching은 다음 시나리오에 맞게 다른 알고리즘을 사용하여 스캔한다
	- Parallel queries
	싱글 인스턴스와 RAC에서는 parallel 쿼리는 big table cache를 사용할 수 있다.
	- Serial queries
	싱글 인스턴스에서만 serial 쿼리는 big table cache를 사용할 수 있다.

테이블이 메모리에 맞지 않는다면, 데이터베이스는 어떤 버퍼를 access pattern에 기초하여 캐싱할지 결정한다. 예를 들어, 만약 테이블의 95%가 메모리에 들어간다면, 메모리에서 블록을 읽고 디스크로 쓰는 것을 반복하기 보다 데이터베이스는 5%를 디스크에 남겨놓을 것이다. 이 현상을thrashing이라고 한다. 다수의 큰 오브젝트를 캐싱할 때 데이터베이스는 더 인기있는 테이블을 뜨겁다고 생각하고 반대를 차갑다고 생각하고, 이는 블록의 캐싱에 영향을 끼친다.
### 버퍼 쓰기
database writer 프로세스가 cold, dirty 버퍼를 디스크로 쓴다

- 서버 프로세스가 clean buffer를 찾지 못할 때
- 데이터베이스가 checkpoint를 해야 할 때
- 테이블스페이스가 read-only상태로 바뀌거나 offline 될 때

### 버퍼 읽기
사용되지 않은 버퍼의 숫자가 적으면, 데이터베이스는 버퍼를 버퍼 캐시로부터 제거해야한다.

- Flash cache가 비활성화 됐을 때
데이터베이스는 각각의 clean buffer를 필요할 때 재사용하고, 덮어쓴다. 만약 덮어씌워진 버퍼가 나중에 필요하게 된다면 데이터베이스는 디스크로부터 다시 읽어와야 한다.
- Flash cache 활성화 됐을 때
DBW는 clean buffer의 body를 flash cache로 씀으로써 메모리 버퍼를 재사용  할 수 있다. 데이터베이스는 flash cache의 버퍼 body의 위치와 상태를 추적하기 위해 버퍼 헤더를 LRU list에 저장한다.

1. 서버 프로세스가 버퍼 캐시에서 모든 버퍼를 탐색한다
2. 서버 프로세스가 flash cache LRU list의 버퍼 헤더를 탐색한다
3. 만약 프로세스가 버퍼를 메모리에서 찾지 못하면 다음과 같은 단계를 밟는다
	1. 디스크의 데이터 파일로부터 블록을 복사해 온다(physical read)
	2. 버퍼에서 logical read를 수행한다
![[cncpt304.gif]]

Buffer cache hit ratio는 데이터베이스가 디스크를 사용하지 않고 버퍼 캐시에서 데이터를 가져온 빈도 수를 측정한다.

데이터베이스는 데이터 파일 뿐만 아니라 임시 파일로부터 데이터를 읽을 수 있다. 이는 작업 중 메모리가 부족해 강제적으로 임시 테이블에 쓰고 돌려받는 것으로, 버퍼 캐시를 거치지 않기 때문에 논리적 I/O가 발생하지 않는다.

#### Buffer touch count
데이터베이스는 LRU list에 접근한 버퍼의 접근 빈도수를 touch count를 통해 측정한다. 이 매카니즘은 LRU list를 셔플하지 않고 pinned된 버퍼의 카운터를 증가시킬 수 있도록 한다.

> 데이터베이스는 물리적으로 메모리를 움직이지 않고, 리스트의 포인터만 움직인다

카운터는 3초가 지나고 버퍼가 pinned 되어야 증가한다. 이 3초 룰은 많은 접근에 의해 버퍼 카운팅이 폭발적으로 증가하는 것을 막기 위함이다.

## 버퍼 풀
버퍼 풀은 버퍼의 집합이다.
데이터베이스의 버퍼 캐시는 1개 이상의 버퍼 풀로 나뉠 수 있다. 버퍼 풀은 버퍼 캐시와 비슷한 알고리즘으로 다뤄진다.

데이터를 버퍼 캐시에 유지하거나 데이터 블록을 사용한 직후에 버퍼 새 데이터에 사용할 수 있도록 하는 별도의 버퍼 풀을 수동으로 구성할 수 있다.  캐시로부터 블록이 age out 되는 방법을 컨트롤 하기 위해 스키마 오브젝트를 적절한 버퍼 풀에 할당할 수도 있다.

가능한 버퍼 풀은 다음과 같다 :
- Default Pool : 블록이 일반적으로 캐시 되는 곳
- Keep Pool : 이 풀은 자주 액세스했지만 공간이 부족하여 기본 풀이 만료된 블록을 대상으로 한다.
- Recycle Pool : 이 풀은 자주 사용되지 않은 블록을 대상으로 한다. 불필요한 공간을 차지하는 오브젝트를 대상으로 한다.

![[cncpt220 2.gif]]


### 버퍼와 풀 테이블 스캔
데이터베이스는 복잡한 알고리즘을 사용하여 테이블 스캔을 관리한다. 기본적으로 버퍼가 디스크로부터 읽힌다면 데이터베이스는 버퍼를 LRU list의 중간에 넣어 hot block이 캐시에 남아있도록 한다. 이로서 디스크를 다시 읽지 않아도 되게 한다.

Full table scan의 경우 순서적으로 table high water mark 아래에서 모든 row를 읽는다.

#### 풀 테이블 스캔의 Default Mode
기본적으로, 데이터베이스는 풀 테이블 스캔에 대한 보수적인 접근을 하는데, 테이블 사이즈가 작을 경우에만 작은 테이블을 메모리에 로드한다.

중간 크기의 테이블을 캐시해야 하는지 여부를 결정하기 위해 데이터베이스는 마지막 테이블 검색, 버퍼 캐시의 오래된 타임스탬프 및 버퍼 캐시에 남아 있는 공간 사이의 간격을 통합하는 알고리즘을 사용한다.

매우 큰 테이블의 경우, 데이터베이스는 형식적으로 버퍼 캐시가 채워지지 않도록 direct path read를 통해 블록을 PGA에 직접 로드하고 SGA를 완전히 우회한다.

중간 크기의 테이블은 direct path read를 할 수도 있고 cache read를 할 수도 있는데, cache read를 할 경우에는 LRU list의 가장 마지막에 넣는다.

#### Prarllel Query Excution
Full table scan을 수행할 때, 데이터베이스는 multiple parallel excution server를 통해 응답 시간을 향상시킬 수 있다.

형식적으로, 병렬 쿼리는 리소스 사용량 때문에 데이터 웨어하우스같은 동시성이 적은 환경에서 사용된다.

#### CACHE Attribute
CACHE 속성을 적용시키면, 데이터베이스는 버퍼 캐시의 블록을 강제로 pin 하지 않고(LRU list의 중간에 넣는 등의 알고리즘을 하지 않고), 다른 테이블 블록과 같은 방법으로 캐싱한다.

#### KEEP 속성
버퍼 풀에 버퍼를 저장하는 대신, Keep 풀에 저장한다.

#### Force Full Database Caching Mode
NOCACHE LOBS를 포함해 모든 데이터베이스를 버퍼 캐시에 강제적으로 적재한다.

오라클은 force full database caching 모드를 각각의 인스턴스의 버퍼 캐시 사이즈가 데이터베이스 사이즈 보다 클 때 사용하기를 권장한다.

## Redo Log Buffer
리두 로그 버퍼는 변경을 표현하는 redo entry를 저장하는 SGA에 존재하는 링형 버퍼이다.

Redo record는 DML 이나 DDL에 의해 만들어진 변경을 재구성하는데 필요한 정보를 담고 있는 데이터 구조이다. 데이터베이스 복구는 redo entry를 잃어버린 변경을 재구성하기 위해 데이터파일에 적용시킨다.

데이터베이스 프로세스는 유저 메모리의 redo entry를 SGA의 redo log buffer로 복사한다. Redo entry는 연속적인 버퍼 공간이. 백그라운드 프로세스인 log writer process는 redo log buffer를 디스크의 redo log group에 쓴다.

![Description of Figure 14-8 follows](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/img/cncpt226.gif "Description of Figure 14-8 follows")


LGWR은 DBW가 디스크에 데이터 블록 쓰기를 수행하는 동안 연속적인 쓰기를 수행한다.

## 공유 풀
Shared pool은 프로그램 데이터의 다양한 타입을 캐싱한다.

예를 들어, 공유 풀은 parsed SQL, PL/SQL code, system parameter, data dictionary 정보를 저장한다. 공유 풀은 데이터베이스에서 일어나는 거의 모든 작업을 포함하고 있다. ![Description of Figure 14-9 follows](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/img/cncpt225.gif "Description of Figure 14-9 follows")

### Library Cache
Library cache는 실행 가능한 SQL과 PL/SQL 코드를 저장하는 공유 풀 메모리 구조이다.

캐시는 공유 SQL과 PL/SQL 공간을 포함하고 libarary cache handle과 락 같은 구조를 컨트롤한다.

#### Shared SQL Areas
데이터베이스는 shared SQL area를 처음 발생하는 SQL 질의를 처리하기 위해 사용한다. 이 영역은 parsed SQL 질의와 실행 계획을 포함한다. 오직 하나의 shared SQL area가 유일한 질의를 위해 존재한다. 같은 SQL을 질의하는 각각의 유저의 private SQL area는 같은 shared SQL area를 가리킨다.

데이터베이스는 다음과 같은 단계를 밟는다:
1. shared SQL area가 동일한 질의문을 가지는지 확인하기 위해 shared pool을 확인한다.
	1. 만약 동일한 질의문이 존재한다면, 데이터베이스는 실행을 위해 shared SQL area를 사용한다
	2. 만약 동일한 질의문이 존재하지 않는다면, 데이터베이스는 새로운 shared SQL area를 할당한다.
2. 세션 대신 private SQL area를 할당한다
![Description of Figure 14-10 follows](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/img/cncpt252.gif "Description of Figure 14-10 follows")

### Data Dictionary Cache
Data dictionary는 데이터베이스와 데이터베이스 구조와 유저에 대한 정보를 포함하는 테이블과 뷰의 집합이다.

오라클 데이터베이스는 데이터 딕셔너리를 SQL statement가 파싱될 때 접근한다.
오라클 데이터베이스는 지정된 장소에 저장된 딕셔너리 데이터에 접근한다.
- 데이터 딕셔너리 캐시
- Library 캐시

### Server Result Cache
server result cache는 shared pool 내부에 있는 메모리 풀이다. 버퍼 풀과는 다르게 server result cache는 데이터 블록이 아닌 result set을 가지고 있다.

### Reserved Pool
Reserved pool은 오라클 데이터베이스가 연속적이고 큰 메모리 청크를 사용할 수 있는 공유 풀 안에 있는 메모리 공간이다.

데이터베이스는 공유 풀을 청크 단위로 할당받는다. Chunking은 5KB가 넘는 큰 오브젝트가 하나의 연속적인 공간을 요구하지 않고 적재될 수 있도록 한다. 이 방법으로 데이터베이스는 파편화로 인한 연속적인 메모리 공간의 부족 가능성을 줄인다.

## Large Pool
Large pool은 optional 메모리 공간으로 shared pool에 적절한 크기보다 큰 메모리 할당을 위한 것이다.

Large pool은 다음에게 큰 메모리 할당을 제공한다 :
- shared server의 UGA
- 병렬 실행에서의 메시지 버퍼
- Recovery Manager I/O slave의 버퍼
- deferred insert 버퍼

![Description of Figure 14-11 follows](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/img/cncpt221.png "Description of Figure 14-11 follows")

### Large Pool Memory Management
Large Pool은 메모리의 일부가 age out 될 수 있도록 LRU list를 사용하는 shared pool과는 다르게 관리한다.

Large Pool은 LRU list를 가지지 않는다. 데이터베이스가 large pool 메모리를 데이터베이스 세션에 할당하면, 이 메모리는 세션에서 해제하지 않으면 해제할 수 없다. 메모리의 일부가 해제되면, 다른 프로세스가 사용할 수 있다. 세션 메모리를 large pool에 할당함으로서 데이터베이스는 shared pool에서 일어날 수 있는 파편화를 막을 수 있다.

### Large Pool Buffer for deferred insert
deferred insert를 위해 데이터베이스는 large pool에 버퍼를 할당한다.

## Java Pool
Java pool은 모든 세션 특정된 자바 코드와 JVM의 데이터를 저장하는 공간이다.

Dedicated server 연결에서는, Java pool은 각각의 자바 클래스의 공유된 부분을 포함하고 코드 벡터나 메소드 등을 포함한다. 하지만 각각의 세션의 자바 상태는 저장하지 않는다. Shared server에서는 pool은 각각의 클래스의 공유 부분을 포함하고 세션 상태 저장에 사용되는 몇몇 UGA도 저장한다.

## Fixed SGA
Fixed SGA는 내부 하우스키핑 영역이다.
fixed SGA는 다음을 포함한다 :
- 백그라운드 프로세스가 접근해야할 데이터베이스와 인스턴스의 일반적인 정보.
- Lock 같은 프로세스 간에 알고 있어야하는 정보.
fixed SGA는 오라클 데이터베이스에 의해 결정되고, 변경할 수 없다. Release에 따라 이 크기는 달라진다.

