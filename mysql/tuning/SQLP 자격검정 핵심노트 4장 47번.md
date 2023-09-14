-- 주문상품은 비파티션 테이블  
-- 한 달 주문상품 = 100만건  
-- 주문상품의 보관 기간 = 10년  
-- 주문상품 총 건수 = 총 1억 2천만 건  
-- 할인 유형코드 조건을 만족하는 데이터 비중 = 20%
-- 등록된 상품 = 2만 개  
-- 2만 개 상품을 고르게 주문

비파티션 테이블 : 풀 테이블 스캔을 염두에 둬야함
할인 유형 코드로 80%의 데이터를 거를 수 있으므로 인덱스 생성을 고려해야함
한 달 단위로 조인 시 상품 테이블은 모든 데이터를 조회하기 때문에 풀 테이블 스캔을 염두에 둬야함



# goods 테이블 생성 및 데이터 저장

``` mysql
DELIMITER ;;  
create procedure tb_goods(IN insert_num INTEGER)  
BEGIN  
    DECLARE i INTEGER DEFAULT 1;  
    SET AUTOCOMMIT = FALSE;  
    START TRANSACTION ;  
    WHILE i < insert_num DO  
        INSERT INTO goods (id, goods_name, goods_price) VALUES  
            (i,SUBSTR(MD5(i),10),rand()*10000);  
        end while ;  
    COMMIT;  
    SET AUTOCOMMIT = FALSE;  
end;;  
DELIMITER ;
```

``` mysql
SET AUTOCOMMIT = FALSE;  
CALL tb_goods(20001);  
SET AUTOCOMMIT = TRUE;


tuning> CALL tb_goods(20001)
[2023-03-30 23:21:34] completed in 242 ms
```

# order_goods_nopartition 테이블 생성 및 데이터 저장
``` mysql
DELIMITER ;;  
CREATE PROCEDURE tb_order_goods(IN insert_month_num INTEGER)  
BEGIN  
    DECLARE i INTEGER DEFAULT 1;  
    DECLARE j INTEGER DEFAULT 1;  
    DECLARE month_count INTEGER DEFAULT 1000000;  
    DECLARE STANDARD_DATE DATETIME DEFAULT STR_TO_DATE('20100101', '%Y%m%d');  
  
  
    START TRANSACTION ;  
    WHILE i < insert_month_num + 1  
        DO  
            WHILE j < month_count + 1  
                DO  
                    INSERT INTO order_goods_nopartition (order_count, order_price, discount_code, order_datetime, goods_id)  
                    VALUES (RAND() * 10, RAND() * 10000, CONCAT('CODE', i % 5),  
                            FROM_UNIXTIME(UNIX_TIMESTAMP(STANDARD_DATE) + FLOOR(0 + (RAND() * 2160000))), j % 20000);  
                    SET j = j + 1;  
                end while;  
            SET STANDARD_DATE = DATE_ADD(STANDARD_DATE, INTERVAL 1 MONTH);  
            SET i = i + 1;  
            SET j = 1;  
        end while;  
    COMMIT;  
end;;  
DELIMITER ;
```

``` mysql
SET AUTOCOMMIT = FALSE;  
CALL tb_order_goods(120);  
SET AUTOCOMMIT = TRUE;

tuning> CALL tb_order_goods(120)
[2023-03-31 01:06:41] completed in 1 h 14 m 39 s 993 ms
```

# 실행(13개월치 데이터, 분포 불균형)
discount_code(CODE0,CODE1,CODE2,CODE3,CODE4)가 한 달 단위로 라운드 로빈방식으로 변경되며 저장됨.

