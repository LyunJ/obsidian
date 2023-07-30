`mysql` 스키마는 시스템 스키마다. 이 스키마는 실행중인 MySQL이 필요로 하는 정보를 저장하는 테이블을 담고 있다. 광범위하게 분류하자면,`mysql` 스키마는 데이터베이스 오브젝트 메타데이터를 저장하는 `data dictionary table`들과 다른 운영 목적으로 사용되는 `system table`들을 담고 있다. 아래의 리스트는 시스템 테이블 셋을 더 작은 범주로 세분화한다.

- Data Dictionary Tables
- Grant System Tables
- Object Information System Tables
- Log System Tables
- Server-Side Help System Tables
- Time Zone System Tables
- Replication System Tables
- Optimizer System Tables
- Miscellaneous System Tables

이 글의 나머지 부분은 테이블들의 각각의 카테고리를 추가적인 정보와 함께 열거한다. 데이터 딕셔너리 테이블과 시스템 테이블은 달리 지정되지 않는 한 InnoDB 스토리지 엔진을 사용한다.

`mysql` 시스템 테이블과 데이터 딕셔너리 테이블은 MySQL 데이터 디렉토리에 있는 `mysql.idb`라는 하나의 InnoDB tablespace 파일에 저장되어 있다.

미사용 데이터 암호화는 `mysql` 시스템 스키마 tablespace에서 활성화 할 수 있다. 더 많은 정보를 얻고 싶으면 [InnoDB Data-at-Rest Encryption](https://dev.mysql.com/doc/refman/8.0/en/innodb-data-encryption.html) 참고

## Data Dictionary Tables
이 테이블들은 데이터베이스 오브젝트들에 관한 메타데이터를 갖고 있는 데이터 딕셔너리로 이루어져 있다. 더 많은 정보를 얻고 싶으면 [MySQL Data Dictionary](https://dev.mysql.com/doc/refman/8.0/en/data-dictionary.html) 참고

> **중요**
> 
> 데이터 딕셔너리는 MySQL 8.0에 새로 추가되었다. 데이터 딕셔너리를 사용할 수 있는 서버는 이전 MySQL 릴리즈와 비교하여 몇 가지 일반적인 운영상의 차이를 수반한다. 더 상세한 내용은 [Data Dictionary Usage Differences](https://dev.mysql.com/doc/refman/8.0/en/data-dictionary-usage-differences.html) 참고. 또한, MySQL 5.7에서 MySQL 8.0으로 업그레이드를 위해서는, 이전 MySQL 릴리즈와는 약간 다른 절차를 거쳐야하고, 해당 절차는 구체적은 필요 조건을 만족함으로써 업그레이드 준비를 검증받아야 한다. 더 많은 정보를 얻고 싶으면, [Upgrading MySQL](https://dev.mysql.com/doc/refman/8.0/en/upgrading.html) 특히 [Preparing Your Installation for Upgrade](https://dev.mysql.com/doc/refman/8.0/en/upgrade-prerequisites.html) 참고 바람

- catalogs: 카탈로그 정보
- character_sets: 가능한 character sets 정보
- check_constraints: 테이블에 정의된 `CHECK` 제약 정보.  [CHECK Constraints](https://dev.mysql.com/doc/refman/8.0/en/create-table-check-constraints.html) 참고
- collations: 각각의 character set의 collation 정보
- column_statistics: 컬럼값의 히스토그램 통계. [Optimizer Statistics](https://dev.mysql.com/doc/refman/8.0/en/optimizer-statistics.html) 참고
- column_type_elements: 컬럼에서 사용하는 타입 정보
- columns: 테이블의 컬럼에 대한 정보
- dd_properties: version: 같은 데이터 딕셔너리 속성을 식별하는 테이블. 서버는 이를 사용해 새 버전으로 업그레이드 돼아 할 데이터 딕셔너리인지 아닌지 결정한다.
- events: 이벤트 스케쥴러가 관리하는 여러 이벤트 정보. [Using the Event Scheduler](https://dev.mysql.com/doc/refman/8.0/en/event-scheduler.html) 참고. 만약 서버가 `--skip-grant-tables` 옵션과 함께 실행됐으면, 이벤트 스케쥴러는 비활성화되고 테이블에 등록되어있는 이벤트들은 실행되지 않는다. [Event Scheduler Configuration](https://dev.mysql.com/doc/refman/8.0/en/events-configuration.html) 참고
- foreign_keys, foreign_key_column_usage: 외래키에 대한 정보
- index_column_usage: 인덱스에 의해 사용된 컬럼에 대한 정보
- index_partitions: 인덱스에 의해 파티션에 대한 정보
- index_stats: `ANALYZE_TABLE` 실행되었을 때 생성된 다이나믹 인덱스 통계를 저장하는데 사용
- indexes: 테이블 인덱스에 대한 정보
- innodb_ddl_log: crash-safe한 DDL 작업을 위해 DDL logs을 저장
- parameter_type_elements: stored procedure와 function parameter에 대한 정보. 그리고 stored function의 return value에 대한 정보
- parameters: stored procedure와 function에 대한 정보.[Using Stored Routines](https://dev.mysql.com/doc/refman/8.0/en/stored-routines.html) 참고.
- resource_groups: 리소스 그룹에 대한 정보. [Resource Groups](https://dev.mysql.com/doc/refman/8.0/en/resource-groups.html) 참고.
- routines: stored procedure와 function에 대한 정보. [Using Stored Routines](https://dev.mysql.com/doc/refman/8.0/en/stored-routines.html) 참고.
- schemata: schemata에 대한 정보. MySQL에선 스키마는 데이터베이스이기 때문에 이 테이블은 데이터베이스에 대한 정보를 제공한다
- st_spatial_reference_systems: 공간 데이터에 사용 가능한 공간 참조 시스템에 대한 정보
- table_partition_values: 테이블 파티션에 사용되는 값에 대한 정보
- table_partitions: 테이블에 사용되는 파티션에 대한 정보.
- table_stats: `ANALYZE_TABLE`이 실행됐을 때 생성된 다이나믹 테이블 통계에 대한 정보
- tables: 데이터베이스 내부의 테이블에 대한 정보
- tablespace_files: tablespace가 사용하는 파일에 대한 정보
- tablespaces: 사용중인 tablespace들의 정보
- triggers: 트리거에 대한 정보
- view_routine_usage: view와 view에서 사용하는 stored function 간의 종속성에 대한 정보
- view_table_usage: view와 view의 기본 테이블 간의 종속성을 추적하는데 사용

데이터 딕셔너리 테이블은 직접 볼 수 없다. `SELECT`절로 조회할 수 없고, `SHOW TABLES`절로도 조회할 수 없고, `INFORMATION_SCHEMA.TABLES` 테이블 리스트 등등에도 찾을 수 없다. 하지만 대부분 경우에는 쿼리로 조회할 수 있는 해당  `INFORMATION_SCHEMA`테이블이 있다. 개념적으로, `INFORMATION_SCHEMA` 는 데이터 사전 메타데이터를 노출시키는 뷰를 제공한다. 

예를 들어, `mysql.schemata` 테이블에 직접 조회할 수 없다.

```
SELECT * FROM mysql.schemata;
[2022-09-21 21:22:02] [HY000][3554] Access to data dictionary table 'mysql.schemata' is rejected. 
```

대신, `INFORMATION_SCHEMA` 테이블을 통해 조회해 보자
```
mysql> SELECT * FROM INFORMATION_SCHEMA.SCHEMATA\G

*************************** 1. row ***************************
              CATALOG_NAME: def
               SCHEMA_NAME: mysql
DEFAULT_CHARACTER_SET_NAME: utf8mb4
    DEFAULT_COLLATION_NAME: utf8mb4_0900_ai_ci
                  SQL_PATH: NULL
        DEFAULT_ENCRYPTION: NO
*************************** 2. row ***************************
              CATALOG_NAME: def
               SCHEMA_NAME: information_schema
DEFAULT_CHARACTER_SET_NAME: utf8mb3
    DEFAULT_COLLATION_NAME: utf8mb3_general_ci
                  SQL_PATH: NULL
        DEFAULT_ENCRYPTION: NO
*************************** 3. row ***************************
              CATALOG_NAME: def
               SCHEMA_NAME: performance_schema
DEFAULT_CHARACTER_SET_NAME: utf8mb4
    DEFAULT_COLLATION_NAME: utf8mb4_0900_ai_ci
                  SQL_PATH: NULL
        DEFAULT_ENCRYPTION: NO
*************************** 4. row ***************************
              CATALOG_NAME: def
               SCHEMA_NAME: sys
DEFAULT_CHARACTER_SET_NAME: utf8mb4
    DEFAULT_COLLATION_NAME: utf8mb4_0900_ai_ci
                  SQL_PATH: NULL
        DEFAULT_ENCRYPTION: NO
*************************** 5. row ***************************
              CATALOG_NAME: def
               SCHEMA_NAME: refactoring
DEFAULT_CHARACTER_SET_NAME: utf8mb4
    DEFAULT_COLLATION_NAME: utf8mb4_0900_ai_ci
                  SQL_PATH: NULL
        DEFAULT_ENCRYPTION: NO
*************************** 6. row ***************************
              CATALOG_NAME: def
               SCHEMA_NAME: employees
DEFAULT_CHARACTER_SET_NAME: utf8mb4
    DEFAULT_COLLATION_NAME: utf8mb4_0900_ai_ci
                  SQL_PATH: NULL
        DEFAULT_ENCRYPTION: NO
*************************** 7. row ***************************
              CATALOG_NAME: def
               SCHEMA_NAME: dbstudy
DEFAULT_CHARACTER_SET_NAME: utf8mb4
    DEFAULT_COLLATION_NAME: utf8mb4_0900_ai_ci
                  SQL_PATH: NULL
        DEFAULT_ENCRYPTION: NO
7 rows in set (0.01 sec)

ERROR:
No query specified
```

`mysql.indexes`은 위의 방법으로 찾을 수 없지만 `INFORMATION_SCHEMA.STATISTICS`는 대부분 비슷한 정보를 담고 있다.

또한 아직까지는 `mysql.foreign_keys, mysql.foreign_key_column_usage`도 위의 방법으로 찾을 수 없다. 외래키 정보를 얻는 SQL를 이용한 표준 방법은 `INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS, INFORMATION_SCHEMA.KEY_COLUMN_USAGE`를 사용하는 것이다. 이러한 테이블들은 `foreign_keys, foreign_key_column_usage` 뷰와 다른 데이터 딕셔너리 테이블들로 구현되어 있다.

MySQL8.0 이전의 몇몇 시스템 테이블들은 데이터 딕셔너리 테이블로 대체되었고, mysql 시스템 스키마에 존재하지 않는다.

- `event` 데이터 딕셔너리 테이블은 MySQL8.0이전 버전의 `event` 테이블을 대체한다
- `parameters`와 `routines`데이터 딕셔너리 테이블들은 MySQL8.0이전 버전의 `proc`테이블을 대체한다

## Grant System Tables
이 시스템 테이블들은 유저 계정에 대한 승인 정보와 유저의 권한 정보를 가지고 있다. [Grant Table](https://dev.mysql.com/doc/refman/8.0/en/grant-tables.html)

현재 MySQL 8.0에서는 grant table은 InnoDB(transactional) table이다. 이전에는 이는 MyISAM(nontransactional) table이었다. grant table 저장 엔진의 변경은 MySQL 8.0의 `CREATE USER`와 `GRANT`와 같은 계정 관리 쿼리문의 동작에 수반되는 변경의 기초가 된다. 이전에는 여러명의 사용자를 지정하는 계정 관리 쿼리문은 일부 사용자에게 성공하고 다른 사용자에게 실패할 수 있었다. 쿼리문은 이제 transactional하고, 유저를 지정하거나 롤백하는데 부작용이나 에러가 발생하지 않는다.

> 만약 MySQL이 이전 버전에서 업그레이드 됐지만 grant table이 MyISAM에서 InnoDB로 업그레이드 되지 않았다면, 서버는 그들을 읽기 전용으로 간주하고 계정 관리 쿼리문은 에러를 발생시킨다. 업그레이드 설명에 대해서는 [Upgrading MySQL](https://dev.mysql.com/doc/refman/8.0/en/upgrading.html) 참고

- user: 유저 계정, global 권한, 그리고 다른 비권한 컬럼.
- global_grants: 유저에게 동적 전역 권한 할당. [Static Versus Dynamic Privileges](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#static-dynamic-privileges) 참고
- db: 데이터베이스 레벨의 권한
- tables_priv: 테이블 레벨의 권한
- columns_priv: 컬럼 레벨의 권한
- procs_priv: stored procedure와 function의 권한
- proxies_priv: 프록시 유저 권한
- default_roles: 이 테이블은 사용자가 연결하여 인증하거나 `SET ROLE DEFAULT` 를 실행한 이후에 활성화되어야할 디폴트 역할이 나열되어 있다.
- role_edges: 역할 서브그래프의 엣지를 나열한다
	- 주어진 사용자 테이블의 행은 사용자 계정이나 역할에 의해 언급될 것이다. 서버는 행이 사용자 계정이든 역할이든 혹은 둘 다를 나타내던지 상관없이 인증 ID간의 관계에 대한 정보를 위한 `role_edges` 테이블을 조회함으로써 구분할 수 있다.
- password_history: 비밀번호 변경에 대한 정보

## Object Information System Tables
이 시스템 테이블은 컴포넌트, load 가능한 함수, 서버사이드 플러그인에 대한 정보를 가지고 있다.
- component: `INSTALL COMPONENT`를 사용하여 설치된 서버 컴포넌트 레지스트리. 이 테이블에 나열된 모든 컴포넌트들은 서버 시작 단계 중에 loader service에 의해 설치된다. [Installing and Uninstalling Loadable Functions](https://dev.mysql.com/doc/refman/8.0/en/component-loading.html) 참고
- func: `CREATE FUNCTION`을 사용하여 설치된 loadable function의 레지스트리. 일반적이 시작 단계 동안에, 서버는 이 테이블에 등록된 함수들을 load한다. 만약 서버가 `--skip-grant-tables` 옵션과 함께 시작되었다면 이 테이블에 등록된 함수들은 load되지 않고 비활성화된다. 
> `mysql.func` 시스템 테이블 처럼 `performance_schema.user_defined_functions` 테이블은 `CREATE FUNCTION`으로 설치된 load 가능한 함수를 나열한다. `mysql.func` 테이블과는 다르게, `user_defined_functions` 테이블은 서버 컴포넌트나 플러그인에 의해 자동으로 설치된 함수들 또한 나열할 수 있다. 이 차이는 설치된 함수를 확인할 때 `mysql.func` 보다 `user_defined_functions`를 더 선호하게 만든다. [The user_defined_functions Table](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-user-defined-functions-table.html) 참고

- plugins: `INSTALL PLUGIN`을 사용해 설치된 서버사이드 플러그인의 레지스트리. 일반적인 시작 단계 동안 서버는 이 테이블에 등록된 플러그인들을 load한다. 만약 서버가 `--skip-grant-tables` 옵션과 함께 시작되었다면, 이 테이블에 등록된 플러그인은 load되지 않고 비활성화된다. [Installing and Uninstalling Plugins](https://dev.mysql.com/doc/refman/8.0/en/plugin-loading.html) 참고

## Log System Tables
서버는 로깅을 위해 아래의 시스템 테이블들을 사용한다

- general_log: 일반적인 쿼리 로그 테이블
- slow_log: 느린 쿼리 로그 테이블

로그 테이블은 `CSV` 스토리지 엔진을 사용한다.
더 많은 정보는 [MySQL Server Logs](https://dev.mysql.com/doc/refman/8.0/en/server-logs.html) 참고

## Server-Side Help System Tables
이 시스템 테이블들은 server-side help 정보를 담고 있다
- help_category: help categories에 대한 정보
- help_keyword: help 주제에 관한 키워드
- help_relation: help 키워드와 주제간의 매핑
- help_topic: help 주제 내용

더 많은 정보는 [Server-Side Help Support](https://dev.mysql.com/doc/refman/8.0/en/server-side-help-support.html) 참고

## Time Zone System Tables
이 시스템 테이블들은 표준 시간대 정보를 담고 있다

- time_zone: 표준 시간대 ID 및 해당 시간대 ID의 윤초 사용 여부
- time_zone_leap_second: 윤초가 발생할 때
- time_zone_name: 표준시간대 ID와 이름간의 매핑
- time_zone_transition, time_zone_transition_type: 표준 시간대 설명

더 많은 정보는 [MySQL Server Time Zone Support](https://dev.mysql.com/doc/refman/8.0/en/time-zone-support.html) 참고

## Replication System Tables
서버는 복제를 지원하기 위해 이 시스템 테이블을 사용한다

- gtid_executed: GTID 값을 저장하기 위한 테이블. [mysql.gtid_executed Table](https://dev.mysql.com/doc/refman/8.0/en/replication-gtids-concepts.html#replication-gtids-gtid-executed-table) 참고
- ndb_binlog_index: NDB Cluster replication의 바이너리 로그 정보. 이 테이블은 서버가 `NDBCLUSTER` 지원과 함께 빌드되었을 때 생성된다. [NDB Cluster Replication Schema and Tables](https://dev.mysql.com/doc/refman/8.0/en/mysql-cluster-replication-schema.html) 참고
- slave_master_info, slave_relay_log_info, slave_worker_info: 레플리카 서버에 replication 정보를 저장하기 위해 사용. [Relay Log and Replication Metadata Repositories](https://dev.mysql.com/doc/refman/8.0/en/replica-logs.html) 참고

위의 나열된 테이블들은 `InnoDB` 스토리지 엔진에서만 사용된다.

## Optimizer System Tables
이 시스템 테이블은 옵티마이저가 사용하기 위한 것이다.

- innodb_index_stats, innodb_table_stats: 영구적인 `InnoDB` 옵티마이저 통계를 위해 사용된다. [Configuring Persistent Optimizer Statistics Parameters](https://dev.mysql.com/doc/refman/8.0/en/innodb-persistent-stats.html) 참고
- server_cost, engine_cost: 옵티마이저 비용 모델은 쿼리가 실행되는 동안 발생하는 작업에 대한 비용 추정 정보를 담고 있는 테이블을 사용한다. `server_cost`는 일반적인 서버 작업에 대한 옵티마이저 비용 추정값을 담고 있다. `engine_cost`는 특정 스토리지 엔진의 작업의 추정값을 가지고 있다. [The Optimizer Cost Model](https://dev.mysql.com/doc/refman/8.0/en/cost-model.html) 참고

## Miscellaneous System Tables
이외의 시스템 테이블은 이전 카테고리와 맞지 않다.

- audit_log_filter, audit_log_user: 만약 MySQL Enterprise Audit이 설치되면, 이 테이블을 영구적인 audit log filter 정의와 사용자 계정 스토리지를 제공한다. [Audit Log Tables](https://dev.mysql.com/doc/refman/8.0/en/audit-log-reference.html#audit-log-tables) 참고
- firewall_group_allowlist, firewall_groups, firewall_membership, firewall_users, firewall_whitelist: 만약 MySQL Enterprise Firewall 이 설치되었다면, 이 테이블은 영구적인 방화벽에 대한 정보를 제공한다. [MySQL Enterprise Firewall](https://dev.mysql.com/doc/refman/8.0/en/firewall.html) 참고
- servers: `FEDERATED` 스토리지 엔진에서 사용한다. [Creating a FEDERATED Table Using CREATE SERVER](https://dev.mysql.com/doc/refman/8.0/en/federated-create-server.html) 참고
- innodb_dynamic_metadata: auto-increment counter 값이나 index tree corruption flags같은 빠르게 바뀌는 테이블 메타데이터를 저장하기 위해 `InnoDB` 스토리지 엔진에서 사용. `InnoDB` 시스템 테이블스페이스에 저장되어 있는 데이터 딕셔너리 버퍼 테이블을 교체시킬 수 있다.