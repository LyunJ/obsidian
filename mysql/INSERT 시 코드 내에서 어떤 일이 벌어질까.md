MySQL은 크게 클라이언트 프로그램과 서버 프로그램으로 나눠져 있다. 클라이언트 프로그램은 사용자의 요청을 받고 API를 사용해 서버에 다시 요청을 보내 원하는 결과를 만들어 낸다. 서버 프로그램은 클라이언트의 요청에 따라 적절한 동작을 통해 원하는 데이터를 클라이언트로 보내게 된다. 

서버에서는 쿼리를 해석해 실행계획을 세우고 스토리지 엔진을 통해 데이터에 접근하게 된다. 쿼리를 해석하는 과정은 다음과 같다:

1. Query parser
2. Preprocessor
3. Query optimizer
4. Query Excution Engine

마지막 Query Excution Engine에서는 Storage Engine에 대한 API call를 담당하고 응답받은 데이터를 처리하는 역할을 한다.

만약 데이터에 대한 변경이나 추가가 이뤄진 경우라면 변경점이 redo log file에 기록된다. redo log file에 기록되는 순서는 다음과 같다:

1. User thread의 변경 요청
2. 데이터를 추가한 경우 log buffer에 데이터 저장 및 log recent written buffer에 링크 저장
3. 기존 데이터에 대한 변경일 경우 flush list에 더티 페이지 저장 및 log recent closed buffer에 링크 저장
4. 백그라운드 로그 스레드가 log buffer의 데이터를 OS buffer로 복사
5. 백그라운드 로그 스레드가 OS buffer의 데이터를 log file로 flush

# 쿼리 해석
## Query parser