## 튜닝 전
``` mysql
EXPLAIN ANALYZE  
SELECT P.id, MIN(P.goods_name) as goods_name, MIN(P.goods_price) as goods_price,  
       SUM(O.order_count) as total_order_count, SUM(O.order_price) as total_order_price  
FROM order_goods_nopartition O , goods P  
WHERE order_datetime >= date_sub(str_to_date('2019-01-01','%Y-%m-%d'),INTERVAL 1 MONTH ) AND  
      discount_code = 'CODE1' AND  
      P.id = O.goods_id  
GROUP BY P.id  
ORDER BY total_order_price DESC, P.id;
```
1회차 수행
```
-> Sort: total_order_price DESC, p.id  (actual time=643547.028..643547.028 rows=0 loops=1)  
    -> Stream results  (cost=17476276.55 rows=19571) (actual time=643547.018..643547.018 rows=0 loops=1)  
        -> Group aggregate: min(p.goods_name), min(p.goods_price), sum(o.order_count), sum(o.order_price)  (cost=17476276.55 rows=19571) (actual time=643547.017..643547.017 rows=0 loops=1)  
            -> Nested loop inner join  (cost=17368312.31 rows=1079642) (actual time=643547.016..643547.016 rows=0 loops=1)  
                -> Index scan on P using PRIMARY  (cost=2053.05 rows=19571) (actual time=0.822..44.425 rows=20000 loops=1)  
                -> Filter: ((o.discount_code = 'CODE1') and (o.order_datetime >= <cache>((now() + interval -(1) month))))  (cost=806.31 rows=55) (actual time=32.174..32.174 rows=0 loops=20000)  
                    -> Index lookup on O using order_goods_nopartition_goods_id_fk (goods_id=p.id)  (cost=806.31 rows=810) (actual time=0.877..31.770 rows=6000 loops=20000)


[2023-03-31 01:49:12] 1 row retrieved starting from 1 in 10 m 43 s 576 ms (execution: 10 m 43 s 560 ms, fetching: 16 ms)
```

## 튜닝 시작
### 인덱스 추가
``` mysql
ALTER TABLE order_goods_nopartition ADD INDEX order_goods_nopartition_X1 (discount_code,order_datetime);
[2023-03-31 01:59:52] completed in 2 m 33 s 652 ms

ALTER TABLE order_goods_nopartition ADD INDEX order_goods_nopartition_X2 (goods_id,order_datetime);
[2023-03-31 15:53:06] completed in 1 m 48 s 881 ms
```

``` mysql
EXPLAIN ANALYZE  
SELECT P.id, MIN(P.goods_name) as goods_name, MIN(P.goods_price) as goods_price,  
       SUM(O.order_count) as total_order_count, SUM(O.order_price) as total_order_price  
FROM order_goods_nopartition O , goods P  
WHERE order_datetime >= date_sub(str_to_date('2019-01-01','%Y-%m-%d'),INTERVAL 1 MONTH ) AND  
      discount_code = 'CODE1' AND  
      P.id = O.goods_id  
GROUP BY P.id  
ORDER BY total_order_price DESC, P.id;
```

```
-> Sort: total_order_price DESC, p.id  (actual time=21368.081..21369.183 rows=19999 loops=1)  
    -> Table scan on <temporary>  (actual time=21357.337..21360.465 rows=19999 loops=1)  
        -> Aggregate using temporary table  (actual time=21357.329..21357.329 rows=19998 loops=1)  
            -> Nested loop inner join  (cost=6122161.40 rows=4000598) (actual time=34.350..17288.995 rows=1999900 loops=1)  
                -> Filter: (o.goods_id is not null)  (cost=4721952.10 rows=4000598) (actual time=34.343..15916.281 rows=2000000 loops=1)  
                    -> Index range scan on O using order_goods_nopartition_X1 over (discount_code = 'CODE1' AND '2018-12-01 00:00:00' <= order_datetime), with index condition: ((o.discount_code = 'CODE1') and (o.order_datetime >= <cache>((str_to_date('2019-01-01','%Y-%m-%d') - interval 1 month))))  (cost=4721952.10 rows=4000598) (actual time=34.341..15842.597 rows=2000000 loops=1)  
                -> Single-row index lookup on P using PRIMARY (id=o.goods_id)  (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=2000000)

[2023-03-31 15:11:37] 1 row retrieved starting from 1 in 21 s 390 ms (execution: 21 s 379 ms, fetching: 11 ms)
```

