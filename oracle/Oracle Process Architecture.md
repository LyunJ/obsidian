## Introduction to Processes
프로세스 실행 아키텍쳐는 OS에 의존적이다. 
예를 들어 윈도우에서 오라클 백그라운드 프로세스는 프로세스 내부의 스레드 실행이다. 
리눅스나 유닉스에서는 오라클 프로세스는 운영체제 내의 프로세스 또는 프로세스 내의 스레드이다. 

### 프로세스 타입
- 클라이언트 프로세스 : 어플리케이션이나 오라클 도구 코드를 실행시킨다
- 오라클 프로세스 : 오라클 데이터베이스 코드를 실행시키는 유닛. 멀티 스레드 아키텍쳐에서는 오라클 프로세스는 OS의 프로세스일 수 있고 스레드일 수 있다
	- 백그라운드 프로세스 : 데이터베이스 인스턴스를 시작하고 인스턴스 복구, redo buffer 쓰기 같은 유지보수 업무를 수행한다.
	- 서버 프로세스 : 클라이언트 요청에 따른 작업을 한다.
		- SQL 쿼리 파싱, shared pool에 적재, 쿼리 플랜 생성 등등..

![Description of Figure 15-1 follows](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/img/cncpt283.gif "Description of Figure 15-1 follows")

### Client Process 개요
#### 클라이언트와 서버 프로세스
클라이언트 프로세스는 인스턴스와 직접적으로 상호작용할 수 없고, SGA에 직접 읽고 쓰기를 할 수 없다.

#### Connection과 Session
데이터베이스 커넥션은 클라이언트 프로세스와 데이터베이스 인스턴스 간의 물리적 통신 경로다.

형식적으로 커넥션은 클라이언트 프로세스와 서버프로세스 또는 디스패쳐 사이에서 발생하는데, Oracle Connection Manager와 클라이언트 프로세스 사이에서도 발생한다.

데이터베이스 세션은 현재 유저가 데이터베이스에 로그인한 상태를 대표하는 데이터베이스 인스턴스 메모리 내의 논리적인 엔티티다.

![Description of Figure 15-2 follows](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/img/cncpt237.gif "Description of Figure 15-2 follows")

![Description of Figure 15-3 follows](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/img/cncpt238.gif "Description of Figure 15-3 follows")

## Server Process 개요
오라클 데이터베이스는 인스턴스에 연결되어 있는 클라이언트 프로세스의 요청을 처리하기 위해 서버 프로세스를 생성한다. 클라이언트 프로세스는 언제나 서버 프로세스를 통해 데이터베이스와 통신한다.

서버 프로세스는 다음과 같은 작업을 한다 :
- 쿼리 플랜을 생성하고 실행시키고, SQL 질의를 파싱하고 실행시킨다
- PL/SQL 코드를 실행시킨다
- 데이터 파일로 부터 읽은 데이터 블록을 버퍼 캐시에 저장

### Dedicated Server Process
Dedicated server connection에서는, 클라이언트 커넥션은 하나의 서버 프로세스와 하나만 연결된다.

서버 프로세스는 프로세스 정보를 PGA 내부의 UGA에 저장한다.

### Shared Server Process
Shared server connection에서는, 클라이언트 어플리케이션이 dispatcher process를 통해 연결하게 된다. 

Dedicated server process와 같이, shared server process도 PGA를 가진다. 하지만 세션을 위한 UGA는 SGA에 저장되어 어떤 shared server도 세션 데이터에 접근할 수 있게 된다.

### How Oracle Database Creates Server Processes
데이터베이스는 연결 방법에 따라 다양한 방법으로 서버 프로세스를 생성한다.

연결 방법은 다음과 같다 :
- Bequeath : SQL Plus, OCI Client나 다른 클라이언트 어플리케이션이 서버 프로세스를 직접 생성한다.
- Oracle Net Listener : 리스너를 통해 데이터베이스에 연결한다.
- Dedicated Broker : Foreground 프로세스를 생성하는 데이터베이스 프로세스이다. 리스너와는 다르게 브로커는 데이터베이스 인스턴스 내부에 있다. dedicated broker를 사용하면, 클라이언트는 리스너에 연결한 다음 dedicated broker에 대한 연결을 끊는다.

