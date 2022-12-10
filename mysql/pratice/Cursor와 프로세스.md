# 개요
Cursor 를 사용하여 프로시저를 개발하던 중 Loop내부에서 Cursor를 열고 닫으면 어떤 일이 벌어지는지 궁금해졌다. Cursor를 닫게 되면 Cursor의 자원이 모두 해제되는 것은 이해하고 있었지만 열고 닫는 속도가 빠르면 자원을 해제하는 오버헤드가 점점 누적될 수도 있을 것 같아 테스트를 해보았다.

## 테스트
### LOOP 밖에서 OPEN 실행
```
DELIMITER ;;  
CREATE PROCEDURE cursor_process(IN sleep_time INTEGER, IN sleep_active_count INTEGER)  
BEGIN  
    DECLARE v_name VARCHAR(10);  
    DECLARE v_email VARCHAR(100);  
    DECLARE v_counter INTEGER DEFAULT 0;  
  
    -- CURSOR 종료 조건 변수 생성  
    DECLARE done BOOLEAN DEFAULT FALSE;  
  
    -- USER 테이블을 조회할 CURSOR 정의 및 조회 완료 후 핸들러 정의  
    DECLARE cursor_select_user CURSOR FOR SELECT `name`,`email` FROM user LIMIT 1000;  
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;  
  
    OPEN cursor_select_user;  
    cursor_loop : LOOP  
        FETCH cursor_select_user INTO v_name,v_email;  
  
        -- SLEEP 함수 실행  
        IF v_counter = sleep_active_count THEN  
            DO sleep(sleep_time);  
        end if;  
  
        -- counter 변수 증가  
        SET v_counter = v_counter + 1;  
        -- LOOP 종료 조건  
        if done = TRUE THEN  
            LEAVE cursor_loop;  
        end if;  
    end loop;  
    CLOSE cursor_select_user;  
end;;  
DELIMITER ;

CALL cursor_process(30,100);
[2022-12-10 21:56:00] completed in 30 s 5 ms
```

다른 세션으로 접속 후
```
SHOW PROCESSLIST;
```
![[Pasted image 20221210232340.png]]

#### 결과
OPEN 명령어가 한 번만 실행되었기 때문에 실행중인 프로세스도 하나인 것을 알 수 있다

### LOOP 안에서 OPEN 실행
```
DELIMITER ;;  
CREATE PROCEDURE cursor_process_2(IN sleep_time INTEGER, IN sleep_active_count INTEGER)  
BEGIN  
    DECLARE v_name VARCHAR(10);  
    DECLARE v_email VARCHAR(100);  
    DECLARE v_counter INTEGER DEFAULT 0;  
  
    -- CURSOR 종료 조건 변수 생성  
    DECLARE done BOOLEAN DEFAULT FALSE;  
  
    -- USER 테이블을 조회할 CURSOR 정의 및 조회 완료 후 핸들러 정의  
    DECLARE cursor_select_user CURSOR FOR SELECT `name`,`email` FROM user ORDER BY RAND() LIMIT 1;  
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;  
  
    -- LOOP 내부에서 CURSOR OPEN    cursor_loop : LOOP  
        OPEN cursor_select_user;  
        FETCH cursor_select_user INTO v_name,v_email;  
        CLOSE cursor_select_user;  
  
        -- SLEEP 함수 실행  
        IF v_counter = sleep_active_count THEN  
            DO sleep(sleep_time);  
        end if;  
        -- LOOP 종료 조건  
        if v_counter=sleep_active_count THEN  
            LEAVE cursor_loop;  
        end if;  
        -- counter 변수 증가  
        SET v_counter = v_counter + 1;  
    end loop;  
end;;  
DELIMITER ;

CALL cursor_process_2(30,100);
[2022-12-10 23:51:46] completed in 1 m 3 s 696 ms
```
랜덤으로 한 명의 유저 정보를 sleep_active_count만큼 FETCH 하는 프로시저

다른 세션으로 접속 후
```
SHOW PROCESSLIST;
```
실행 직후 결과
![[Pasted image 20221210235326.png]]
일정 시간이 지난 후 결과 
![[Pasted image 20221210235347.png]]

#### 결과
실행 직후 프로세스 리스트를 보면 아래에서 세번째 프로세스가 테스트 프로시저를 실행시키는 프로세스임을 알 수 있다. 일정 시간이 지난 후 같은 프로세스가 SLEEP 함수를 실행시키며 30초를 쉬었기 때문에 1분이 넘는 시간동안 프로세스가 실행되었음을 알 수 있다.


### 테스트 결과
LOOP 외부에서 OPEN이 실행 됐을 경우, OPEN은 단 한번 실행된다. OPEN은 빠르게 실행될 것이기 때문에 프로시저를 실행시키는 프로세스와 OPEN이 실행되는 프로세스가 같은 프로세스인지 알 수 없었다. 

두번째 실험 결과, SLEEP이 실행되기 전에 프로시저를 실행시키는 프로세스가 오랫동안 살아있었고, 이후 같은 프로세스에서 SLEEP 함수가 실행되는 것을 알 수 있었다.  즉, 하나의 프로시저에서 여러번 실행되는 OPEN&CLOSE 과정이 순차적으로 완료된 후 SLEEP이 실행되는 것을 알 수 있었다.

정리하자면 프로시저는 단일 프로세스에서 실행되기 때문에 CURSOR를 여러번 OPEN한다면 성능이 낮아지게 된다.