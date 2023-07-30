사용자가 mysqld에 접속하면 새로운 connection이 생성되면서 유저 스레드를 할당받게 된다. THD는 스레드와 1:1 매칭되는 Class이며, 유저가 작용시킬 여러 동작들의 메타데이터를 가지고 있게 된다.

THD는 handle_connection 함수에서 생성되는데, handle_connection 함수가 하는 일은 다음과 같다
- 스레드 초기화
- 생성된 스레드와 함께 사용될 THD 초기화
- 유저 인증
- 현재 연결된 connection에서 요청된 쿼리 모두 실행
- connection 종료
- 슬레드 종료 / 스레드 캐시의 스레드를 사용해서 다음 connection 처리