Bequeath를 사용하지 않는 연결일 경우 데이터베이스는 다음과 같은 방법으로 서버 프로세스를 생성한다 :
1. 클라이언트 어플리케이션이 리스너나 브로커에게 새로운 연결을 요청한다
2. 리스너나 브로커가 새로운 프로세스나 스레드의 생성을 초기화한다.
3. 운영체제는 새로운 프로세스나 스레드를 생성한다
4. 오라클 데이터베이스는 다양한 컴포넌트나 알림을 초기화한다.
5. 데이터베이스가 커넥션 및 커넥션별 코드를 전달한다.

## 백그라운드 프로세스 개요
백그라운드 프로세스는 멀티 프로세스 오라클 데이터베이스에서 사용되는 추가적인 프로세스이다. 백그라운드 프로세스는 데이터베이스를 운영하기 위해 필요한 작업이나 다수의 사용자를 상대로 성능을 고도화 시키기 위한 작업을 유지 관리 하기 위해 사용된다.

각각의 프로세스는 분리된 작업을 하지만 다른 프로세스와 함께 협업한다. 예를 들어 LGWR 프로세스는 리두 로그 버퍼에서 온라인 리두 로그로 데이터를 쓴다. 꽉 찬 리두 로그 파일이 아카이브 될 준비가 됐다면, LGWR은 리두 로그 파일을 아카이브 하기 위해 다른 프로세스에게 신호를 보낸다.

오라클 데이터베이스는 데이터베이스 인스턴스가 시작될 때 자동적으로 백그라운드 프로세스를 만든다. 

### 필수적인 백그라운드 프로세스
- Process Monitor Process(PMON) group
- Process Manager (PMAN)
- Listener Registration Process (LREG)
- System Monitor Process (SMON)
- Database Writer Process (DBW)
- Log Writer Process (LGWR)
- Checkpoint Process(CKPT)
- Manageability Monitor Processes (MMON and MMNL)
- Recoverer Process (RECO)

#### Process Monitor Process(PMON) Group
PMON group은 PMON과 Cleanup Main Process (CLMN), Cleanup Helper Processes(CLnn)을 포함한다. 이 프로세스들은 다른 프로세스의 모니터링과 프로세스 정리를 담당한다.

PMON 그룹은 버퍼 캐시의 정리 및 클라이언트 프로세스에서 사용되는 리소스의 해제를 감독한다.예를 들어 PMON 그룹은 액티브 트랜잭션 테이블 상태의 재세팅, 더이상 필요 없는 락의 해제, 종료된 프로세스 ID의 제거를 담당한다.

##### Process Monitor Process (PMON)
PMON 백그라운드 프로세스의 종료를 감지한다. 만약 서버나 디스패처 프로세스가 비정상적으로 종료됐다면, PMON 그룹은 프로세스 복구를 실행시킬 책임이 있다.

프로세스 종료는 다양한 원인이 있다.
```
ALTER SYSTEM KILL SESSION
```

##### Cleanup Main Process(CLMN)
PMONdms Cleanup Main Process(CLMN)에 정리 작업을 위임한다. 비정상 종료를 감지하는 작업은 PMON에 남아 있다.

CLMN은 주기적으로 종료된 프로세스, 종료된 세션, 트랜잭션, 네트워크 커넥션, 쉬는 상태의 세션, 분리된 트랜잭션, 분리된 네트워크 커넥션의 정리를 담당한다.

##### Cleanup Helper Processes(CLnn)
CLMN은 정리 작업을 CLnn helper process에 위임한다.

CLnn 프로세스는 종료된 프로세스와 세션의 정리를 보조한다. Helper 프로세스의 수는 수행할 정리 작업의 양과 정리의 현재 효율에 비례한다.

Cleanup 프로세스가 차단되어 다른 프로세스를 정리하지 못할 수 있다. 또한 만약 다수의 프로세스가 정리를 요구하면, 정리 시간은 중요해진다. 그렇기 때문에, 오라클 데이터베이스는 다수의 helper 프로세스를 병렬로 사용하여 정리를 수행할 수 있으므로 성능 저하를 줄일 수 있다.