Driving Table : order_goods_nopartitions
Driving Table row expectation : 2백만 개
Driven Table : goods

200만 개의 row를 가져와서 각각의 goods_id와 goods 테이블의 id와 Join할 때 Primary Index를 사용하여 단일 행을 검색함. index scan을 200만번 하지만, MySQL은 Primary key로 단일 검색했을 때의 성능이 매우 빠르므로 좋은 결과가 나왔음.


### 상품 테이블을 Driving Table로 설정했을 때
``` mysql
EXPLAIN ANALYZE  
SELECT /*+ JOIN_ORDER(P,O) */ P.id, MIN(P.goods_name) as goods_name, MIN(P.goods_price) as goods_price,  
       SUM(O.order_count) as total_order_count, SUM(O.order_price) as total_order_price  
FROM order_goods_nopartition O , goods P  
WHERE order_datetime >= date_sub(str_to_date('2019-01-01','%Y-%m-%d'),INTERVAL 1 MONTH ) AND  
      discount_code = 'CODE1' AND  
      P.id = O.goods_id  
GROUP BY P.id  
ORDER BY total_order_price DESC, P.id;
```

```
-> Sort: total_order_price DESC, p.id  (actual time=101206.891..101208.009 rows=19999 loops=1)  
    -> Stream results  (cost=18100537.32 rows=19571) (actual time=41.159..101187.536 rows=19999 loops=1)  
        -> Group aggregate: min(p.goods_name), min(p.goods_price), sum(o.order_count), sum(o.order_price)  (cost=18100537.32 rows=19571) (actual time=41.152..101162.549 rows=19999 loops=1)  
            -> Nested loop inner join  (cost=18043334.56 rows=572028) (actual time=17.568..100734.416 rows=1999900 loops=1)  
                -> Index scan on P using PRIMARY  (cost=2053.05 rows=19571) (actual time=1.043..23.237 rows=20000 loops=1)  
                -> Filter: (o.discount_code = 'CODE1')  (cost=838.03 rows=29) (actual time=2.257..5.032 rows=100 loops=20000)  
                    -> Index lookup on O using order_goods_nopartition_X2 (goods_id=p.id), with index condition: (o.order_datetime >= <cache>((str_to_date('2019-01-01','%Y-%m-%d') - interval 1 month)))  (cost=838.03 rows=838) (actual time=1.627..4.977 rows=650 loops=20000)

[2023-03-31 15:55:43] 1 row retrieved starting from 1 in 1 m 41 s 229 ms (execution: 1 m 41 s 217 ms, fetching: 12 ms)
```

Driving Table : goods
Driving Table row expectation : 2만 개
Driven Table : order_goods_nopartition

2만개의 row를 가져와서 id와 order_goods_nopartition 테이블의 goods_id를 비교하여 Join한다. 2만 개의 row에 대해 각각 order_goods_nopartition_X2 인덱스를 range scan하고 Table Access한 뒤, discount_code가 일치하는 row를 가져와 Join한다. 즉, order_goods_nopartition_X2 인덱스 range scan과 필터링 작업을 2만 번 하게 되는 것이다.  order_goods_nopartition_X2 인덱스 range scan 결과 예측치는 ((120000000 : 총 row 수/20000 : goods 수)/ 120) * 13 = 650 개이고, 650개의 데이터에 대해 discount_code와 일치하는지 비교 작업을 실시하므로, 인덱스 range scan은 2만 번 수행하고 필터링 작업은 650 * 20000 = 1300만 번 수행하므로 느린 결과를 보여준다.

