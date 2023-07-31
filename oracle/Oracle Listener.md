
# Oracle Net Service 소개

[Oracle Net Services DOC](https://docs.oracle.com/en/database/oracle/oracle-database/19/netag/introducing-oracle-net-services.html#GUID-7A30E150-8CB3-44BA-8966-FC8F28C1638B)
Oracle Net Services는 분산되고 이기종인 컴퓨팅 환경끼리의 연결 솔루션이다.

## Connectivity
Oracle Net Service의 컴포넌트인 Oracle Net은 클라이언트와 데이터베이스간의 네트워크 세션 생성과 유지를 책임지고, 둘 사이의 데이터 교환을 책임진다. Oracle Net이 이런 일을 할 수 있는 이유는 통신 주체의 컴퓨터에 Oracle Net이 설치되어 있기 때문이다.

### Client/Server Application Connection
Oracle Net은 전통적인 client/server application에서 Oracle Database server간의 연결을 가능하게 한다. 단, Oracle Net은 각각의 기기에 설치되어 있어야 연결이 가능하다. Oracle Net은 TCP/IP로 통신한다. Oracle Net은 연결을 생성하고 유지하는 Oracle Net foundation layer와 foundation layer의 기술과 산업 표준 프로토콜을 매핑하는  Oracle protocol support로 구성되어 있다.

## Oracle Net Listener
Oracle Database는 Oracle Net Listener로부터 첫 연결을 받는다. Listener는 클라이언트의 요청을 서버에게 전달해준다. 리스너는 프로토콜 주소로 구성되어 있고, 같은 프로토콜 주소로 구성된 클라이언트가 리스너에게 연결 요청을 보낼 수 있다. 연결이 생성되면, 클라이언트와 오라클 서버는 직접적으로 통신할 수 있다.

# Database 식별과 접속
## 인스턴스와 서비스
### 인스턴스
데이터베이스는 최소 1개의 인스턴스를 가진다. 인스턴스는 SGA 메모리 공간과 백그라운드 프로세스로 구성된다. 인스턴스는 효과적으로 데이터베이스의 데이터를 가져와 클라이언트에게 제공한다. 

인스턴스 이름은 INSTANCE_NAME이라는 초기화 파라미터로 결정되지만, 기본적으로는 데이터베이스 인스턴스의 SID 값으로 결정된다.

### 서비스
클라이언트는 오라클 데이터베이스를 서비스 이름으로 인식한다. 데이터베이스는 1개 이상의 서비스 이름을 가질 수 있다.

인스턴스가 가동되면 하나 이상의 서비스 이름을 사용하여 LISTENER에 등록한다. 클라이언트나 데이터베이스가 리스너에 연결할 때 서비스에게 연결을 요청하게 된다.

서비스 이름은 다수의 인스턴스를 식별할 수 있고, 인스턴스는 다수의 서비스에 속할 수 있다. 그렇기 때문에 리스너는 클라이언트와 인스턴스 사이에서 중재자 역할을 하고, 적절한 인스턴스에 요청을 라우팅하는 역할을 한다. 서비스에 연결을 요청하는 클라이언트는 어떤 인스턴스에 연결할지 특정하지 않아도 된다.

서비스 이름은 서버 파라미터 파일 내부의 SERVICE_NAMES라는 초기화 파라미터를 통해 초기화 된다. 서버 파라미터 파일은 ALTER SYSTEM 명령어를 통해 초기화 파라미터를 바꿀 수 있게 하고, SHUTDOWN과 STARTUP에 의해 변경 사항을 적용시킨다. 서비스 이름은 global database name에 의해 설정되고 DB_NAME과 DB_DOMAIN의 조합으로 생겨난다.

19C 부터 SERVICE_NAME 파라미터는 deprecated 됐기 때문에, SRVCTL이나 GDSCTL 명령어를 사용하거나 DBMS_SERVICE 패키지를 사용해야 한다.

하나의 데이터베이스에 다수의 서비스를 할당하는 것의 이점 :
- 단일 데이터베이스가 각기 다른 방법으로 식별 될 수 있다.
- 데이터베이스 관리자가 시스템 자원을 제한하거나 예약할 수 있다. 