The `V$CLEANUP_PROCESS` and `V$DEAD_CLEANUP` views contain metadata about CLMN cleanup

##### Database Resource Quarantine
만약 프로세스나 세션이 종료되면, PMON 그룹은 가지고 있던 리소스를 데이터베이스에 돌려준다. 몇몇 상황에서는 PMON 그룹은 자동적으로 quarantine 감염이 될 수 있는데, 복구 불가능한 자원이기에 데이터베이스 인스턴스는 즉시 종료되지 않는다.

PMON 그룹은 감염된 리소스를 가지고 있는 프로세스나 세션을 가능한 계속적으로 정리를 수행한다. The `V$QUARANTINE` view contains metadata such as the type of resource, amount of memory consumed, Oracle error causing the quarantine, and so on.


#### Process Manager(PMAN)
Process Manager(PMAN)은 shared server, pooled server, job queue process 같은 백그라운드 프로세스에 대한 감독을 수행한다.

PMAN은 다음과 같은 프로세스를 모니터하고, 생성하고, 종료한다 :
- Dispatcher와 shared server process
- DRCP를 위한 Connection broker와 pooled server process
- Job queue processes
- 재시작 가능한 백그라운드 프로세스

#### Listener Registration Process (LREG)
Listener registration process(LREG)는 Oracle Net Listener에 데이터베이스 인스턴스와 dispatcher process에 대한 정보를 등록한다.

인스턴스가 시작될 때, LREG는 리스너를 polling 하여 실행중인지 여부를 확인한다. 만약 리스너가 실행중이면, LREG는 관련 파라미터를 제공한다. 만약 실행중이 아니면, LREG는 주기적으로 접근하려고 한다.

#### System Monitor Process (SMON)
System monitor process(SMON)은 다양한 시스템 수준의 정리 의무를 가진다.

임무는 다음과 같다 :
- 인스턴스 복구. 필요하다면, 인스턴스 시작 시 인스턴스 복구 수행. 오라클 RAC에서는, 하나의 데이터베이스 인스턴스의 SMON 프로세스는 실패한 인스턴스의 복구를 수행할 수 있다.
- 파일 읽기나 테이블 스페이스 오프라인 에러 때문에 인스턴스 복구 중 생략되어 종료된 트랜잭션의 복구. SMON은 테이블 스페이스나 파일이 온라인 상태로 돌아왔을 때 트랜잭션을 복구한다.
- 사용하지 않은 임시 세그먼트 정리. 예를 들어, 오라클 데이터베이스는 인덱스를 생성할 때 extent를 할당한다. 만약 작업이 실패할 경우, SMON은 임시 공간을 정리한다.
- Dictionary-managed 테이블 스페이스 내부의 연속된 사용 가능한 extent 병합.

#### Database Writer Process(DBW)
Database writer process(DBW)는 데이터베이스 버퍼의 내용을 데이터 파일로 쓰는 작업을 한다. DBW 프로세스는 변경된 버퍼를 디스크로 쓴다.

대부분의 시스템에서는 DBW가 하나 있어도 충분하지만, 추가적인 DBW를 추가하여 성능을 향상 시킬 수 있다. 유니 프로세스 프로그램에서는 적절하지 않다

DBW 프로세스는 다음과 같은 조건에서 더티 버퍼를 디스크로 쓴다 :
- 서버 프로세스가 임계값 숫자만큼 스캔을 수행하고 재사용 가능한 빈 버퍼를 찾지 못하면,  DBW에게 쓰기 신호를 보낸다. DBW는 더티 버퍼를 디스크로 비동기적으로 쓴다.
- DBW가 주기적으로 checkpoint를 진행시키기 위해 버퍼 쓰기 작업을 한다. Checkpoint는 인스턴스 복구가 시작되는 redo 스레드의 위치이다.

##### Log Writer Process (LGWR)
Log writer process(LGWR)은 online redo log buffer를 관리한다.

LGWR은 버퍼의 일부를 online redo log에 기록한다. 데이터베이스 버퍼 변경 작업을 분리하고 디스크에 더티 버퍼를 분산 쓰기 작업을 수행하며 디스크에 redo를 빠르게 순차적으로 쓰기 작업을 수행함으로서 데이터베이스의 성능이 향상된다.

