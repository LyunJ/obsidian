GTID 복제 방식으로 연결되어있는 두 데이터베이스에서 복제 연결이 안된 데이터베이스를 그대로 다른 데이터베이스에 넣어줘야할 상황이 생겼다. 현재 데스크탑 컴퓨터와 맥북에 각각 하나씩 노드가 설치되어있고 맥북 노드가 Source 노드였는데, Replica 노드의 복제 연결이 안되어있는 testdata 데이터베이스를 Source노드로 가져와야했다. testdata는 실시간 트랜잭션의 영향을 받지 않는 데이터베이스라 dump를 사용해 데이터를 한꺼번에 가져와도 상관없을 것 같았고, 그렇게 진행을 했지만 문제가 생겼다.

## Issue
```
# 데이터베이스 덤핑
mysqldump -h localhost -u root -p -D testdata > testdatadump
```

```
# 데이터베이스 로드
mysql -h localhost -u root -p -D testdata < ./testdatadump.sql

ERROR: ASCII '\0' appeared in the statement, but this is not allowed unless option --binary-mode is enabled and mysql is run in non-interactive mode. Set --binary-mode to 1 if ASCII '\0' is expected. Query: '��-'.
```

### 해결과정
위의 에러를 구글에 검색해본 결과 --binary-mode를 추가해서 실행해보라는 답변을 받았다.

```
# binary-mode 추가
mysql -h localhost -u root -p -D testdata --binary-mode < ./testdatadump.sql

ERROR 1064 (42000) at line 1: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' at line 1
```

--binary-mode를 추가하고 실행했더니 단순 구문 오류가 발생하였다.  --binary-mode는 mysqlbinlog의 결과물에 BLOB 타입의 데이터가 있을 때 사용하는 옵션으로, \\r\\n을 \\n으로, 질의 종결자인 \\0를 무시하게 만든다. binary log에 쓰이는 명령어가 dump file에 쓰였기 때문에 질의문 종결자를 무시하여 발생한 오류라고 생각한다. 그렇다면 \\0는 어디에 쓰이는 단어일까? 

순간 인코딩 문제일 수 있다는 생각이 들어 해당 파일의 문자열 인코딩을 봤더니 utf16le로 되어있었다. mysqldump에 --default-character-set을 지정하지 않으면 자동으로 utf8mb4로 설정이 되는데 이상한 일이다. 나는 윈도우 환경에서 dump파일을 생성하여 iCloud에 올린 뒤 맥북에서 다운받았다. 윈도우에서 iCloud로 옮길때나 혹은 iCloud에서 맥북으로 옮길 때 파일 인코딩 방식이 바뀐 것 같았다.

그럼 해결법은 덤프파일의 문자열 인코딩을 utf8mb4로 변경하거나 덤프 파일을 로드하는 mysql client의 character-set을 utf16le로 바꾸는 것이었다. 

첫번째 방법은 iconv를 통해 수행하려고 했다.

```
# utf16le -> utf8mb4로 변경
iconf -f utf16le -t utf8mb4 testdatadump.sql > testdatadump_utf8mb4.sql

iconv: conversion to utf8mb4 unsupported
```
하지만 utf8mb4는 mysql에서만 지원하는 character-set이었으므로 iconv에서는 지원하지 않았다.

두번째 방법은 --default-character-set 옵션을 통해 수행할 수 있었다.
```
# mysql client character set 변경
mysql -h localhost -u root -p -D testdata --default-character-set=utf16le < testdatadump.sql

ERROR 1231 (42000): Variable 'character_set_client' can't be set to the value of 'utf16le'
```
하지만 character_set_client 변수는 utf16le로 설정될 수 없었기 때문에, 다른 방법을 찾아야만 했다.

마지막으로 덤프 파일을 utf8로 변경하고 --default-character-set으로 변경하는 방식을 선택했다.
utf8은 예전 mysql의 기본 문자열 인코딩 값인 utf8mb3과 같기 때문에 character_set_client 변수에 담을 수 있을 것이라 생각했다.

```
# dump file character set을 utf8로 변경 후 mysql client character set도 utf8로 변경
iconv -f utf-16le -t utf-8 testdatadump.sql > testdatadump_utf8.sql

mysql -h localhost -u root -p -D testdata --default-character-set=utf8 < ./testdatadump_utf8.sql
```
결과는 성공적이었고, 모든 데이터가 잘 들어간 것을 확인할 수 있었다.

### 부작용

```
show full colums from user\G;

...
*************************** 8. row ***************************
     Field: delete_dt
      Type: timestamp
 Collation: NULL
      Null: YES
       Key:
   Default: NULL
     Extra:
Privileges: select,insert,update,references
   Comment: ?좎? ??젣 ????젣 ?쇱떆 湲곕줉
8 rows in set (0.01 sec)
```
한글이 깨지는 현상이 발생했다. 

인코딩 변환 과정에서 문제가 생긴 것 같으니 덤프 파일을 생성하고 로드하는 프로세스를 살펴보았다.
1. 기본 데이터(utf8mb4)
2. 덤프파일 생성(utf-16le)
3. 덤프파일 인코딩 변경 (utf-8)
4. mysql client 인코딩 변경(utf-8)
기본 데이터에서 덤프파일을 생성하는 과정에서 문제가 생겼거나 덤프파일의 인코딩이 utf-16le로 변경되는 과정에서 문제가 생긴 것 같다.
```
# utf16le
CREATE TABLE `user` (
`id` int NOT NULL AUTO_INCREMENT,
`status` enum('ACTIVATE','DEACTIVATE') NOT NULL,
`name` varchar(10) NOT NULL,
`email` varchar(30) NOT NULL COMMENT 'ID ??븷',
`password` char(32) NOT NULL,
`create_dt` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
`update_dt` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
`delete_dt` timestamp NULL DEFAULT NULL COMMENT '?좎? ??젣 ????젣 ?쇱떆 湲곕줉',
PRIMARY KEY (`id`),
KEY `user_name_index` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=1000001 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```
utf-16le로 인코딩된 덤프파일을 살펴보니 이미 한글이 깨져있는 것을 알 수 있다. 따라서 utf8mb4가 모종의 이유로 utf16le로 변경되었는데, 이 때 한글 변환이 제대로 이루어지지 않아 한글이 깨져 보이는 것 같다.