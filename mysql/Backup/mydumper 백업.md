테스트 환경 : [[MySQL 복제 구축]]

처음부터 모든 데이터를 논리적 백업으로 백업을 한다면 매우 오랜 시간이 걸릴 수 있다. 왜냐하면 논리적 백업으로 생성된 SQL을 구문 분석하고 실행하는 비용이 크기 때문이다.

매우 큰 테이블이 저장된 데이터베이스를 복원하는데 얼마나 많은 시간이 걸리는지 테스트 해보겠다.

# 데이터베이스 전체 논리적 백업 테스트
아래와 같이 1억 2천만건의 데이터가 삽입된 테이블과 이를 파티션한 테이블이 저장된 데이터베이스를 백업하고 복원해보겠다.

```
mysql> show tables;
+-------------------------+
| Tables_in_tuning        |
+-------------------------+
| goods                   |
| order_goods_nopartition |
| order_goods_partition   |
+-------------------------+
3 rows in set (0.01 sec)

mysql> select count(*) from order_goods_nopartition;
+-----------+
| count(*)  |
+-----------+
| 120000000 |
+-----------+
1 row in set (21.65 sec)
```

먼저 데이터가 저장된 위치를 파악한다.
```
mysql> show variables like 'datadir'
    -> ;
+---------------+--------------------------+
| Variable_name | Value                    |
+---------------+--------------------------+
| datadir       | /opt/homebrew/var/mysql/ |
+---------------+--------------------------+
1 row in set (0.02 sec)
```

아래와 같이 14GB의 비파티셔닝 테이블에 대한 데이터 파일과 10년치의 월별 파티셔닝된 테이블에 대한 데이터 파일이 저장되어 있는 것을 확인할 수 있다.
```
ls -al /opt/homebrew/var/mysql/tuning
drwxr-x--- moti admin 3.9 KB Tue Apr  4 00:01:52 2023  .
drwxr-xr-x moti admin 1.2 KB Sat Aug 19 20:37:22 2023  ..
.rw-r----- moti admin 9.0 MB Thu Mar 30 23:21:35 2023  goods.ibd
.rw-r----- moti admin  14 GB Fri Mar 31 17:41:43 2023  order_goods_nopartition.ibd
.rw-r----- moti admin 144 KB Tue Apr  4 00:13:29 2023  order_goods_partition#p#p.ibd
.rw-r----- moti admin 112 MB Tue Apr  4 00:14:39 2023  order_goods_partition#p#p_201001.ibd
.rw-r----- moti admin 112 MB Tue Apr  4 00:13:29 2023  order_goods_partition#p#p_201002.ibd
...
```

위의 데이터베이스를 mydumper를 통해 백업해보겠다.
```
mydumper -h localhost -u root -a  --database tuning
```

// 실행 시간 넣기

mydumper 실행 이후 아래와 같이 6.1GB의 논리적 백업 파일이 생성된 것을 볼 수 있다.
```
mydumper ls
 export-20230928-150528   export-20231004-003133

export-20231004-003133 ls -al
drwxr-x--- moti staff 352 B  Wed Oct  4 00:32:30 2023  .
drwxr-xr-x moti staff 128 B  Wed Oct  4 00:31:33 2023  ..
.rw-r--r-- moti staff 502 B  Wed Oct  4 00:32:30 2023  metadata
.rw-r--r-- moti staff 155 B  Wed Oct  4 00:31:33 2023  tuning-schema-create.sql
.rw-r--r-- moti staff   0 B  Wed Oct  4 00:31:33 2023  tuning-schema-triggers.sql
.rw-r--r-- moti staff 310 B  Wed Oct  4 00:31:33 2023  tuning.goods-schema.sql
.rw-r--r-- moti staff 768 KB Wed Oct  4 00:31:33 2023  tuning.goods.00000.sql
.rw-r--r-- moti staff 615 B  Wed Oct  4 00:31:33 2023  tuning.order_goods_nopartition-schema.sql
.rw-r--r-- moti staff 6.1 GB Wed Oct  4 00:32:21 2023  tuning.order_goods_nopartition.00000.sql
.rw-r--r-- moti staff 8.7 KB Wed Oct  4 00:31:33 2023  tuning.order_goods_partition-schema.sql
.rw-r--r-- moti staff 6.1 GB Wed Oct  4 00:32:30 2023  tuning.order_goods_partition.00000.sql
```

총 12GB가 넘는 논리적 백업 파일로 복구를 수행했을 때 얼마나 많은 시간이 소요되는지 확인해 보겠다.
```
myloader --directory=export-20231004-003133 --overwrite-tables --user=root

** (myloader:54296): WARNING **: 00:37:42.575: zstd command not found on any static location, use --exec-per-thread for non default locations

# 완료 시각 : 01:40
```

12GB의 데이터를 논리적 백업을 통해 복구했을 경우 1시간이 넘는 시간이 걸렸다. 멀티 스레드 기능을 사용하지 않아 그런 것도 있지만 상당히 많은 시간이 걸렸다.