### HASH JOIN 사용
``` mysql
EXPLAIN ANALYZE  
SELECT /*+ BNL(P,O) */ P.id, MIN(P.goods_name) as goods_name, MIN(P.goods_price) as goods_price,  
       SUM(O.order_count) as total_order_count, SUM(O.order_price) as total_order_price  
FROM goods P IGNORE INDEX (`PRIMARY`), order_goods_nopartition O IGNORE INDEX(order_goods_nopartition_X2)  
WHERE order_datetime >= date_sub(str_to_date('2019-01-01','%Y-%m-%d'),INTERVAL 1 MONTH ) AND  
      discount_code = 'CODE1' AND  
      P.id = O.goods_id  
GROUP BY P.id  
ORDER BY total_order_price DESC, P.id;
```

```
-> Sort: total_order_price DESC, p.id  (actual time=60552.806..60553.913 rows=19999 loops=1)  
    -> Table scan on <temporary>  (actual time=60542.036..60545.263 rows=19999 loops=1)  
        -> Aggregate using temporary table  (actual time=60542.033..60542.033 rows=19998 loops=1)  
            -> Inner hash join (p.id = o.goods_id)  (cost=7834329393.50 rows=4000598) (actual time=56772.969..57113.420 rows=1999900 loops=1)  
                -> Table scan on P  (cost=0.01 rows=19571) (actual time=0.033..2.894 rows=20000 loops=1)  
                -> Hash  
                    -> Index range scan on O using order_goods_nopartition_X1 over (discount_code = 'CODE1' AND '2018-12-01 00:00:00' <= order_datetime), with index condition: ((o.discount_code = 'CODE1') and (o.order_datetime >= <cache>((str_to_date('2019-01-01','%Y-%m-%d') - interval 1 month))))  (cost=4722385.43 rows=4000598) (actual time=65.163..56478.537 rows=2000000 loops=1)

[2023-03-31 16:23:04] 1 row retrieved starting from 1 in 1 m 0 s 610 ms (execution: 1 m 0 s 602 ms, fetching: 8 ms)
```

row 수가 2만 개로 상대적으로 적은 goods 테이블이 build 테이블이 되고, order_goods_nopartition 테이블이 probe 테이블이 된다. 

goods 테이블로 In-memory Hash Table을 만들고, 약 200만 개의 row가 각각 조회 키로 비교하여 결과 집합을 생성한다. 

NL Join보다 상대적으로 느린데, discount_code 분포가 한달 단위로 몰려있어 인접한 페이지에 위치할 가능성이 높기 때문에 NL Join은 Table Access 단계에서 빠른 속도를 보였을 것으로 추측된다(Explain에서 MRR 사용이 확인됨). Hash Join도 마찬가지로 충분한 혜택을 누렸을 것 같지만, 충분히 많은 데이터를 사용하지 않아 더 느린 결과가 나온 것으로 보인다.

# discount_code 분포 변경(균형)
discount_code(CODE0,CODE1,CODE2,CODE3,CODE4)가 한 달 내의 데이터에도 균등하게 분포되도록 변경하였음.

### 인덱스 추가
```
-> Sort: total_order_price DESC, p.id  (actual time=57047.666..57047.888 rows=4000 loops=1)  
    -> Table scan on <temporary>  (actual time=57044.977..57045.631 rows=4000 loops=1)  
        -> Aggregate using temporary table  (actual time=57044.970..57044.970 rows=4000 loops=1)  
            -> Nested loop inner join  (cost=7501451.66 rows=4910974) (actual time=55.681..52089.212 rows=2600000 loops=1)  
                -> Filter: (o.goods_id is not null)  (cost=5782610.76 rows=4910974) (actual time=55.672..50047.742 rows=2600000 loops=1)  
                    -> Index range scan on O using order_goods_nopartition_X1 over (discount_code = 'CODE1' AND '2018-12-01 00:00:00' <= order_datetime), with index condition: ((o.discount_code = 'CODE1') and (o.order_datetime >= <cache>((str_to_date('2019-01-01','%Y-%m-%d') - interval 1 month))))  (cost=5782610.76 rows=4910974) (actual time=55.670..49949.964 rows=2600000 loops=1)  
                -> Single-row index lookup on P using PRIMARY (id=o.goods_id)  (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=2600000)

[2023-03-31 17:56:46] 1 row retrieved starting from 1 in 57 s 69 ms (execution: 57 s 57 ms, fetching: 12 ms)
```

