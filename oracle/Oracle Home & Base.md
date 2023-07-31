Oracle Home과 Base는 Oracle 설치 시 반드시 설정해야 하는 경로다. 환경 변수 ORACLE_HOME과 ORACLE_BASE로 미리 설정할 수 있고, ORACLE_HOME은 ORACLE_BASE의 자식 디렉토리여야 한다. 

ORACLE은 보안 문제 때문에, 다른 유저가 생성한 ORACLE_BASE에 ORACLE_HOME을 넣을 수 없다. 즉, ORACLE_BASE에 설치된 ORACLE 소프트웨어들은 한 명의 관리자만 관리가 가능하다. 

ORACLE_HOME은 오라클 소프트웨어 1개가 설치된 디렉토리다. ORACLE_BASE 아래에는 다수의 ORACLE 소프트웨어를 설치할 수 있다. 그렇기 때문에 ORACLE_HOME을 변경하며 ORACLE 소프트웨어를 추가로 설치할 수 있다.

