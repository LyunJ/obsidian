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
PMON group은 PMON과 Cleanup Main Process (CLMN), Cleanup Helper Processes(CLnn)을 포함한다. 이 프로세스들은 다른 프로세스의 모니터링과 삭제를 담당한다.

