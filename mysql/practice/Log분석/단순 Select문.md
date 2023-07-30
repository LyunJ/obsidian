```sql
select * from employees limit 10;
```

# Trace Log
## sql_parse.cc: do_command(THD*)
MySQL은 커넥션 하나당 하나의 스레드가 할당되어 커넥션의 작업을 수행하게 된다. 사용자가 MySQL에 접속하게 되면 스레드가 생성되어 커넥션이 끊어질 때 스레드는 사라진다. 이 스레드를 포그라운드 스레드라고 한다. 포그라운드 스레드는 사용자가 요청하는 작업을 직접 수행하게 된다. 백그라운드 스레드는 사용자가 접근할 수 없지만 사용자의 작업을 돕거나 데이터 보호를 위한 작업을 한다. 예를 들어 사용자에 의해 변경된 데이터를 DB손상에 대비하여 redo log에 복사하는 작업을 하고, 잠금이나 데드락을 감지하는 작업을 한다. 

백그라운드 스레드는 모니터링이나 손상 방지 등의 작업을 주로 하는데 이 때문에 포그라운드 스레드와 병렬적으로 작업을 할 수 있을 것 처럼 보인다. 하지만 몇몇 작업과 상황에 대해서는 포그라운드가 백그라운드 스레드의 작업을 기다려야 하는 상황이 발생한다. **innodb_flush_log_at_trx_commit** 이 기본값인 1로 되어있을 경우 데이터가 추가되거나 변경되었을 때 그 즉시 redo log file에 기록한다. 포그라운드 스레드는 로그 버퍼에 데이터를 기록하는데, 나머지 작업인 로그 버퍼에서 OS 버퍼로 이동시키는 작업과 OS 버퍼에서 disk file로 이동시키는 작업은 백그라운드 스레드에서 수행하게 된다. 따라서 포그라운드 스레드는 백그라운드 스레드의 작업을 기다려야만 한다.

엔터프라이즈 버전의 경우 스레드 풀에서 대기 상태의 스레드를 가져와 작업을 하거나 스케쥴러에 의해 할당받은 스레드를 사용한다. 이는 CPU에서 코어에 작업을 할당하는 것과 비슷하다. 

do_command(THD*)는 커넥션에 의해 할당받은 스레드에서 사용자가 요청한 작업을 수행하는 함수이다. 커넥션이 발생하면 스레드와 함께 생성된 스레드의 정보를 담고 있는 THD라는 스레드 설명자 클래스의 인스턴스가 할당된다. 이 후 스레드가 쿼리를 받을 준비를 끝마치면 `thd_connection_alive(thd)`가 True이고 do_command(THD*)가 False일때 do_command(THD*)를 반복하여 실행하게 된다. (그런데 쿼리 입력 대기 상태일 때 왜 do_commmand에 대한 br이 작동하지 않는지 모르겠음)

## THD::enter_stage
```text
  sql_parse.cc:   548: do_command: THD::enter_stage: 'starting' /Users/moti/CLionProjects/mysql-server/sql/conn_handler/init_net_server_extension.cc:103
```

enter_stage는 