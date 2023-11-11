# MySQL 설치
## 다운로드 및 설치
### 설치 환경
- Tibero가 설치된 시스템의 root 계정
### 다운로드 및 패키지 설정
[MySQL Community Download](https://dev.mysql.com/downloads/repo/yum/)

위의 링크에서 rpm을 다운 받거나, 다운로드 링크를 복사하여 설치를 진행한다.

```bash
# RHEL8일 경우
yum install https://dev.mysql.com/get/mysql80-community-release-el8-8.noarch.rpm
```

### Default MySQL Module 설정(EL8 시스템만)
EL8 시스템일 경우 MySQL 모듈이 기본적으로 enable 상태인데, 이것을 끄지 않으면 다운 받은 MySQL을 사용하지 못한다.

```bash
sudo yum module disable mysql
```

### MySQL 설치
```bash
sudo yum install mysql-community-server
```
이 명령어는 mysql-server와 client, 에러 메시지나 character set 정보를 가진 mysql-common, shared client library(mysql-libs)들을 포함하고 있다.

### MySQL 실행
```bash
systemctl start mysqld
```

## MySQL 초기 설정
### root 사용자 비밀번호 변경
처음 서버를 기동했을 때, 데이터 디렉토리가 비어있으면 MySQL은 다음과 같은 작업을 자동으로 수행한다.

- 서버 초기 설정 수행(기본 설정된 위치에 필요한 디렉터리 생성)
- SSL 인증과 key file이 data directory 에 생성
- validate_password(비밀번호 정책 컴포넌트)가 설치되고 적용됨
- superuser 계정인 'root'@'localhost'가 생성됨 

root의 비밀번호는 임시 비밀번호로 `/var/log/mysqld.log`에 임시 비밀번호가 저장되어 있다. 
- 아래 명령어로 임시 비밀번호를 찾아 로그인한다.
```bash
sudo grep 'temporary password' /var/log/mysqld.log
```

- 로그인
```bash
mysql -uroot -p
```

- 로그인했다면, root 유저의 비밀번호를 변경한다.
```bash
ALTER USER 'root'@'localhost' IDENTIFIED BY '비밀번호';
```

### 계정 생성 및 데이터베이스 생성
티베로 java gateway를 통해서 MySQL에 접속하기 위해 새로운 계정과 데이터베이스를 생성해준다.

#### MySQL 8.0 비밀번호 정책 변경
MySQL 8.0부터 비밀번호 정책이 강화되어 반드시 대문자와 소문자를 섞어 써야 하고, 숫자와 특수 문자를 하나씩 사용해야 한다.

하지만 Tibero에서 Gateway를 통해 접속할 시 유저 계정이 모두 대문자로 변환되고, 비밀번호의 특수 문자를 명령어의 일부로 인식하여 문제가 발생한다.

따라서 유저 계정을 처음부터 대문자로 생성하고, 비밀번호 정책 설정을 변경하여 비밀번호에 특수 문자를 사용하지 않아도 되도록 한다.

```bash
su -
vi /etc/my.cnf
```
```
# 맨 아래줄에 추가
validate_password.policy=LOW 
```

MySQL 설정 파일의 수정이 끝났다면, MySQL을 재시작한다.
```bash
systemctl restart mysqld
```

#### 계정 생성 및 데이터베이스 생성
- 계정 생성
```mysql
create user 'TIBERO'@'%' identified by 'TiberoTmax';
```

- 권한 설정
```mysql
grant all privileges on *.* to 'TIBERO'@'%';
flush privileges;
```

- 데이터베이스 생성
```bash
# bash
mysql -u TIBERO -p
```

```mysql
create database mysql8;
```

# Gateway 설치
## MySQL Connector 설치
### MySQL Connector binary 다운로드
[MySQL Connector Download](https://dev.mysql.com/downloads/connector/j/)
위의 링크에서 Platform Independent를 선택하여 TAR 압출 파일을 다운 받는다.

### MySQL Connector 설치
## Java Gateway 설치
### 설치 환경
- Tibero가 설치된 시스템의 Tibero 관리 계정

### Java Gateway 설치
```bash
# 사용자 홈 디렉토리에서 시작
mkdir gwMysql
cp -Rf $TB_HOME/client/bin/tbJavaGW.zip gwMysql/
cd gwMysql
```

`unzip -l` 명령어로 zip 파일의 내부를 확인하여 내부 파일을 감싸주는 상위 폴더가 있는지 확인한다.
```bash
unzip -l tbJavaGW.zip
```
```
Archive:  tbJavaGW.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  11-25-2021 17:19   tbJavaGW/
        0  11-25-2021 17:19   tbJavaGW/lib/
     1301  11-25-2021 17:19   tbJavaGW/jgw.cfg
     2264  11-25-2021 17:19   tbJavaGW/jgw_service.bat
      421  11-25-2021 17:19   tbJavaGW/jgwlog.properties
   559366  11-25-2021 17:19   tbJavaGW/lib/commons-collections.jar
    24019  11-25-2021 17:19   tbJavaGW/lib/commons-daemon-1.0.6.jar
    42492  11-25-2021 17:19   tbJavaGW/lib/commons-pool.jar
   391834  11-25-2021 17:19   tbJavaGW/lib/log4j-1.2.15.jar
   485167  11-25-2021 17:19   tbJavaGW/lib/tbgateway.jar
      792  11-25-2021 17:19   tbJavaGW/tbgw
---------                     -------
  1507656                     11 files
```

tbJavaGW라는 디렉토리가 있으므로 그대로 압축을 푼다.
```bash
unzip tbJavaGW.zip
```

## MySQL Connector 설치
### MySQL Connector binary 다운로드
[MySQL Connector Download](https://dev.mysql.com/downloads/connector/j/)
위의 링크에서 Platform Independent를 선택하여 TAR 압축 파일을 다운 받는다.

### MySQL Connector Jar 파일 이동
다운 받은 connector 압축 파일을 홈 디렉토리에 옮기고 압축을 해제한다.
``` bash
tar -zxvf mysql-connector-j-8.1.0.tar.gz
```

mysql-connector-j-8.1.0 폴더로 이동하여 jar 파일이 존재하는지 확인한다.
```bash
cd mysql-connector-j-8.1.0/
ls
```
```
CHANGES   INFO_SRC  README     mysql-connector-j-8.1.0.jar
INFO_BIN  LICENSE   build.xml  src
```

jar 파일을 위에서 설치한 게이트웨이의 lib 디렉토리에 복사한다.
```bash
cp mysql-connector-j-8.1.0.jar ~/gwMysql/tbJavaGW/lib
```

## 게이트웨이 설정
### 게이트웨이 환경 설정 파일 수정
게이트웨이의 환경 설정 파일을 다음과 같이 수정한다.
MySQL 8.0부터 DataSource 클래스명이 변했기 때문에 아래 이름을 참고한다.
```bash
vi ~/gwMysql/tbJavaGW/jgw.cfg
```
```
# Target database
DATABASE=MySQL

# Datasource class name for target database
# use with DATABASE=JDBC30 option.
# MySQL 8.0 Connector의 Datasource 클래스명
DATASOURCE_CLASS_NAME=com.mysql.cj.jdbc.MysqlDataSource

# XA datasource class name for target database
# use with DATABASE=JDBC30 option.
# MySQL 8.0 Connector의 XADatasource 클래스명
XA_DATASOURCE_CLASS_NAME=com.mysql.cj.jdbc.MysqlXADataSource
  
# Listener port
LISTENER_PORT=8083
```

### 게이트웨이 실행 스크립트 파일 수정
게이트웨이 실행 스크립트 파일을 다음과 같이 수정한다.
위에서 이동시킨 MySQL Connector의 경로를 수정하고, 게이트웨이 실행 명령문에 mysql connector jar 정보를 입력한다.
```bash
vi ~/gwMysql/tbJavaGW/tbgw
```
```
#Classpath
commonsdaemon=./lib/commons-daemon-1.0.6.jar
commonspool=./lib/commons-pool.jar
commonscollections=./lib/commons-collections.jar
log4j=./lib/log4j-1.2.15.jar
msjdbc=./lib/sqljdbc.jar:./lib/sqljdbc4.jar
asejdbc=./lib/jconn3.jar
postgresqljdbc=./lib/postgresql-8.4-701.jdbc3.jar
gateway=./lib/tbgateway.jar
mysqljdbc=./lib/mysql-connector-j-8.1.0.jar # 추가

#log4j properties
#log4jfile must be exists on classpath
log4jfile=jgwlog.properties

#Main Class
mainclass=com.tmax.tibero.gateway.main.GatewayMain
configfile=./jgw.cfg

if [[ $# -gt 0 ]] && [[ $1 = "-v" ]] ; then
    java -jar $gateway
else
# jar 경로에 $mysqljdbc 추가
    java -Xms128m -Xmx512m -Dlog4j.configuration=$log4jfile -classpath $commonsdaemon:$commonspool:$commonscollections:$log4j:$gateway:$msjdbc:$mysqljdbc:$asejdbc:$postgresqljdbc:. $mainclass CONFIG=$configfile $* &
    sleep 1
fi
```

### tbgw 실행
tbgw 실행 후 정상 가동되었는지 확인한다.
```bash
sh ~/gwMySQL/tbJavaGW/tbgw
```
```
-------------------------------
 Name : TmaxData JAVA GATEWAY
 Database: 8
 Port : 8083
-------------------------------
```

8083 포트가 열려있는지 확인한다.
``` bash
netstat -nltp | grep 8083
```
```
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp6       0      0 :::8083                 :::*                    LISTEN      93366/java          
```

게이트웨이 프로세스가 실행 중인지 확인한다.
```bash
ps -ef | grep com.tmax.tibero.gateway
```
```
tibero     93366       1  0 09:51 pts/3    00:00:00 java -Xms128m -Xmx512m -Dlog4j.configuration=jgwlog.properties -classpath ./lib/commons-daemon-1.0.6.jar:./lib/commons-pool.jar:./lib/commons-collections.jar:./lib/log4j-1.2.15.jar:./lib/tbgateway.jar:./lib/sqljdbc.jar:./lib/sqljdbc4.jar:./lib/jconn3.jar:./lib/postgresql-8.4-701.jdbc3.jar:. com.tmax.tibero.gateway.main.GatewayMain CONFIG=./jgw.cfg

tibero     94647   92961  0 10:04 pts/3    00:00:00 grep --color=auto com.tmax.tibero.gateway
```

### Tibero의 tbdsn.tbr 파일 수정
Tibero 접속 정보를 저장하고 있는 tbdsn.tbr 파일을 수정하여 gateway와 mysql의 접속 정보를 입력한다.
```
vi $TB_HOME/client/config/tbdsn.tbr
```
```
mysql_link_remote=(
                 (GATEWAY=(LISTENER=(HOST=192.168.17.25)
                                    (PORT=8083))
                          (TARGET=192.168.17.25:3306:mysql8)
                          (TX_MODE=LOCAL))
)
```
- GATEWAY=(LISTENER...) 부분은 게이트웨이의 IP와 포트 번호를 기입한다.
- TARGET에는 MySQL의 IP와 PORT, 접속할 Database명을 입력한다.


## 접속 테스트
### Tibero Java Gateway를 통한 MySQL 접속 테스트
```bash
tbsql "TIBERO"/"TiberoTmax"@mysql_link_remote
```
```
tbSQL 6  

TmaxData Corporation Copyright (c) 2008-. All rights reserved.

Connected to JDBC3.0 GATEWAY using mysql_link_remote.
```

### DBLINK 테스트
1. DBLINK 생성
```bash
tbsql tibero/tmax
```
```sql
drop database link mylink;

create database link mylink connect to 'TIBERO' identified by 'TiberoTmax' using 'mysql_link_remote';
```

2. MySQL 테이블 생성 및 데이터 추가
```shell
mysql -u TIBERO -p
```
```mysql
use mysql8;
create table sample(id INTEGER, name VARCHAR(100));
insert into sample (id,name) values (1,'KIM');
```

3. DBLINK로 테이블 조회
```sql
select * from "sample"@mylink;
```
```
        id
----------
name
--------------------------------------------------------------------------------
         1
KIM
```