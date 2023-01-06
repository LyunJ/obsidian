```
ALTER TABLE testdata.user_profile modify nick varchar(12) not null
[2023-01-05 14:06:58] 1,000,000 rows affected in 2 s 179 ms
```

| operation | INSTANT | INPLACE | MODIFY TABLE |
| -- | -- | -- | -- |
| Changing the column data type | NO | NO | YES |

```csvtable
source: mysql/test_data/schema_change/char_to_varchar.csv
```
```csvtable
source: mysql/test_data/schema_change/tmp_create.csv
```
Online DDL이 MODIFY TABLE로 실행되었기 때문에 전체 테이블을 복사한 임시 테이블이 생성되고 임시테이블의 이름을 변경한 것을 알 수 있다.