다음과 같은 상황에서 LGWR은 마지막으로 쓰여진 시점으로부터 버퍼로 복사된 모든 redo entry를 쓴다 :
- 트랜잭션에서 유저 커밋
- online redo log switch 발생
- LGWR이 쓰기 작업을 한 뒤로부터 3초 지남
- redo log buffer가 1/3이 차있거나 1MB의 버퍼 데이터가 포함될 때.
- DBW는 반드시 변경된 버퍼를 디스크로 써야한다.

###### LGWR and Commits
오라클 데이터베이스는 커밋된 트랜잭션을 위해 fast commit 매커니즘을 사용한다.

유저가 commit 질의를 요청하면, 트랜잭션은 SCN을 할당받는다. LGWR은 redo log buffer에 커밋 레코드를 넣고 커밋 SCN 및 트랜잭션의 redo 항목과 함께 즉시 디스크에 기록한다.

Redo log buffer는 원형이다. LGWR이 redo entry를 redo log buffer에서 online redo log file로 쓸 때, 서버 프로세스는 디스크로 쓴 공간에 새로운 entry를 쓸 수 있다. LGWR은 일반적으로 online redo log에 접근하는 것이 무거운 작업임에도 버퍼에 새로운 공간을 만들기 위해 빠르게 쓰려고 한다.

오라클 데이터베이스는 데이터가 디스크에 쓰여지지 않았음에도 불구하고 트랜잭션 커밋 성공 코드를 반환한다. DBW가 데이터 파일에 데이터 블록을 쓰는 것이 효율적일 때까지 데이터 블록에 대한 해당 변경 사항은 연기된다.

연결이 많아지면, LGWR은 그룹 커밋을 통해 redo log file의 lock에 의한 성능 저하를 줄일 수 있다. 커밋 요청이 많아지면, LGWR은 다수의 commit record를 보유할 수 있다.


###### LGWR and Inaccessible Files
LGWR은 동기적으로 active mirrored group의 online redo log file에게 쓴다.

만약 로그 파일이 접근할 수 없으면 LGWR은 그룹의 다른 파일에 쓰기를 지속하고, LGWR 트레이스 파일에 에러를 기록하고 alert log를 기록한다. 만약 그룹의 모든 파일이 데미지를 입었거나 그룹이 아카이브 되지 않아서 사용 불가능해졌다면 LGWR은 기능을 할 수 없다.

##### Checkpoint Process(CKPT)
Checkpoint process(CKPT)는 컨트롤 파일과 체크포인트 정보를 포함한 데이터 파일 헤드를 업데이트하고 DBW에게 디스크에 블록을 쓰도록 신호를 보낸다. 체크포인트 정보는 체크포인트 position, SCN, online redo log의 복구 위치를 포함한다.

![[cncpt228 1.gif]]

##### Manageability Monitor Processes (MMON and MMNL)
Manageability monitor process (MMON)은 Automatic Workload Repository(AWR)에 연관된 많은 작업을 수행한다.

예를 들어, 메트릭이 임계값을 위반하고 스냅샷을 생성하며 최근 수정된 SQL 개체에 대한 통계 값을 캡처할 때 MMON이 기록한다.

Manageability monitor lite process(MMNL)은 SGA의 Active Session History(ASH)로부터 디스크에 통계를 기록한다. MML은 ASH 버퍼가 가득 찼을 때 디스크에 기록한다.

##### Recoverer Process(RECO)
분산 데이터베이스에서는 recoverer process(RECO)가 분산 트랜잭션의 장애를 자동적으로 해결한다.

노드의 RECO 프로세스는 의심스러운 분산 트랜잭션과 관련된 다른 데이터베이스에 자동으로 연결된 다른 데이터베이스에 자동으로 연결된 RECO가 데이터베이스 사이에서 커넥션을 재생성할 때, 자동적으로 모든 의심스로운 트랜잭션을 해결하여 각 데이터베이스의 보류 트랜잭션 테이블에서 해결된 트랜잭션에 해당하는 행을 제거한다.

