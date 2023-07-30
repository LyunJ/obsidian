# 오라클 티베로 파라미터 비교

| 오라클 | 티베로 | 설명 |
| -- | -- | -- |
| db_name | db_name | 데이터베이스 이름 |
| db_unique_name | -- | 같은 db_name을 가지더라도 db_unique_name은 달라야 한다 |
| memory_target | memory_target | 총 메모리 크기 |
| processes | MAX_SESSION_COUNT | 오라클 : 유저 프로세스 개수, 티베로 : 워커 스레드 개수 |
| audit_file_dest | audit_file_dest | 감사 파일 저장 디렉토리 설정 |
| audit_trail | audit_trail | os : os 파일 시스템에 저장, db : 테이블에 저장 |
| db_block_size | db_block_size | 블록 사이즈 |
| db_domain | -- | db 링크에 사용되는 도메인 url |
| db_recovery_file_dest | db_recovery_file_dest | fast recovery area에 있는 복구 파일 위치 |
| db_recovery_file_dest_size | db_recovery_file_dest_size | fast recovery area에 있는 복구 디렉토리 크기 |
| diagnostic_dest | -- | 인스턴스 진단 파일 위치 |
| dispatchers | -- | shared server 아키텍쳐에서 dispatcher process 구성 설정 |
| open_cursors | open_cursors | 한 세션이 가질 수 있는 cursor(독립적인 SQL 공간)의 개수 |
| remote_login_passwordfile | -- | shared : 1개 이상의 데이터베이스가 password file을 사용할 수 있다. exclusive : 1개의 데이터베이스만 password file을 사용할 수 있다. none : password file을 무시한다 |
| undo_tablespace | undo_tablespace | 인스턴스가 시작할 때 사용할 undo tablespace를 지정한다 |
| control_files | control_files | 컨트롤 파일 위치 |
| compatible | -- | 데이터베이스 버전 변경(10g 부터는 버전 역행 불가) |
| -- | total_shm_size | 공유메모리 크기 | 
