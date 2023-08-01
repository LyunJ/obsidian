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

PGA에 비유되는 것은 파일 점원이 사용하는 임시 카운터톱 작업 공간이다. 이 비유에서 파일 점원은 클라이언트를 대신하여 작업을 수행하는 서버 프로세스이다.