# 해석 순서
id, select_type, table, paritions, type, possible_keys, key, key_len, ref, rows, filtered, Extras
id : 단위 select 쿼리를 구분해주는 속성. 중요하지 않음
select_type : 각 단위 select 쿼리가 어떤 타입의 쿼리인지 표시되는 컬럼
table : 실행 계획을 세우는 기준점이 되는 테이블에 대한 정보
partitions : 쿼리 처리를 위해 필요한 파티션을 ,로 구분
type : 조인 타입
possible_keys : 사용 후보 인덱스
key : 최종 선택된 인덱스
key_len : 조회에 사용한 인덱스 키의 크기
ref : 비교 조건에 사용된 컬럼
rows : 쿼리를 처리하기 위해 거쳐가는 총 row 수
filtered : 필터링되고 남은 레코드의 비율
extra : 위의 정보를 제외한 쿼리 실행 계획
# id
id는 단위 SELECT 쿼리마다 달리 주어지는 번호이다. 만약 단위 SELECT 문장이 다른 테이블을 사용하는 SUB 쿼리를 가진다 하더라도 SELECT로 시작하는 구문이 없다면 같은 번호가 주어지게 된다.
id값은 순서와는 상관 없기 때문에 착각해서는 안된다.

