# 목표
각각의 서비스에 필요한 데이터들을 설계하고 더미 데이터를 삽입하여 의미있는 DB 연습을 할 수 있는 환경을 구축한다. 

# 구축 환경
- MySQL (8.0.31)
- Window 11

# 서비스
## 구조
채팅 서비스를 베이스로 하는 하나의 유저 풀이 여러 서비스에서 인증하며 사용할 수 있는 형태

## 종류
- 음식점 검색 서비스
- 게시판이 있는 커뮤니티 서비스
- 채팅 서비스

### 채팅 서비스
#### 요구사항
- 모든 서비스의 회원 가입은 채팅 서비스를 통해 이루어짐
- 모든 회원은 자신의 프로필을 가지고 있음(미설정 시 Default 데이터)
- 모든 회원은 친구를 제한 없이 가질 수 있음
- 채팅방에는 n명의 사람이 들어갈수 있음(1< n <= 200)
- 채팅방 탈퇴시 탈퇴 알림이 유저 정보와 함께 채팅방에 기록이 남음
- 특정 시간 안에 한 명의 유저가 연속된 채팅을 보낼 경우 유저 프로필 사진을 첫 채팅에만 보여줌(클라이언트에서 처리)

#### 데이터 모델링
https://www.erdcloud.com/d/nokABASwGgQpJHacd

### 음식점 검색 서비스
#### 요구사항
- 지리좌표계와 투영좌표계 모두 지원하는 좌표 데이터
- 다양한 범위를 설정해 주변 음식점을 검색할 수 있도록 함
- 음식점 카테고리 검색 기능
- 별점 검색 기능


### 게시판 서비스
#### 요구사항
- 본문에 사진과 텍스트를 자유롭게 배치할 수 있다
- 카테고리는 관리자가 추가 삭제 가능
- 자유로운 댓글 대댓글
- 추천 비추천 기능은 한 게시글에 한번만 선택 가능(변경 가능)