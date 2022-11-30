MySQL 서버는 어떤 작업이 수행되고 있는지 확인하는데 도움이 되는 여러 로그가 있다.

| Log Type | Information Written to Log |
| --- | --- |
| Error log | `mysqld`가 시작, 실행 중, 정지할 때 생긴 문제 기록 |
|  General query log | 클라이언트 커넥션 생성과 클라이언트로부터의 질의 기록 |
| Binary log | 데이터가 변경된 질의(복제 DB에서도 사용) |
| Relay log | 복제 원본 서버에서 받은 데이터 변경 기록 |
| Slow query log | `long_query_time` 을 초과한 쿼리 기록 |
| DDL log (metadata log) | DDL 질의로부터 수행된 메타데이터 작업 |

기본적으로, 윈도우에서 에러 로그를 기록하는 것을 제외하면 로깅은 비활성화 상태이다(DDL 로그는 항상 필요할 때 기록되며 유저가 설정할 수 있는 수단은 없다).  [The DDL Log](https://dev.mysql.com/doc/refman/5.7/en/ddl-log.html) 참고. 이 후의 섹션은 로깅을 활성화시키는 서버 옵션에 대한 정보를 제공한다.

기본적으로, 서버는 활성화된 로그를 데이터 디렉토리에 파일의 형태로 저장한다. 서버에게 로그 파일을 닫고 다시 열게 강제할 수 있다(혹은 몇몇 경우엔 새로운 로그 파일로 교체할 수 있다).  로그 플러싱은 `FLUSH LOGS`문을 실행시켰을 때 일어난다. 예를들어 `mysqladmin`을 `flush-logs`나 `refresh` 와 함께 실행시키거나 `mysqldump`를 `--flush-logs`나 `--master-data`옵션과 함께 실행시켰을 때 `FLUSH LOGS`문이 실행된다.  [Section 13.7.8.3, “FLUSH Statement”](https://dev.mysql.com/doc/refman/8.0/en/flush.html "13.7.8.3 FLUSH Statement"), [Section 4.5.2, “mysqladmin — A MySQL Server Administration Program”](https://dev.mysql.com/doc/refman/8.0/en/mysqladmin.html "4.5.2 mysqladmin — A MySQL Server Administration Program"), 그리고 [Section 4.5.4, “mysqldump — A Database Backup Program”](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html "4.5.4 mysqldump — A Database Backup Program") 참고. 추가로, 바이너리 로그는 로그의 크기가 `max binlog size` 시스템 변수의 값에 다다랐을 때 플러쉬 된다.

general query와 slow query 로그를 런타임에 통제할 수 있다. 로깅을 활성화/비활성화 시키거나 로그 파일의 이름을 변경시킬 수 있다. 서버에게 general query나 slow query 엔트리를 로그 테이블, 로그 파일 혹은 둘 다를 작성하도록 지시할 수 있다. [Section 5.4.1, “Selecting General Query Log and Slow Query Log Output Destinations”](https://dev.mysql.com/doc/refman/8.0/en/log-destinations.html "5.4.1 Selecting General Query Log and Slow Query Log Output Destinations"), [Section 5.4.3, “The General Query Log”](https://dev.mysql.com/doc/refman/8.0/en/query-log.html "5.4.3 The General Query Log"), 그리고 [Section 5.4.5, “The Slow Query Log”](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html "5.4.5 The Slow Query Log") 참고.

릴레이 로그는 레플리카에서도 적용되어야할 복제 원본 서버의 변경 데이터를 저장하기 위해 레플리카에서만 사용된다. 릴레이 로그 내용과 설정에 대해서는 [Section 17.2.4.1, “The Relay Log”](https://dev.mysql.com/doc/refman/8.0/en/replica-logs-relaylog.html "17.2.4.1 The Relay Log") 참고.

오래된 로그 파일의 만기일 같은 로그 유지 보수 작업에 대한 정보는 [Section 5.4.6, “Server Log Maintenance”](https://dev.mysql.com/doc/refman/8.0/en/log-file-maintenance.html "5.4.6 Server Log Maintenance") 참고.

로그 보안에 대해서는 [Section 6.1.2.3, “Passwords and Logging”](https://dev.mysql.com/doc/refman/8.0/en/password-logging.html "6.1.2.3 Passwords and Logging") 참고.