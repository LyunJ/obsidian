
# 개요
데이터베이스의 격리 수준은 크게 READ UNCOMMITED, READ COMMITED, REPEATABLE READ 그리고 SERIALIZABLE로 4가지가 있다. READ UNCOMMITED와 SERIALIZABLE은 각자의 특성 때문에 범용적으로 사용되기 어려운 격리 수준이지만, READ COMMITED와 REPEATABLE READ는 많은 데이터베이스에서 사용하고 있는 격리 수준으로 각각의 구현 방식도 데이터베이스마다 다르다. 

READ COMMITED는 더티 리드를 허용하지 않음으로서 데이터의 일관성을 지키는 격리 수준이다. 더티 리드란 트랜잭션의 결과가 COMMIT되지 않았음에도 변경사항들이 조회가 되는 경우를 뜻한다. READ COMMITED는 undo log에 변경되기 전 데이터를 기록하여 변경작업을 하는 트랜잭션이 실행되는 동안에는 undo log에 기록된 변경 전 데이터를 조회할 수 있어 더티 리드가 발생하지 않는다. 하지만 만약 트랜잭션이 진행되는 동안에 같은 작업영역을 가지는 다른 트랜잭션이 COMMIT을 하게 된다면 어떻게 될까? 트랜잭션이 진행되던 중 같은 영역의 데이터를 여러번 조회하게 될 경우 일관되지 않은 결과값이 조회되기 때문에 하나의 트랜잭션에서 여러 작업을 하기 어려워지게 된다. 이 같은 현상을 NONREPEATABLE-READ라고 한다.

InnoDB의 기본 격리 수준은 REPEATABLE READ이다. REPEATABLE READ이란 같은 트랜잭션 내에서는 같은 질의에 같은 결과를 얻을 수 있는 조건을 만족하는 격리 수준이다. 이전 단계의 NONREPEATABLE-READ 를 해결한 방법인데, MYSQL은 이를 MVCC를 사용하여 해결하였다.


## InnoDB MVCC
MVCC는 다중 버전 동시성 제어라고도 하는데, 이 기술을 통해 데이터의 동시성과 일관성을 높일 수 있다. 