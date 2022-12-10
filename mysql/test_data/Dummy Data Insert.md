# User, User_Profile Table Insert
```
WHILE i <= cnt DO  
    INSERT IGNORE INTO user  
        (id, status, name, email, password)  
            values  
                (i,'ACTIVATE',CONCAT('N',i),CONCAT('email',i,'@email.com'),MD5(CONCAT('PASSWORD',i)));  
  
    INSERT IGNORE INTO user_profile (id, user_id, nick, phone_num, gender, image_url, background_url)  
    values (i, i, CONCAT('NICK', i), CONCAT('010-', IF((i - 10000) <= 0, '0000', LPAD((i - 10000),4,'0')),'-', LPAD((i % 10000),4,'0')), IF((i % 2) = 0, 'M', 'F'), CONCAT('IMAGE_URL_', i), CONCAT('BACKGROUND_URL_', i));  
  
    SET i = i + 1;  
    end while ;
```
## 100만 Row Insert
user, user_profile 각각 100만개의 row를 insert했다. 처음에는 1000만개를 넣으려 했지만 1시간 20분이 지나도 60만개를 겨우 넘는 것을 보고 세션 종료 후 `INSERT IGNORE`로 변경 후 나머지 40만개를 채웠다. 인덱스가 없는 상황에서 총 200만개의 ROW가 삽입되는데 2시간 가까운 시간이 왜 걸렸던 것일까?

### 원인
#### AUTO COMMIT 모드 사용
AUTO COMMIT 모드를 사용하면 프로시저의 쿼리 하나하나에 트랜잭션이 열리고 닫혀 더 많은 시간이 소요되게 된다. 

#### 외래키 제약조건 존재
외래키 제약조건은 데이터간의 관계에서 무결성을 보장해주지만, 데이터를 대량 삽입할 때는 쿼리 성능을 낮추는 요인이 된다. 

#### CONCAT, LPAD등 함수 사용
내부 함수를 사용하여 함수 처리 시간 뿐만 아니라 INTEGER에서 VARCHAR로 형변환하는 오버헤드까지 존재하여 데이터 전처리 시간 또한 늘어났을 것. 

### 의문점
#### 프로시저 중지 후 두 테이블의 ROW 수가 일치하지 않음
반복문이 빠르게 돌면서 트랜잭션이 다수 발생하는데, 옵티마이저가 알아서 USER 쿼리를 먼저 모아서 실행시키고 USER_PROFILE을 실행시키는지 모르겠지만, USER보다 USER_PROFILE의 ROW 수가 적었음.


### 오해
#### AUTO_INCREMENT LOCK 이 영향을 줬나?
AUTO_INCREMENT LOCK은 `innodb_autoinc_lock_mode` 시스템 변수의 영향을 받는다. 하지만 innoDB에서 `innodb_autoinc_lock_mode`는 2이므로 락이 걸리지 않는다. 
[[AUTO_INCREMENT#innodb_autoinc_lock_mode]]

# Chat Table INSERT
```
CREATE PROCEDURE td_chat(IN chat_num INTEGER)  
BEGIN  
    DECLARE v_i INTEGER DEFAULT 0;  
    DECLARE v_chatroom_id INTEGER DEFAULT 1;  
    DECLARE v_user_id INTEGER DEFAULT 1;  
  
    DECLARE v_rand_chatroom_member CURSOR FOR  
    SELECT chatroom_id,user_id FROM chatroom_member ORDER BY RAND() LIMIT 1;  
  
    START TRANSACTION;  
    WHILE v_i < chat_num DO  
        OPEN v_rand_chatroom_member;  
        FETCH v_rand_chatroom_member INTO v_chatroom_id, v_user_id;  
        CLOSE v_rand_chatroom_member;  
  
        INSERT INTO chat (chatroom_id, user_id, content)  
        VALUES (v_chatroom_id, v_user_id, CONCAT('CONTENT',v_i));  
  
        SET v_i = v_i + 1;  
        END WHILE;  
    COMMIT;  
end;
```
## 성능
### 1000 ROW INSERT
```
testdata> CALL td_chat(1000)
[2022-12-10 17:08:37] completed in 17 s 334 ms
```
**1 ROW 당 17ms**

### 1 LOOP 당 1 CURSOR