이전과 같은 실행계획이지만, 시간이 증가했다.
Driving Table의 row 수가 200만 개에서 260만 개로 증가한 것도 있지만, MRR의 혜택을 제대로 받지 못한 것도 시간 증가에 영향을 줬을 것 같다.

### 상품 테이블을 Driving Table로 설정했을 때
```
-> Sort: total_order_price DESC, p.id  (actual time=119796.829..119797.047 rows=4000 loops=1)  
    -> Stream results  (cost=17784397.97 rows=19571) (actual time=61.594..119789.485 rows=4000 loops=1)  
        -> Group aggregate: min(p.goods_name), min(p.goods_price), sum(o.order_count), sum(o.order_price)  (cost=17784397.97 rows=19571) (actual time=61.587..119774.816 rows=4000 loops=1)  
            -> Nested loop inner join  (cost=17717689.19 rows=667088) (actual time=4.839..119237.826 rows=2600000 loops=1)  
                -> Index scan on P using PRIMARY  (cost=1981.35 rows=19571) (actual time=0.064..19.092 rows=20000 loops=1)  
                -> Filter: (o.discount_code = 'CODE1')  (cost=822.16 rows=34) (actual time=5.097..5.956 rows=130 loops=20000)  
                    -> Index lookup on O using order_goods_nopartition_X2 (goods_id=p.id), with index condition: (o.order_datetime >= <cache>((str_to_date('2019-01-01','%Y-%m-%d') - interval 1 month)))  (cost=822.16 rows=830) (actual time=1.774..5.901 rows=650 loops=20000)

[2023-03-31 17:47:41] 1 row retrieved starting from 1 in 1 m 59 s 813 ms (execution: 1 m 59 s 803 ms, fetching: 10 ms)
```

실행 계획은 이전과 같고 실행 시간은 약간 증가했다.

### Hash Join

```
-> Sort: total_order_price DESC, p.id  (actual time=71743.784..71744.007 rows=4000 loops=1)  
    -> Table scan on <temporary>  (actual time=71741.226..71741.856 rows=4000 loops=1)  
        -> Aggregate using temporary table  (actual time=71741.222..71741.222 rows=4000 loops=1)  
            -> Inner hash join (p.id = o.goods_id)  (cost=9617107841.14 rows=4910974) (actual time=66972.759..67384.530 rows=2600000 loops=1)  
                -> Table scan on P  (cost=0.01 rows=19571) (actual time=0.501..10.385 rows=20000 loops=1)  
                -> Hash  
                    -> Index range scan on O using order_goods_nopartition_X1 over (discount_code = 'CODE1' AND '2018-12-01 00:00:00' <= order_datetime), with index condition: ((o.discount_code = 'CODE1') and (o.order_datetime >= <cache>((str_to_date('2019-01-01','%Y-%m-%d') - interval 1 month))))  (cost=5795626.13 rows=4910974) (actual time=57.210..66574.629 rows=2600000 loops=1)

[2023-03-31 17:54:57] 1 row retrieved starting from 1 in 1 m 11 s 799 ms (execution: 1 m 11 s 789 ms, fetching: 10 ms)
```

NL Join과의 실행 시간 차이가 줄어들었다.
먼저 Probe Table의 row 개수가 260만 개로 60만개 증가하였다. Hash Join은 소규모 테이블과 대규모 테이블을 조인할 때 가장 효과적이므로 Probe Table의 row 수가 증가한 것이 실행 시간 차이를 줄인데 영향을 끼친 것 같다.

