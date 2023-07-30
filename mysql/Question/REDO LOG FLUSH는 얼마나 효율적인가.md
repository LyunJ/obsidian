redo log와 double write buffer의 존재의의는 MySQL 서버가 비정상적인 종료를 했을 경우 disk에 flush 되지 않은 데이터들을 보존해두고 있다가 다시 MySQL이 구동되었을 때 기록해주는 것이다. 또한 redo log와 double write buffer는 dirty page(redo log는 dirty page의 변경점만)를 기록해 두고 있다가 특정 시점에 한번에 flush 함으로써 disk I/O의 더 효율적인 동작과 쿼리 증폭의 완충제 역할도 한다.

하지만 여기서 발생하는 호기심은 다른 쿼리 요청이 없는 환경에서 REDO LOG를 사용하는 것이 트랜잭션이 COMMIT 될 때마다 DISK에 WRITE 하는 것 보다 얼마나 효율적인가 이다.

MySQL 성능 최적화라는 책에서는

> InnoDB는 로그를 사용해서 랜덤 디스크 I/O를 순차적인 I/O로 변환합니다

라고 언급되어있다.
즉, 트랜잭션이 COMMIT 될 때마다 DISK WRITE하는 것은 랜덤 I/O이고, log를 활용하는 것은 순차적 I/O를 사용한다. 그렇다면 어떤 방식으로 log를 활용하여 랜덤 I/O를 순차적 I/O로 변경시키는지 알아보자.



