# Oracle 설치 준비
## 체크 리스트

| 순서 |       확인할 것       | 확인 여부 |
|:----:|:---------------------:|:---------:|
|  1   | /etc/hosts 파일 설정  |     X     |
|  2   | kernel parameter 수정 |     X     |
|  3   | 필수 패키지 설치  |     X     |
|  4   |      xhost 설정       |     X     |
|  5   |  dba 그룹 유저 생성   |     X     |
|  6   |    환경 변수 설정     |     X     |
|  7   |     runInstaller      |     X     |
|  8   |     listener 설정     |     X     |


## /etc/hosts 파일 설정
/etc/hosts 파일에 oracle 서버의 ip주소와 host를 매핑해놓아야 한다.

오라클은 시스템 함수를 통해 hostname과 ip 주소를 얻는데, 위 작업을 하지 않으면 외부 접속 시 올바른 ip값을 응답하지 않게된다.

## Kernel parameter 수정
[오라클 최소 파라미터](https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/minimum-parameter-settings-for-installation.html#GUID-CDEB89D1-4D48-41D9-9AC2-6AD9B0E944E3)


```
# /etc/sysctl.conf

fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 524288
kernel.shmmax = 2147483648
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
kernel.panic_on_oops = 1
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576

# 이후 sysctl -p 명령어 수행하여 파라미터 적용
```

## 필수 패키지 설치
```
rpm -q binutils compat-libstdc++ gcc glibc libaio libgcc libstdc++ \
make sysstat unixodbc
```

## xhost 설정
### ssh 설정
```
# /etc/ssh/sshd_config
X11Forwarding yes

# bash
systemctl restart sshd
```

### xhost 설치
```
yum install xorg-x11-xauth
yum install xorg-x11-server-Xorg
```

이 후, 윈도우에 xming을 설치한다

### 연결 확인
#### putty
세션을 실행하기 전에, `Connection > SSH > Auth > X11`에서 Enable X11 forwarding 체크 후 X display location을 `localhost:0.0`으로 설정
![[Pasted image 20230816144657.png]]

Putty가 자동으로 연결 설정을 하기 때문에 접속 후 xhost 명령어로 바로 확인이 가능함.



## 유저 및 그룹 생성

```
groupadd -g 1200 dba
useradd -g dba -u 1200 ora19c
passwd ora19c

mkdir -p /app/ora19c
chown -R ora19c.dba /app
chmod -R 755 /app
```

## 환경 변수 설정

```
su ora19c
```

```sh
# ~/.bash_profile

export ORACLE_HOSTNAME=moti
export TMP=/tmp
export TMPDIR=$TMP
export ORACLE_OWNER=ora19c
export ORACLE_BASE=/app/ora19c
export ORACLE_HOME=/app/ora19c/19c
export PATH=$PATH:$ORACLE_HOME/bin:$ORACLE_HOME:/usr/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME/lib:/lib:/usr/lib
export ORACLE_SID=DB19
export NLS_LANG=AMERICAN_AMERICA.KO16MSWIN949
export TNS_ADMIN=$ORACLE_HOME/network/admin

export LANG=en_US.UTF-8
# RHEL 8.0 이상일 경우
export CV_ASSUME_DISTID=RHEL7.6
```


## 서버 구성 체크리스트 확인

| Check                                                | Task                                                         |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| /tmp 디렉토리                                        | 최소 1GB                                                     |
| Ram 크기에 따른 Swap space 할당(Oracle 데이터베이스) | 1~2 GB : 1.5배, 2~16GB : RAM 사이즈와 같게, 16GB 이상 : 16GB |
| RAM 크기에 따른 Swap space 할당(Oracle 재시작)       | 8~16 GB : RAM 크기와 같게, 16GB 이상 : 16GB                  |