# 25개월치 데이터 (균일 분포)
order_goods_nopartition 테이블의 row 수를 늘려가며 NL Join과 Hash Join의 실행 시간을 측정하여 Hash Join이 효과를 보이는 조건을 찾고자 함

### 인덱스 추가
```
-> Sort: total_order_price DESC, p.id  (actual time=114861.770..114861.997 rows=4000 loops=1)  
    -> Table scan on <temporary>  (actual time=114858.940..114859.590 rows=4000 loops=1)  
        -> Aggregate using temporary table  (actual time=114858.937..114858.937 rows=4000 loops=1)  
            -> Nested loop inner join  (cost=15482037.87 rows=10115676) (actual time=56.761..105197.564 rows=5000000 loops=1)  
                -> Filter: (o.goods_id is not null)  (cost=11941551.27 rows=10115676) (actual time=56.754..101227.052 rows=5000000 loops=1)  
                    -> Index range scan on O using order_goods_nopartition_X1 over (discount_code = 'CODE1' AND '2017-12-01 00:00:00' <= order_datetime), with index condition: ((o.discount_code = 'CODE1') and (o.order_datetime >= <cache>((str_to_date('2018-01-01','%Y-%m-%d') - interval 1 month))))  (cost=11941551.27 rows=10115676) (actual time=56.752..101038.125 rows=5000000 loops=1)  
                -> Single-row index lookup on P using PRIMARY (id=o.goods_id)  (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=5000000)

[2023-03-31 18:04:19] 1 row retrieved starting from 1 in 1 m 54 s 891 ms (execution: 1 m 54 s 876 ms, fetching: 15 ms)
```


### Hash join
```
-> Sort: total_order_price DESC, p.id  (actual time=119826.606..119826.828 rows=4000 loops=1)  
    -> Table scan on <temporary>  (actual time=119823.971..119824.629 rows=4000 loops=1)  
        -> Aggregate using temporary table  (actual time=119823.968..119823.968 rows=4000 loops=1)  
            -> Inner hash join (p.id = o.goods_id)  (cost=19809421981.10 rows=10115676) (actual time=110634.350..111413.960 rows=5000000 loops=1)  
                -> Table scan on P  (cost=0.01 rows=19571) (actual time=0.052..3.160 rows=20000 loops=1)  
                -> Hash  
                    -> Index range scan on O using order_goods_nopartition_X1 over (discount_code = 'CODE1' AND '2017-12-01 00:00:00' <= order_datetime), with index condition: ((o.discount_code = 'CODE1') and (o.order_datetime >= <cache>((str_to_date('2018-01-01','%Y-%m-%d') - interval 1 month))))  (cost=11939816.53 rows=10115676) (actual time=65.783..109888.454 rows=5000000 loops=1)

[2023-03-31 18:08:13] 1 row retrieved starting from 1 in 1 m 59 s 856 ms (execution: 1 m 59 s 844 ms, fetching: 12 ms)
```

NL Join과 실행 시간이 조금 더 줄어들었다.

# 49개월치 데이터(균일)
### 인덱스 추가
```
-> Sort: total_order_price DESC, p.id  (actual time=431137.934..431138.161 rows=4000 loops=1)  
    -> Stream results  (cost=18010188.87 rows=19571) (actual time=62.833..431128.303 rows=4000 loops=1)  
        -> Group aggregate: min(p.goods_name), min(p.goods_price), sum(o.order_count), sum(o.order_price)  (cost=18010188.87 rows=19571) (actual time=62.827..431108.278 rows=4000 loops=1)  
            -> Nested loop inner join  (cost=17674978.64 rows=3352102) (actual time=2.917..429107.124 rows=9800000 loops=1)  
                -> Index scan on P using PRIMARY  (cost=1981.35 rows=19571) (actual time=0.052..44.860 rows=20000 loops=1)  
                -> Filter: (o.discount_code = 'CODE1')  (cost=819.98 rows=171) (actual time=17.515..21.435 rows=490 loops=20000)  
                    -> Index lookup on O using order_goods_nopartition_X2 (goods_id=p.id), with index condition: (o.order_datetime >= <cache>((str_to_date('2016-01-01','%Y-%m-%d') - interval 1 month)))  (cost=819.98 rows=830) (actual time=2.150..21.228 rows=2450 loops=20000)

[2023-03-31 18:18:45] 1 row retrieved starting from 1 in 7 m 11 s 174 ms (execution: 7 m 11 s 160 ms, fetching: 14 ms)
```

