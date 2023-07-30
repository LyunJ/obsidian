## v$sysstat
- recursive calls : 유저의 요청에 따른 추가적인 시스템 내부적인 처리 횟수
- db block gets : 캐시 영역에 있는 데이터 블록 읽을 횟수
- physical reads : disk의 데이터 블록 읽은 횟수
- sorts(memory) : 메모리에서 정렬한 횟수
- sorts(disk) : 디스크에서 정렬이 필요한 경우
- sorts(rows) : sort 대상 row 수 


## df, du
- -a : 모든 파일시스템을 출력
- -B : 지정 용량을 블럭단위로 정하여 용량을 출력한다
- -- total : 총합 토탈 라인을 추가하여 출력

## gen_tip.sh
- $TB_SID 수정 시 $TB_SID에 해당하는 tip 파일을 찾게 되는데, 이 때 tip 파일의 이름을 변경하면 실행되지 않는다
- tbdsn 파일의 sid를 수정하지 않으면, 네트워크 연결이 불가능함
- $TB_SID를 수정하지 않고 tip 파일의 db_name 수정 시 mount 모드로 넘어갈 수 없음
- tbdsn.tbr의 sid만 수정할 경우 네트워크 연결이 불가능함
- 모두 수정 후 실행시키면 database directory에 새로운 디렉토리가 생성됨

## foreground, background, nohup
- 문자열을 반복적으로 찍어내는 셸 스크립트
- jobs는 현재 셸 세션에서 실행시킨 백그라운드 작업의 목록이 출력된다.
- 각각의 job에는 번호가 부여되어 있고, 이 번호로 fg작업과 bg 작업을 수행할 수 있다.

## max_session_count
- MAX_SESSION_COUNT는 같은 개수의 스레드의 LIMIT을 설정한다.
- WORKING PROCESS는 10개의 THREAD를 가질 수 있으므로 MAX_SESSION_COUNT / 10의 개수를 가진다
- CTHR은 자기 자신에 속한 워킹 스레드의 상태를 검사하여 현재 유휴한 워킹 스레드에 클라이언트의 접속을 할당한다.

## 한국어 지원 character set
- MSWIN949는 KSC5601을 포함하기 때문에 호환 가능하다
- UTF-8은 유니코드에서 지정한 문자 매핑 규칙을 인코딩한 것
- KSC5601을 MSWIN949로 바꾸면 문제가되지 않지만, 그 반대는 문제가 된다.

## /dev/shm
- shared memory를 파일 시스템을 통해 접근할 수 있도록 한 것
- 일정 크기의 공간을 /dev/shm 공간으로 잡고, 필요할 경우에만 램에서 공간을 할당하여 디스크로 사용
- 사용되지 않는 공간은 일반 램으로서 사용된다.

## swap 공간
- HugePage란 TLB의 Hit율을 높여주기 위해서 페이지 크기를 높여주는 것이다.


## 환경변수 설정
- oracle_owner는 oracle을 설치하는 유저여야하고, dba 그룹에 들어가있어야한다

## 오라클 파일 다운로드
- dedicated server와 shared server를 구성하는 부분이 없어서 따로 설명하겠음
- v$session을 통해서 dedicated 서버인지 아닌지 알 수 있음


## 사전 준비 과정
- locale은 사용자 인터페이스에서 사용되는 언어,지역설정, 출력 형식 등을 정의하는 문자열.
- desktop class는 최소 구성으로 설치가 가능하고, server class는 더 상세한 구성을 할 수 있다.
- sid가 orcl로 나와있지만, 나중에 확인해본 결과 service name이 orcl이고 sid는 환경변수에서 설정한 대로 나오는 것을 확인.
- smon은 오라클 인스턴스를 관리하는 프로세스. 오라클 인스턴스 fail시 인스턴스를 복구하는 역할을 한다.

## tnsnames, listener
- tnsnames.ora는 클라이언트에서 서버로 접속하기 위해 필요한 파일이고, listener.ora는 서버가 요청을 받기 위해 필요한 파일이다.

## 오라클 디렉토리 구조
- oraInventory는 Oracle Software 제품에 관한 정보와 Server에 설치 되어 있는 Oracle_Home의 정보를 가지고 있는 디렉토리다.
- 설치 시 발생하는 로그를 저장하는 logs 디렉토리가 있다.
- GridSetupAction
- GridSetupAction 디렉토리는 인스톨러가 oracle asm 관련 유틸리티인 asm file의 메타데이터를 확인하는 kfod.bin을 실행하면 생성된다.
- 오라클 버그 리포트에 올라와 있는데, 삭제해도 무관하다고 한다.