실행 계획에 변화가 생겼다. goods 테이블이 Driving Table이 되고, order_goods_nopartition 테이블이 Driven Table이 됐다. 또한 order_goods_nopartition_X2 인덱스를 사용하면서 인덱스 스캔은 2만 번이면 되고,  하나의 goods_id에 대한 인덱스 스캔 결과가 2450개가 되었고, 문자열 비교 횟수는 4900만 번이 되었다.
만약 실행 계획이 변경되지 않았으면, Primary index를 980만번 탐색해야한다. 문자열 비교 횟수는 크게 상관이 없는 것 같고, index range scan 2만 번(2450 rows each)과 unique index scan 980만번은 전자가 더 빠르다고 판단되는 것 같다.

### Hash join
```
-> Sort: total_order_price DESC, p.id  (actual time=50406.923..50407.157 rows=4000 loops=1)  
    -> Table scan on <temporary>  (actual time=50404.158..50404.784 rows=4000 loops=1)  
        -> Aggregate using temporary table  (actual time=50404.151..50404.151 rows=4000 loops=1)  
            -> Inner hash join (p.id = o.goods_id)  (cost=48309620876.93 rows=24677543) (actual time=32306.692..33806.370 rows=9800000 loops=1)  
                -> Table scan on P  (cost=0.04 rows=19571) (actual time=0.053..3.127 rows=20000 loops=1)  
                -> Hash  
                    -> Filter: ((o.discount_code = 'CODE1') and (o.order_datetime >= <cache>((str_to_date('2016-01-01','%Y-%m-%d') - interval 1 month))))  (cost=12327207.33 rows=24677543) (actual time=18657.722..31582.568 rows=9800000 loops=1)  
                        -> Table scan on O  (cost=12327207.33 rows=119644934) (actual time=0.142..23762.643 rows=120000000 loops=1)

[2023-03-31 18:21:00] 1 row retrieved starting from 1 in 50 s 436 ms (execution: 50 s 429 ms, fetching: 7 ms)
```
여기도 실행 계획에 변화가 생겼다. goods 테이블과 order_goods_nopartition 테이블 모두 full table scan을 한다. 
그리고 실행 시간이 오히려 줄었다. NL Join보다 더 빠른 결과를 기대했는데 이전 실행계획보다 빨라진 것을 보고 이전 쿼리들도 Hash Join을 build, probe 테이블 모두 풀 테이블 스캔으로 유도하여 진행해 보았다.

결과는 13개월치 34초, 25개월치 35초, 49개월치 50초로 모든 경우에서 더 빨랐다. Hash Join에서는 인덱스를 사용하는 것 보다 풀 테이블 스캔으로 처리하는 것이 더 효율적으로 보인다. 하지만 풀 테이블 스캔이 느려질 정도로 테이블의 총 row 수가 많고, 선택되는 데이터가 적으면 인덱스가 효율적일 수도 있을 것 같다.

# 정리
| 최적화 방법 | 실행 시간 | discount_code 컬럼 분포 변경(골고루) | 25개월치 데이터 | 49개월치 데이터 |
| -- | -- | -- | -- | -- |
| 튜닝 안함 | 10 m 43 s 576 ms| - | - | - |
| 인덱스 추가(힌트 없이) | 21 s 379 ms| 57 s 69 ms | 1 m 54 s 891 ms | 7 m 11 s 174 ms |
| 상품 테이블을 driving 테이블로 변경 | 1 m 41 s 229 ms | 1 m 59 s 813 ms | - | - |
| Hash Join 사용 | 1 m 0 s 610 ms | 1 m 11 s 799 ms | 1 m 59 s 856 ms | 50 s 436 ms |
| Hash Join (full table scan) | - |  34 s 998 ms | 35 s 599 ms | 50 s 436 ms |

## Driving 테이블이 상품주문 테이블일 때가 더 빨랐다 

지금까지 Driving Table의 개수를 줄여 NL Join의 반복 횟수를 줄이는 것이 좋은 성능을 낼 것이라고 생각했다. 하지만 개수가 적은 테이블이 Primary Index를 사용하여 조회된다면 MySQL의 특성상 Primary Index 조회가 매우 빨라 데이터가 많은 테이블이 Driving Table일 경우가 더 빠를 수 있다.

discount_code 컬럼을 균형적으로 분포한 케이스로 비교해보면,
주문상품 테이블이 Driving Table일 경우 260만 개의 row를 가지게 되고, 2만개인 상품 테이블과 Join을 한다면 상품 테이블의 Primary Index를 조회하는 횟수 즉, Ramdom I/O가 260만 회 진행된다.
상품 테이블이 Driving Table일 경우 2만 개의 row를 가지게 되고, 주문상품 테이블의 인덱스를 range scan하게 된다. range scan의 결과 개수는 ((테이블 총 row 수 / goods_id cardinality) / 저장하는 월 개수) * 13개월
로 구한다(650개).

range scan으로 650개의 결과를 2만 회 조회하는 것과 Primary Index를 260만 회 조회하는 것의 차이이고 Primary Index를 260만 회 조회하는 것이 더 빨랐다. 하지만 조회하는 데이터 수가 많아졌을 때, 옵티마이저는 상품 테이블을 Driving Table로 선택하였다. 왜냐하면 상품 테이블이 Driving Table일 때는 인덱스를 조회하는 횟수는 2만회로 고정되지만, 주문상품 테이블이 Driving Table일 경우 Primary Index를 조회하는 횟수는 조회하려는 데이터 수가 증가하면 같이 증가하기 때문이다. 49개월치 데이터를 조회한 케이스로 위의 계산대로 비교해 보면, range scan으로 2450개의 결과를 2만 회 조회하는 것과 Primary Index를 980만 회 조회하는 것 중 옵티마이저가 판단하기에 Primary Index를 980만 회 조회하는 것이 더 비효율적이라고 판단하였다. 

## Hash Join에서 Index를 사용하는 것 보다 Full Table Scan이 더 빠른 이유

Index를 사용하게 되면 랜덤 엑세스가 필연적으로 따른다. mysql은 특히 클러스터링 인덱스를 사용하기 때문에 검색하는 과정에서 disk I/O가 많이 발생할 것이다. 따라서 rowid를 통해 직접 데이터 블록의 위치에 접근하는 다른 DBMS와는 다르게 인덱스 레인지 스캔(커버링 인덱스가 아닐 때)의 범위가 넓어질 수록 성능이 떨어질 것이다.

위 실험에서 25개월과 49개월의 Hash Join은 각각 Probe Input을 가져올 때 Index range scan과 Full table scan을 하였다. 25개월의 경우 500만개의 데이터를 Index를 range scan하는데 `actual time=65.783..109888.454` 으로 `109822.67 ms` 가 걸린다. 49개월은 1억 2천만개의 데이터를 풀 테이블 스캔으로 가져오는데 `actual time=0.142..23762.643` 으로 `23762.501 ms`가 걸린다. 풀 테이블 스캔이 훨씬 빠르다.