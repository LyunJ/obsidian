```toc
```


스토어드 프로그램은 스토어드 루틴이라고도 하는데, 스토어드 프로시저와 스토어드 함수, 트리거와 이벤트 등을 모두 아우르는 명칭이다

## 스토어드 프로그램의 장단점
스토어드 프로그램은 절차적인 처리를 제공하긴 하지만 **애플리케이션을 대체할 수 있을지 충분히 고려해 봐야 한다.** 따라서 스토어드 프로그램의 장단점을 알아둘 필요가 있다.

### 장점
- 데이터베이스의 보안 향상
	- MySQL의 스토어드 프로그램은 자체적인 보안 설정 기능을 가지고 있으며, 스토어드 프로그램 단위로 실행 권한을 부여할 수 있다
	- MySQL 서버의 스토어드 프로그램은 입력 값의 유효성을 체크한 후에야 동적인 SQL 문장을 생성하므로 SQL의 문법적인 취약점을 이용한 해킹은 어렵다
- 기능의 추상화
- 네트워크 소요 시간 절감
- 절차적 기능 구현
- 개발 업무의 구분

### 단점
- 낮은 처리 성능
	- MySQL 서버는 스토어드 프로그램과 같은 절차적 코드 처리를 주목적으로 하는 것이 아이어서 스토어드 프로그램의 처리 성능이 다른 프로그램 언어에 비해 상대적으로 떨어진다.
	- 또한 다른 DBMS의 스토어드 프로그램과 달리 MySQL 서버의 스토어드 프로그램은 실행 시마다 스토어드 프로그램의 코드가 파싱돼야 한다.
- 애플리케이션 코드의 조각화

## 스토어드 프로그램의 문법
함수와 같이 스토어드 프로그램도 헤더 부분과 본문 부분으로 나눌 수 있다.
헤더 부분은 정의부라고 하며, 주로 스토어드 프로그램의 이름과 입출력 값을 명시하는 부분이다. 추가로 스토어드 프로그램의 헤더 부분에는 보안이나 스토어드 프로그램의 작동 방식과 관련된 옵션도 명시할 수 있다.
본문 부분은 스토어드 프로그램의 바디라고도 하며, 스토어드 프로그램이 호출됐을 때 실행하는 내용을 작성하는 부분이다.

### 스토어드 프로시저
#### 정의
스토어드 프로시저는 서로 데이터를 주고 받아야 하는 여러 쿼리를 하나의 그룹으로 묶어서 독립적으로 실행하기 위해 사용하는 것이다.

스토어드 프로시저는 반드시 독립적으로 호출돼야 하며, SELECT나 UPDATE 같은 SQL 문장에서 스토어드 프로시저를 참조할 수 없다.


#### 스토어드 프로시저 생성 및 삭제
스토어드 프로시저는 `CREATE PROCEDURE` 명령으로 생성할 수 있다.

```
CREATE PROCEDURE sp_sum(IN param1 INTEGER, IN param2 INTEGER, OUT param3 INTEGER )
BEGIN
	SET param3 = param1 + param2;
END ;;
```
프로시저 이름 : sp_sum
파라미터 : param1, param2, param3
프로시저 본문 : BEGIN ~ END

##### 주의사항
- 스토어드 프로시저는 기본 반환값이 없다.
- 스토어드 프로시저의 각 파라미터는 다음의 3가지 특성 중 하나를 지닌다.
	- `IN` 타입으로 정의된 파라미터는 입력 전용 파라미터를 의미한다.
	- `OUT` 타입으로 정의된 파라미터는 출력 전용 파라미터다. 외부에서 어떤 값을 전달하려는 목적이 아니라 스토어드 프로시저의 실행이 완료되면 외부 호출자로 결과 값을 전달하는 용도로 사용한다.
	- `INOUT` 타입으로 정의된 파라미터는 입력 및 출력 용도로 모두 사용할 수 있다.


##### 구분자
스토어드 프로시저를 포함한 스토어드 프로그램을 사용할 때는 특별히 SQL 문장의 구분자를 변경해야 한다. 스토어드 프로그램은 본문에 여러 ";" 문자를 가지고 있기 때문에 `CREATE PROCEDURE` 명령의 끝을 정확히 판별할 수 있게 별도의 문자열을 구분자로 설정해야 한다.

명령의 끝을 알려주는 종료 문자를 변경하는 명령어는 `DELEMETER`다. 일반적으로 `CREATE`로 스토어드 프로그램을 생성할 때는 ";;" 또는 "//"과 같이 연속된 2개의 문자열을 종료 문자로 설정한다.

```
DELEMETER ;;

CREATE PROCEDURE sp_sum(IN param1 INTEGER, IN param2 INTEGER, OUT param3 INTEGER)
BEGIN
	SET param3 = param2 + param1;
END ;;

-- 스토어드 프로그램의 생성이 완료되면 다시 종료문자를 기분 종료 문자인 ";"로 복구
DELEMETER ;
```

#### 스토어드 프로시저 실행
스토어드 프로시저는 쿼리에 `SELECT`를 사용할 수 없으며 `CALL` 명령어로 실행해야 한다. 스토어드 프로시저를 실행할 때 `IN`타입의 파라미터는 상숫값을 그대로 전달해도 무방하지만 `OUT`이나 `INOUT`타입의 파라미터는 세션 변수를 이용해 값을 주고받아야 한다.

```
SET @result := 0;
CALL sp_sum(1,2,@result);
SELECT @result;
```

#### 스토어드 프로시저의 커서 반환
스토어드 프로그램은 명시적으로 커서를 파라미터로 전달받거나 반환할 수 없지만 스토어드 프로시저 내에서 커서를 오픈하지 않거나 `SELECT`쿼리의 결과 셋을 페치하지 않으면 해당 쿼리의 결과 셋은 클라이언트로 바로 전송된다.

```
CREATE PROCEDURE sp_selectEmployees (IN in_empno INTEGER)
BEGIN
	SELECT * FROM employees WHERE emp_no=in_empno
END ;;
```

별도로 `OUT`변수에 담거나 화면에 출력하는 처리를 하지 않아도 쿼리의 결과가 클라이언트로 전송된다. 이 기능은 JDBC를 이용하는 자바 프로그램에서도 그대로 이용할 수 있으며, 하나의 스토어드 프로시저에서 2개 이상의 결과 셋을 반환할 수도 있다.

**스토어드 프로시저에서 쿼리의 결과 셋을 클라이언트로 전송하는 기능은 스토어드 프로시저의 디버깅 용도로도 자주 사용한다.**

```
CREATE PROCEDURE sp_sum(IN param1 INTEGER, IN param2 INTEGER, OUT param3 INTEGER)
BEGIN
	SELECT '> Stored procedure started.' AS debug_message;
	SELECT CONCAT('> param1 : ',param1) AS debug_message;
	SELECT CONCAT('> param2 : ',param2) AS debug_message;

	SET param3 = param1 + param2;
	SELECT '> Stored procedure completed.' AS debug_message;
END ;;
```

#### 스토어드 프로시저 딕셔너리
MySQL 8.0 이전 버전 까지는 스토어드 프로시저는 `mysql` 데이터베이스의 `proc` 테이블에 저장됐지만 MySQL 8.0 버전 부터는 `information_schema` 데이터베이스의 `routines` 뷰를 통해 스토어드 프로시저의 정보를 조회할 수만 있다.

### 스토어드 함수
#### 정의
스토어드 함수는 하나의 SQL 문장으로 작성이 불가능한 기능을 하나의 SQL 문장으로 구현해야 할 때 사용한다.

SQL 문장과 관계없이 별도로 실행되는 기능이라면 굳이 스토어드 함수를 개발할 필요가 없다. 스토어드 함수의 유일한 장점은 SQL 문장의 일부로 사용할 수 있다는 것이다.

#### 스토어드 함수 생성 및 삭제
스토어드 함수는 `CREATE FUNCTION` 명령으로 생성할 수 있으며, 모든 입력 파라미터는 읽기 전용이라서 `IN`이나 `OUT`, 또는 `INOUT` 같은 형식을 지정할 수 없다. 스토어드 함수는 반드시 정의부에 `RETURNS` 키워드를 이용해 반환되는 값의 타입을 명시해야 한
```
CREATE FUNCTION sf_sum(param1 INTEGER, param2 INTEGER)
	RETURNS INTEGER
BEGIN
	DECLARE param3 INTEGER DEFAULT 0;
	SET param3 = param1 + param2;
	RETURN param3;
END ;;
```

##### 스토어드 함수 바디에서 사용할 수 없는 것
- PREPARE와 EXCUTE 명령을 이용한 프리페어 스테이트먼트를 사용할 수 없다.
- 명시적 또는 묵시적인 ROLLBACK/COMMIT을 유발하는 SQL문장을 사용할 수 없다.
- 재귀호출을 사용할 수 없다.
- 스토어드 함수 내에서 프로시저를 호출할 수 없다.
- 결과 셋을 반환하는 SQL 문장을 사용할 수 없다

#### 스토어드 함수 실행
스토어드 함수는 스토어드 프로시저와 달리 `CALL` 명령으로 실행할 수 없다. 스토어드 함수는 `SELECT` 문장을 사용해 실행한다

### 트리거
#### 정의
트리거는 테이블의 레코드가 저장되거나 변경될 때 미리 정의해둔 작업을 자동으로 실행해주는 스토어드 프로그램이다.

MySQL의 트리거는 테이블 레코드가 `INSERT`, `UPDATE`, `DELETE`될 때 시작되도록 설정할 수 있다.

#### 용도
대표적으로 칼럼의 유효성 체크나 다른 테이블로의 복사나 백업, 계산된 결과를 다른 테이블에 함께 업데이트하는 등의 작업을 위해 트리거를 자주 사용한다.

#### 제한
MySQL의 트리거는 테이블에 대해서만 생성할 수 있다. 특정 테이블에 트리거를 생성하면 해당 테이블에 발생하는 조작에 대해 지정된 시점(실제 그 이벤트의 실행 전후)에 트리거의 루틴을 실행한다.

MySQL 서버가 ROW 포맷읜 바이너리 로그를 이요해서 복제를 하는 경우 트리거는 복제 소스 서버에서만 실행되고 레플리카 서버에서는 별도로 트리거를 기동하지 않는다. 하지만 이미 복제 소스 서버에서 트리거에 의해 `INSERT` 되거나 `UPDATE`, `DELETE`된 데이터는 모두 바이너리 로그에 기록되기 때문에 실제 리플리카 서버에서도 트리거를 실행한 것과 동일한 효과를 낸다.

바이너리 로그 포맷이 `SATATEMENT`인 경우 복제 소스는 트리거에 의한 데이터 변경은 기록하지 않는다. 하지만 레플리카 서버는 트리거를 실행한다

결국 바이너리 로그 포맷과 관계없이 동일한 결과를 만들어내지만 트리거의 실행 위치가 다르다는 차이가 있다.

#### 트리거 생성
트리거는 `CREATE TRIGGER`명령으로 생성한다. 스토어드 프로시저나 함수와는 달리 `BEFORE`나 `AFTER` 키워드와 `INSERT`, `UPDATE`, `DELETE`로 트리거가 실행될 이벤트와 시점을 명시할 수 있다.

예전 MySQL 버전에서는 `FOR EACH STATEMENT` 키워드를 이용해 문장 기반으로 트리거 작동을 구현할 예정이었지만 해당 기능은 구현되지 않았고, 따라서 MySQL 서버의 트리거는 레코드 단위로만 작동하지만 트리거의 정의부에 `FOR EACH ROW` 키워드는 그대로 사용해야한다.

```
CREATE TRIGGER on_delete BEFORE DELETE ON employees
	FOR EACH ROW
BEGIN
	DELETE FROM salaries WHERE emp_no=OLD.emp_no;
END;;
```

#### 트리거 발생 이벤트
| SQL 종류 | 발생 트리거 이벤트("-->"는 발생하는 이벤트의 순서를 의미) |
| :--- | :--- |
| INSERT | BEFORE INSERT --> AFTER INSERT |
| LOAD DATA | BEFORE INSERT --> AFTER INSERT |
| REPLACE | 중복 레코드가 없을 때 : <br> &nbsp; BEFORE INSERT --> AFTER INSERT <br> 중복 레코드가 있을 때 : <br> &nbsp; BEFORE DELETE --> AFTER DELETE --> BEFORE INSERT --> AFTER INSERT |
| INSERT INTO ... <br> ON DUPLICATE SET | 중복이 없을 때: <br> &nbsp; BEFORE INSERT --> AFTER INSERT <br> 중복이 있을 때 : <br> &nbsp; BEFORE UPDATE --> AFTER UPDATE |
| UPDATE | BEFORE UPDATE --> AFTER UPDATE |
| DELETE | BEFORE DELETE --> AFTER DELETE |
| TRUNCATE | 이벤트 발생하지 않음 |
| DROP TABLE | 이벤트 발생하지 않음 |

#### 트리거가 사용하지 못하는 작업
- 트리거는 외래키 관계에 의해 자동으로 변경되는 경우 호출되지 않는다
- 명시적 또는 묵시적인 `ROLLBACK/COMMIT`(트랜잭션을 시작하거나 끝내는 명령문)을 유발하는 SQL 문장을 사용할 수 없다(`ROLLBACK to SAVEPOINT`는 트랜잭션을 끝내지 않기 때문에 허용된다).
- `RETURN`문장을 사용할 수 없으며, 트리거를 종료할 때는 `LEAVE` 명령을 사용한다
- `mysql`과 `information_schema, performance_schema` 데이터 베이스에 존재하는 테이블에 대해서는 트리거를 생성할 수 없다.

#### 트리거 실행
트리거는 스토어드 프로시저나 함수와 같이 작동을 확인하기 위해 명시적으로 실행해 볼 수 있는 방법이 없다. 직접 `INSERT`, `UPDATE`, `DELETE`를 수행해서 작동을 확인해야한다

#### 트리거 딕셔너리
MySQL 8.0 이전까지는 해당 데이터베이스 디렉터리의 `*.TRG` 라는 파일로 기록됐지만 MySQL 8.0 부터는 `mysql` 데이터베이스의 보이지 않는 시스템 테이블로 저장되고, 사용자는 `information_schema` 데이터베이스의 `TRIGGERS` 뷰를 통해 조회만 할 수 있다.

### 이벤트
#### 정의
주어진 특정한 시간에 스토어드 프로그램을 실행할 수 있는 스케줄러 기능을 이벤트라고 한다. 

#### 실행 조건
MySQL 서버의 이벤트는 스케줄링을 전담하는 스레드가 있는데, 이 스레다가 활성화된 경우에만 이벤트가 실행된다.

이벤트 스케줄러 스레드를 기동하려면 MySQL 서버의 설정 파일에서 `event_scheduler`라는 시스템 변수를 `ON`이나 1로 설정해서 활성화해야 한다.

이벤트 스케줄러가 활성화되면 `SHOW PROCESSLIST` 결과에 `event_scheduler` 프로세스가 표시된다.

#### 이벤트 로그
MySQL의 이벤트는 전체 실행 이력을 보관하지 않고, 최근에 실행된 정보를 `information_schema` 데이터베이스의 `EVENTS`뷰에서 확인 할 수 있다. 필요하지 않더라도 이벤트 처리 로직을 통해 별도의 사용자 테이블에 직접 기록해 놓자.

#### 이벤트 생성
##### 일회성 이벤트
```
CREATE EVENT onetime_job
	ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 1 HOUR
DO
	INSERT INTO daily_rank_log VALUES (NOW(), 'Done');
```
단 한 번 실행되는 일회성 이벤트를 등록하려면 `ON SCHEDULE AT` 절을 명시하면 된다. `AT` 절에는 정확한 시각을 명시할 수도 있고 상대적인 시간을 명시할 수도 있다


##### 반복성 이벤트
```
CREATE EVENT daily_ranking
	on schedule every 1 day starts '2020-08-07 01:00:00' ENDS '2021-01-01 00:00:00'
DO
	INSERT INTO daily_rank_log VALUES(NOW(),'Done');
```

`DO` 절에는 단순히 하나의 쿼리나 스토어드 프로시저를 호출하는 명령을 사용하거나 `BEGIN...END`로 구성되는 복합 절을 사용할 수 있다.

또한 이벤트의 반복성 여부와 관계없이 `ON COMPLETION`절을 이용해 완전히 종료된 이벤트를 삭제할지, 그대로 유지할지 선택할 수 있다. 기본적으로 완전히 종료된 이벤트는 자동으로 삭제되지만 `ON COMPLETION PRESERVE`옵션과 함께 이벤트를 등록하면 이벤트 실행이 완료돼도 삭제되지 않는다.

#### 이벤트의 상태
이벤트는 생성할 때 `ENABLE`, `DISABLE`, `DISABLE ON SLAVE`의 3가지 상태로 생성할 수 있다. 이벤트는 기본적으로 생성되면서 복제 소스 서버에서는 `ENABLE`되며, 복제된 레플리카 서버에서는 `SLAVESIDE_SIDABLED` 상태(`DISABLE ON SLAVE` 옵션을 설정한 것처럼)로 생성된다. 복제 소스 서버의 변경사항은 자동으로 레플리카 서버로 복제되기 때문에 레플리카 서버에서 이벤트는 중복해서 실행할 필요는 없다. 다만 레플리카 서버가 소스 서버로 승격되면 수동으로 이벤트의 상태를 `ENABLE` 상태로 변경해야 한다.

#### 이벤트 딕셔너리
```
SELECT * FROM information_schema.EVENTS \G
```
`EVENTS`뷰를 통해 알 수 있는 것
- 반복성인지 일회성인지
- 언제 실행됐는지
- 이벤트 코드
- 마지막 이벤트 등록 일시
- 마지막으로 실행된 일시
- `ORIGINATOR` : MySQL 서버의 `server_id` 시스템 변수값. 이벤트 상태를 자동 설정하기 위해서 관리된다.

### 스토어드 프로그램 본문 작성

#### BEGIN ... END 블록과 트랜잭션
MySQL에서 트랜잭션을 시작하는 명령으로는 다음 두 가지가 있다
- `BEGIN`
- `START TRANSCATION`

하지만 `BEGIN ... END` 블록 내에서의 `BEGIN`은 `BEGIN ... END` 블록의 시작 키워드로 해석하기 때문에 스토어드 프로그램의 본문에서 트랜잭션을 시작할 때는 `START TRANSACTION` 명령을 사용해야한다. 트랜잭션을 종료할 때는 `COMMIT` 이나 `ROLLBACK` 명령을 사용하면 된다. 스토어드 함수나 트리거에서는 트랜잭션을 사용할 수 없다는 점에 주의하자(다른 트랙잭션에 포함되어 실행되기 때문).

##### 스토어드 프로시저와 트랜잭션
```
CREATE PROCEDURE sp_hello ( IN name VARCHAR(50))
BEGIN
	START TRANSACTION;
	INSERT INTO tb_hello VALUES (name, CONCAT('Hello ', name));
	COMMIT;
END ;;
```

스토어드 프로시저 내부에서 트랜잭션을 완료하면 이 스토어드 프로시저를 호출한 애플리케이션이나 SQL 클라이언트 도구에서는 트랜잭션을 조절할 수 없게 된다. 그래서 스토어드 프로시저 내부에서 트랜잭션을 완료할지, 아니면 프로시저를 호출하는 애플리케이션이나 SQL 클라이언트 도구에서 트랜잭션을 완료할지를 명확히 해야 한다.

#### 변수
##### 스토어드 프로그램에선 로컬 변수를 사용하자
스토어드 프로그램의 `BEGIN ... END` 블록 사이에서 사용하는 변수는 사용자 변수와는 다르다. 사실 스토어드 프로그램에서 사용자변수와 로컬 변수는 혼용할 수 있고, 사용자 변수는 적절히 용도에 맞게 사용하면 스토어드 프로그램의 내부와 외부간의 데이터 전달 용도로 사용할 수 있다. 하지만 사용자 변수를 남발하면 외부의 다른 스토어드 프로그램이나 쿼리에 악영항을 미칠 수 있고, 로컬 변수가 사용자 변수도바 빠르게 처리되므로 스토어드 프로그램에선 가능한 한 사용자 변수 대신 로컬 변수를 사용하는 편이 좋다.

##### 로컬 변수를 사용하는 법
```
-- 로컬 변수 정의
-- DEFAULT 정의하지 않으면 NULL
DECLARE v_name VARCHAR(50) DEFAULT 'Matt';
DECLARE v_email VARCHAR(50) DEFAULT 'matt@email.com';

-- 로컬 변수에 값 할당
SET v_name = 'Kim', v_email = 'kim@email.com';

-- SELECT .. INTO 구문을 이용한 값의 할당
SELECT emp_no, first_name, last_name INTO v_empno, v_firstname, v_lastname
FROM employees
WHERE emp_no=10001
```

`SELECT ... INTO` 구문에서 `SELECT`문은 반드시 1개의 레코드를 반환하는 SQL이어야 한다. 레코드가 한 건도 없거나 2건 이상인 경우에는 에러가 발생한다. 정확히 1건의 레코드가 보장되지 않으면 LIMIT 1과 같은 조건을 추가하는 것이 좋다.

##### 동명의 변수의 우선순위
1. `DECLARE`로 정의한 로컬 변수
2. 스토어드 프로그램의 입력 파라미터
3. 테이블의 칼럼

##### 스토어드 프로그램의 변수명 팁
로컬변수, 사용자 변수, 입력 파라미터 등을 구분하기위해 접두사를 붙이는 습관을 들이자.
1. `p_` : 입력 파라미터
2. `v_` : 로컬 변수
3. `@` : 사용자 변수(필수)
4. `@@` : 서버 시스템 변수(필수)

#### 제어문
##### IF ... ELSEIF ... ELSE ... END IF
##### CASE WHEN --- THEN --- ELSE --- END CASE

#### 반복 루프
반복 루프 처리를 위해서는 `LOOP`,`REPEAT`,`WHILE` 구문을 사용할 수 있다. `LOOP` 문은 별도의 반복 조건을 명시하지 못하는 반면 `REPEAT`와 `WHILE`은 반복 조건을 명시할 수 있다. `LOOP` 구문에서 반복 루프를 벗어나려면 `LEAVE` 명령을 사용하면 된다. 그리고 `REPEAT`문은 먼저 본문을 처리하고 그다음 반복 조건을 체크하지만 `WHILE`은 그 반대로 실행된다는 점이 다르다.

##### LOOP
```
CREATE FUNCTION sf_factorial1(p_max INT)
	RETURNS INT 
BEGIN
	DECLARE v_factorial INT DEFAULT 1;
	
	factorial_loop : LOOP
		SET v_factorial = v_factorial * p_max;
		SET p_max = p_max - 1;
		IF p_max<=1 THEN
			LEAVE factorial_loop; -- 반복문 종료
		END IF;
	END LOOP;

	RETURN v_factorial;
END;;
```

##### REPEAT
반복 처리 내용이 최소 한 번은 실행된다
```
CREATE FUNCTION sf_factorial1(p_max INT)
	RETURNS INT 
BEGIN
	DECLARE v_factorial INT DEFAULT 1;

	REPEAT
		SET v_factorial = v_factorial * p_max;
		SET p_max = p_max - 1;
	UNTIL p_max<=1 END REPEAT;

	RETURN v_factorial;
END;;
```

##### WHILE
처음부터 조건이 `FALSE`인 경우 반복 루프의 내용이 한 번도 실행되지 않는다.
```
CREATE FUNCTION sf_factorial1(p_max INT)
	RETURNS INT 
BEGIN
	DECLARE v_factorial INT DEFAULT 1;
	
	WHILE p_max>1 DO
		SET v_factorial = v_factorial * p_max;
		SET p_max = p_max - 1;
	END WHILE;

	RETURN v_factorial;
END;;
```

### 핸들러와 컨디션을 이용한 에러 핸들링
#### 예외 핸들러?
MySQL 매뉴얼에서 "예외 핸들러" 라고 표현하지 않는 이유는 핸들러는 예외 상황뿐만 아니라 거의 모든 SQL 문장의 처리 상태에 대해 핸들러를 등록할 수 있기 때문이다.

#### 핸들러와 컨디션의 정의
핸들러는 이미 정의한 컨디션 또는 사용자가 정의한 컨디션을 어떻게 처리(핸들링) 할지 정의하는 기능이다.

컨디션은 SQL 문장의 처리 상태에 대해 별명을 붙이는 것과 같은 역할을 수행한다. 

컨디션은 꼭 필요한 것은 아니고 스토어드 프로그램의 가독성을 좀 더 높이는 요소로 생각할 수 있다

#### SQLSATATE와 에러 번호(Error No)


##### 에러 메시지 출력값의 의미
```
SELECT * FROM not_found_table;
ERROR 1146 (42502): Table 'test.not_found_table' doesn't exist
```
MySQL 클라이언트 프로그램을 이용해 쿼릴를 실행하면 에러나 경고가 발생했을 때 "ERROR ERROR-NO (SQL-STATE): ERROR-MESSAGE" 같은 형태의 메시지를 확인할 수 있다.

- ERROR-NO : 4자리(현재까지는) 숫자 값으로 구성된 에러 코드로,  MySQL에서만 유효한 에러 식별 번호다. 
- SQL-STATE : 다섯 글자의 알파벳과 숫자(Alpha-Numeric)로 구성되며, 에러뿐만 아니라 여러 가지 상태를 의미하는 코드다. 이 값은 DBMS 종류가 다르더라도  ANSI SQL 표준을 준수하는 DBMS(ODBC, JDBC 포함)에서는 모두 똑같은 값과 의미를 가진다. 즉, 이 값은 표준값이라서 DBMS 벤더에 의존적이지 않다. 대부분의 MySQL 에러 번호(ErrorNo)는 특정 SqlState 값과 매핑돼 있으며, 매핑되지 않는 ErrorNo는 SqlState 값이 "HY000"(General error)으로 설정된다. SqlState 값의 앞 2글자는 다음과 같은 의미를 가지고 있다.
	- "00" : 정상 처리됨(에러 아님)
	- "01" : 경고 메시지(Warning)
	- "02" : Not found(SELECT나 CURSOR에서 결과가 없는 경우에만 사용됨)
	- 그 이외의 값은 DBMS별로 할당된 각자의 에러 케이스를 의미한다
- ERROR-MESSAGE : 포매팅된 텍스트 문장으로, 사람이 읽을 수 있는 형태의 에러 메시지다. 이 정보 또한 DBMS 벤더별로 내용이나 구조가 다르다.

[**ERROR-NO와 SQL-STATE 매핑 목록**](https://dev.mysql.com/doc/mysql-errors/8.0/en/server-error-reference.html)

###### 핸들러에 사용해야할 번호 : ERROR-NO VS SQL-STATE 
스토어드 프로그램에서 핸들러를 정의할 때 에러 번호로 핸들러를 정의할 수도 있다. 하지만 이때 똑같은 원인에 대해 여러개의 에러 번호를 가지는 경우 에러 번호 중 하나라도 빠뜨리면 제대로 에러 핸들링을 못할 수도 있다.

이러한 문제를 해결할 수 있는 방법은 핸들러를 에러 번호로 정의하는 것이 아니라 SQLSTATE로 정의하는 것이다. **여러 에러 번호에 대해 같은 SQLSTATE를 가지는 경우가 많기 때문에 에러 번호 보다는 SQLSTATE를 핸들러에 사용하는 것이 좋다**.

#### 핸들러
##### 문법
```
DECLARE handler_type HANDLER
	FOR condition_value [, condition_value] ... handler_statements
```

- 핸들러 타입(`handler_type`)
	- `CONTINUE` : `handler_statements`를 실행하고 스토어드 프로그램의 마지막 실행 지점으로 다시 돌아가서 나머지 코드를 처리한다.
	- `EXIT` : 정의된 `handler_statements`를 실행한 뒤에 이 핸들러가 정의된 `BEGIN ... END` 블록을 벗어난다. 현재 핸들러가 최상위 `BEGIN ... END` 블록에 정의됐다면 현재 스토어드 프로그램을 벗어나서 종료된다. 스토어드 함수에서 `EXIT` 핸들러가 정의된다면 이 핸들러의 `handler_statements` 부분에 함수의 반환 타입에 맞는 적절한 값을 반환하는 코드가 반드시 포함되어야 한다.

- `condition_value`
	- `SQLSTATE`
	- `SQLWARNING` : `SQLSTATE`값이 "01"로 시작하는 이벤트
	- `NOT FOUND` : `SQLSTATE` 값이 "02"로 시작하는 이벤트
	- `SQLEXCEPTION` : 경고, NOT FOUND, "00"(정상 처리)으로 시작하는 `SQLSTATE` 이외의 모든 케이스
	- MySQL의 에러코드를 직접 명시

- `handler_statements` : 특정 이벤트가 발생했을 때 그 이벤트에 대한 처리 코드를 정의

##### 예시
```
DECLARE CONTINUE HANDLER FOR SQLEXCEPTION SET error_flag=1;
```

```
DECLARE EXIT HANDLER FOR SQLEXCEPTION
	BEGIN
		ROLLBACK;
		SELECT 'Error occurred - terminating';
	END;;
```

```
DECLARE CONTINUE HANDLER FOR 1022, 1062 SELECT 'Duplicate key in index';
```

```
DECLARE CONTINUE HANDLER FOR SQLSTATE '23000' SELECT 'Duplicate key in index';
```

```
DECLARE CONTINUE HANDLER FOR NOT FOUND SET process_done=1;
```

```
DECLARE EXIT HANDLER FOR SQLWARNING, SQLEXCEPTION
	BEGIN
		ROLLBACK;
		SELECT 'Process terminated, Because error';
		SHOW ERRORS;
		SHOW WARNINGS;
	END ;
```

#### 컨디션
##### 정의
MySQL의 핸들러는 어떤 조건(이벤트)이 발생했을 때 실행할지를 명시하는 여러 가지 방법이 있는데, 그중 하나가 컨디션이다. 각 에러 번호나 `SQLSTATE`가 어떤 의미인지 예측할 수 있는 이름을 반들어 두면 훨씬 더 쉽게 코드를 이해할 수 있을 것이다. 

이러한 조건의 이름을 등록하는 것이 컨디션이다.

#### 문법
```
DECLARE condition_name CONDITION FOR condition_value
```

`condition_value`는 다음 2가지 방법으로 정의한다.
- 에러번호로 정의할 때는 여러개 동시에 명시할 수 없는 것에 유의
- `SQLSTATE`를 명시하는 경우, `SQLSTATE` 키워드를 입력하고 그 뒤에 `SQLSTATE` 값을 입력하면 된다


#### 컨디션을 사용하는 핸들러 정의
```
CREATE FUNCTION sf_testfunc()
	RETURNS BIGINT
BEGIN
	DECLARE dup_key CONDITION FOR 1062;
	DECLARE EXIT HANDLER FOR dup_key
		BEGIN
			RETURN -1;
		END;

	INSERT INTO tb_test VALUES (1);
	RETURN 1;
END ;;
```


### 시그널을 이용한 예외 발생
#### 시그널
예외나 에러에 대한 핸들링이 있다면 반대로 예외를 사용자가 직접 발생시킬 수 있는 기능이 있어야 할 것이다.

MySQL의 스토어드 프로그램에서 사용자가 직접 예외나 에러를 발생시키려면 시그널(SIGNAL) 명령을 사용해야한다.

#### 스토어드 프로그램의 BEGIN ... END 블록에서 SIGNAL 사용
```
CREATE FUNCTION sf_divide (p_dividend INT, p_divisor INT)
	RETURNS INT
BEGIN
	DECLARE null_divisor CONDITION FOR SQLSTATE '45000';

	IF p_divisor IS NULL THEN
		SIGNAL null_divisor
			SET MESSAGE_TEXT='Divisor can not be null', MYSQL_ERRNO=9999;
	ELSEIF p_divisor=0 THEN
		SIGNAL SQLSTATE '45000'
			SET MESSAGE_TEXT='Dividend is null, so regarding dividend as 0', MYSQL_ERRNO=9997;

		RETURN 0;
	END IF;

	RETURN FLOOR(p_dividend / p_divisor);
END ;;
```

##### SQLSTATE의 종류
| SQLSTATE 클래스 | 의미 |
| :-- | :-- |
| "00" | 정상 처리됨(Success) |
| "01" | 처리 중 경고 발생(Warning) |
| 그 밖의 값 | 처리 중 오류 발생(Error) |

##### SQLSTATE 종류에 따른 SIGNAL 처리
"00"으로 시작하는 `SQLSTATE` 값은 정상 처리를 의미하기 때문에 `SIGNAL` 명령문을 "00"으로 시작되는 `SQLSTATE`와 연결해서는 안 된다.

"01"으로 시작하는 `SQLSTATE`값은 경고를 의미하므로 "01"로 연결된 `SIGNAL`은 스토어드 프로그램을 종료시키지 않는다. 다만 스토어드 프로그램이 종료된 이후 경고 메시지가 출력될 것이다.

그 밖의 모든 `SQLSTATE` 값은 에러로 간주해서 즉시 처리를 종료하고 에러 메시지와 에러 코드를 호출자에게 전달한다. 일반적으로 사용하는 `SIGNAL` 명령은 대부분 유저 에러나 예외일 것이므로 그에 해당하는 "45"로 시작하는 `SQLSTATE`를 사용할 것을 권장한다.

#### 핸들러 코드에서 SIGNAL 사용
```
CREATE PROCEDURE sp_remove_user (IN p_userid INT)
BEGIN
	DECLARE v_affectedrowcount INT DEFAULT 0;
	DECLARE EXIT HANDLER FOR SQLEXCEPTION
		BEGIN
			SIGNAL SQLSTATE '45000'
			SET MESSAGE_TEXT='Can not remove user information', MYSQL_ERRNO=9999;
		END;

	-- // 사용자의 정보를 삭제
	DELETE FROM tb_user WHERE user_id=p_userid;
	-- // 위에서 실행된 DELETE 쿼리로 삭제된 레코드 건수를 확인
	SELECT ROW_COUNT() INTO v_affectedrowcount;
	-- // 삭제된 레코드 건수가 1건이 아닌 경우에는 에러 발생
	IF v_affectedrowcount<>1 THEN
		SIGNAL SQLSTATE '45000';
	END IF;
END ;;
```

### 커서
#### 정의
스토어드 프로그램의 커서(CURSOR)는 JDBC 프로그램에서 자주 사용하는 결과 셋(ResultSet)이다.

#### 제약
- 스토어드 프로그램의 커서는 전 방향(전진) 읽기만 가능하다.
- 스토어드 프로그램에서는 커서의 칼럼을 바로 업데이트하는 것(Updatable ResultSet)이 불가능하다.

##### 센서티브 커서와 인센서티브 커서
- 센서티브(Sensitive) 커서 : 일치하는 레코드에 대한 정보를 실제 레코드의 포인터만으로 유지하는 형태. 
	- 센서티브 커서는 커서를 이용해 칼럼의 데이터를 변경하거나 삭제하는 것이 가능하다. 
	- 또한 칼럼의 값이 변경돼서 커서를 생성한 `SELECT` 쿼리의 조건에 더는 일치하지 않거나 레코드가 삭제되면 커서에서도 즉시 반영된다.
	- 센서티브 커서는 별도로 임시 테이블로 레코드를 복사하지 않기 때문에 커서의 오픈이 빠르다.
- 인센서티브(Insensitive) 커서 : 일치하는 레코드를 별도의 임시 테이블로 복사해서 가지고 있는 형태.
	- 인센서티브 커서는 `SELECT` 쿼리에 부합되는 결과를 우선적으로 임시 테이블로 복사해야 하기 때문에 느리다.
	- 이미 임시 테이블로 복사된 데이터를 조회하는 것이라서 커서를 통해 칼럼의 값을 변경하거나 레코드를 삭제하는 작업이 불가능하다.
	- 다른 트랜잭션과의 충돌은 발생하지 않는다.


##### 어센서티브(Asensitive)
센서티브 커서와 인센서티브 커서를 혼용해서 사용하는 방식을 어센서티브라고 하는데, MySQL의 스토어드 프로그램에서 정의되는 커서는 어센서티브에 속한다.

그래서 MySQL의 커서는 데이터가 임시 테이블로 복사될 수도 있고, 아닐 수도 있다. 하지만 만들어진 커서가 센서티브인지 인센서티브인지 알 수 없으며, 결론적으로 커서를 통해 칼럼을 삭제하거나 변경하는 것이 불가능하다.

##### 커서 사용방법
`SELECT` 쿼리 문장으로 커서를 정의하고, 정의된 커서를 오픈(OPEN)하면 실제로 쿼리가 MySQL 서버에서 실행되고 결과를 가져온다.

이렇게 오픈된 커서는 페치(FETCH) 명령으로 레코드 단위로 읽어서 사용할 수 있다.

사용이 완료된 후에 `CLOSE` 명령으로 커서를 닫으면 관련 자원이 모두 해제된다

##### 예시
```
CREATE FUNCTION sf_emp_count(p_dept_no VARCHAR(10))
	RETURNS BIGINT
BEGIN
	DECLARE v_total_count INT DEFAULT 0;
	DECLARE v_no_more_date TINYINT DEFAULT 0;
	DECLARE v_emp_no INTEGER;
	DECLARE v_from_date DATE;
	DECLARE v_emp_list CURSOR FOR

	SELECT emp_no, from_date FROM dept_emp  WHERE dept_no=p_dept_no;
	DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_no_more_data = 1;

	OPEN v_emp_list;
	REPEAT
		FETCH v_emp_list INTO v_emp_no, v_from_date;
		IF v_emp_no > 2000 THEN
			SET v_total_count = v_total_count + 1;
		END IF;
	UNTIL v_no_more_data END REPEAT;

	CLOSE v_emp_list;

	RETURN v_total_count;
END ;;
```

##### DECLARE 명령으로 CONDITION이나 HANDLER, CURSOR를 정의하는 순서
1. 로컬 변수와 CONDITION
2. CURSOR
3. HANDLER

## 스토어드 프로그램의 보안 옵션
### DEFINER와 SQL SECURITY 옵션
#### DEFINER와 SQL SECURITY 옵션
- `DEFINER` : 스토어드 프로그램이 기본적으로 가지는 옵션으로, 해당 스토어드 프로그램의 소유권과 같은 의미를 지닌다. 또한 `SQL SECURITY` 옵션에 설정된 값에 따라 조금씩은 다르지만 스토어드 프로그램이 실행될 때의 권한으로 사용되기도 한다.
- `SQL SECURITY` : 스토어드 프로그램을 실행할 때 누구의 권한으로 실행할지 결정하는 옵션이다. `INVOKER` 또는 `DEFINER` 둘 중하나로 선택할 수 있다. `DEFINER`는 스토어드 프로그램을 생성한 사용자를 의미하며, `INVOKER`는 그 스토어드 프로그램을 호출한 사용자를 의미한다.

##### SQL SECURITY 옵션에 따른 권한
`SQL SECURITY` 옵션이 `DEFINER`일 경우 스토어드 프로그램을 생성한 유저의 권한으로 스토어드 프로그램이 실행된다. 이 때 실행하는 유저가 스토어드 프로그램에서 원하는 권한을 갖지 않아도 스토어드 프로그램을 생성한 유저가 그 권한을 갖고 있으면 실행이 가능하다.

`INVOKER`일 경우 실행하는 유저의 권한으로 스토어드 프로그램이 실행된다.

스토어드 프로그램의 `SQL SECURITY`를 `DEFINER`로 설정하는 것은 유닉스 운영체제의 setUIR 같은 기능이라고 이해하면 된다. 유닉스의 setUID가 그러하듯이 MySQL 스토어드 프로그램도 보안 취약점이 될 수도 있으므로 꼭 필요한 용도가 아니라면 `SQL SECURITY`를 `DEFINER` 보다는 `INVOKER`로 설정하는 것이 좋다.

#### DETERMINISTIC과 NOT DETERMINISTIC 옵션
이 옵션들은 스토어드 프로그램의 보안과 관련된 옵션이 아니라 성능과 관련된 옵션이다. 이 두 옵션은 서로 배타적이라서 둘 중 하나를 반드시 선택해야 한다.

- `DETERMINISTIC` : "스토어드 프로그램의 입력이 같다면 시점이나 상황에 관계없이 괄과가 항상 같다(확정적이다)"를 의미하는 키워드다.
- `NOT DETERMINISTIC` : 입력이 같아도 시점에 따라 결과각 달라질 수도 있음을 의미한다.

일회성으로 싱행되는 스토어드 프로시저는 이 옵션의 영향을 거의 받지 않지만, SQL 문장에서 반복적으로 호출될 수 있는 스토어드 함수는 영향을 많이 받으며, 쿼리의 성능을 급격하게 떨어뜨리기도 한다.


##### 예제
```
CREATE FUNCTION sf_getdate1()
	RETURNS DATETIME
	NOT DETERMINISTIC
BEGIN
	RETURN NOW();
END;;

CREATE FUNCTION sf_getdate2()
	RETURNS DATETIME
	DETERMINISTIC
BEGIN
	RETURN NOW();
END ;;
```
위의 두 스토어드 함수의 차이는 `DETERMINISTIC`이냐 `NOT DETERMINISTIC`이냐의 차이뿐이다.


```
-- type : ALL
EXPLAIN SELECT * FROM dept_emp WHERE from_date>sf_getdate1();
-- type : range
EXPLAIN SELECT * FROM dept_emp WHERE from_date>sf_getdate2();
```
두 `SELECT` 쿼리 모두 인덱스 레인지 스캔으로 실행 계획을 수립할 것이라고 생각했는데, `NOT DETERMINISTIC` 옵션으로 정의된 스토어드 함수를 사용하는 쿼리는 풀 테이블 스캔을 사용하고 있다.

`NOT DETERMINISTIC` 옵션의 의미는 앞에서도 언급했지만 입력값이 같아도 호출되는 시점에 따라 값이 달라진다는 사실을 MySQL에 알려주는 것이다.

`DETERMINISTIC`으로 정의된 sf_getdate2() 함수는 쿼리를 실행하기 위해 딱 한 번만 스토어드 함수를 호출하고, 함수의 결괏값을 상수화해서 쿼리를 실행한다

`NOT DETERMINISTIC`으로 정의된 sf_getdate1()함수는 `WHERE` 절이 비교를 수행하는 레코드마다 매번 값이 재평가 돼야 한다. 따라서 `NOT DETERMINISTIC`으로 정의된 스토어드 함수는 절대 상수가 될 수 없다.

더 중요한 점은 이렇게 풀 테이블 스캔을 유도하는 `NOT DETERMINISTIC` 옵션이 스토어드 함수의 디폴트 옵션이라는 것이다.

## 스토어드 프로그램의 참고 및 주의사항
### 한글 처리
스토어드 프로그램의 소스 코드에 한글 문자열 값을 사용해야 한다면 스토어드 프로그램을 생성하는 클라이언트 프로그램이 어떤 문자 집합으로 MySQL 서버에 접속돼 있는지가 중요해진다.

```
-- // 현재 연결된 커넥션과 데이터베이스 서버가 어떤 문자 집합을 사용하는지 확인하는 쿼리
SHOW VARIABLES LIKE 'character%';
```

스토어드 프로그램을 생성하는 데 관여하는 부분은 `character_set_connection`과 `character_set_client` 세션 변수 정도다. 

이 세션 변수는 "latin1"을 기본값으로 갖는데, 이는 영어권 알파벳을 위한 문자 집합이지, 한글을 포함한 아시아권의 멀티 바이트를 사용하는 언어를 위한 문자 집합은 아니다. 따라서 이 상태에서 코드에 한글이 포함된 스토어드 프로그램을 실행하면 한글이 깨질 수도 있다.

다음과 같이 문자 집합 관련 변수를 변경하자
```
SET character_set_client = "utfmb4";
SET character_set_result = "utfmb4";
SET character_set_connection = "utfmb4";

-- // 재접속시 효과가 사라짐
SET NAMES utfmb4;
```

그리고 스토어드 프로시저나 함수에서 값을 넘겨받을 때도 다음 예제와 같이 넘겨받는 값에 대해 문자 집합을 별도로 지정해 이러한 문제를 피해야 한다는 점도 기억해두자.
```
CREATE FUNCTION sf_getstring()
	RETURNS VARCHAR(20) CHARACTER SET utfmb4
BEGIN
	RETURN '한글 테스트';
END ;;
```

### 스토어드 프로그램과 세션 변수
`DECLARE`로 스토어드 프로그램의 로컬 변수를 정의할 때는 정확한 타입과 길이를 명시해야 하지만 사용자 변수는 이런 제약이 없다. 그래서 스토어드 프로그램에서 로컬 변수는 전혀 사용하지 않고 사용자 변수만 사용할 때도 있다.

```
CREATE FUNCTION sf_getsum(p_arg1 INT, p_arg2 INT)
	RETURNS INT
BEGIN
	DECLARE v_sum INT DEFAULT 0;
	SET v_sum=p_arg1 + p_arg2;
	SET @v_sum=v_sum;

	RETURN v_sum;
END ;;
```

사용자 변수는 타입을 지정하지 않기 때문에 데이터 타입에 대해 안전하지 않고, 영향을 미치는 범위가 로컬 변수보다 넓기 때문에 의도하지 않게 영향도가 커질 수 있다.

또한 사용자 변수는 적어도 그 커넥션에서는 계속 그 값을 유지한 채 남아 있기 때문에 항상 사용하기 전에 적절한 값으로 초기화하고 사용해야 한다는 것도 주의하자.

### 스토어드 프로시저와 재귀 호출
스토어드 프로그램에서도 재귀 호출을 사용할 수 있는데, 이는 스토어드 프로시저에서만 사용 가능하며, 스토어드 함수와 트리거, 이벤트에서는 사용할 수 없다.

`max_sp_resursion_depth` 시스템 변수로 최대 몇 번까지 재귀 호출을 허용할지를 설정할 수 있다. 이 시스템 변수의 기본값은 0이므로 설정값을 변경하지 않으면 MySQL의 스토어드 프로시저에서 재귀 호출을 사용할 수 없다.

```
CREATE PROCEDURE sp_getfactorial(IN p_max INT, OUT p_sum INT)
BEGIN
	SET max_sp_recursion_depth=50; /* 최대 재귀 호출 횟수는 50회 */
	SET p_sum1;

	IF p_max>1 THEN
		CALL sp_decreaseandmultiply(p_max, p_sum);
	END IF;
END ;;

-- // 최대 재귀 호출 횟수가 기본값인 0이므로 에러 발생
CREATE PROCEDURE sp_decreaseandmultiply(IN p_current INT, INOUT p_sum INT)
BEGIN
	SET p_sum=p_sum * p_current;
	IF p_current>1 THEN
		CALL sp_decreaseandmultiply(p_current-1,p_sum);
	END IF;
END ;;
```

### 중첩된 커서 사용
일반적으로는 하나의 커서를 열고 사용이 끝나면 닫고 다시 새로운 커서를 열어서 사용하는 형태도 많이 사용하지만 중첩된 루프 안에서 두 개의 커서를 동시에 열어서 사용해야 할 때도 있다.

기본적으로 두 개의 커서를 동시에 열어서 사용할 때는 특별히 예외 핸들링에 주의해야 한다.

```
CREATE PROCEDURE sp_updateemployeehiredate()
BEGIN
	DECLARE v_dept_no CHAR(4);
	DECLARE v_emp_no INT;
	DECLARE v_no_more_rows BOOLEAN DEFAULT FALSE;
	DECLARE v_dept_list CURSOR FOR SELECT sept_no FROM deparments;
	DECLARE v_emp_list CURSOR FOR SELECT emp_no FROM dept_emp WHERE dept_no=v_dept_no LIMIT 1;
	DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_no_more_rows := TRUE;

	OPEN v_dept_list;
	LOOP_OUTER: LOOP
		FETCH v_dept_list INTO v_dept_no;
		IF v_no_more_rows THEN
			CLOSE v_dept_list;
			LEAVE loop_outer;
		END IF;

		OPEN v_emp_list;
		LOOP_INNER: LOOP
			FETCH v_emp_list INTO v_emp_no;
			IF v_no_more_rows THEN
			-- // 반드시 no_more_rows를 FALSE로 다시 변경해야 한다.
			-- // 그렇지 않으면 내부 루프 때문에 외부 루프까지 종료돼 버린다.(가독성 떨어짐)
				SET v_no_more_rows := FALSE;
				CLOSE v_emp_list;
				LEAVE loop_inner;
			END IF;
		END LOOP loop_inner;
	END LOOP loop_outer;
END ;;
```
이처럼 반복 루프가 여러 번 중첩되어 커서가 사용될 때는 LOOP_OUTER와 LOOP_INNER를 서로 다른 `BEGIN ... END` 블록으로 구분해서 작성하면 쉽고 깔끔하게 해결할 수 있다. 스토어드 프로시저 코드의 처리 중 발생한 에러나 예외는 항상 가장 가까운 블록에 정의되니 핸들러가 사용되므로 각 반복 루프를 블록으로 해결할 수 있는 것이다.


```
CREATE PROCEDURE sp_updateemployeehiredate()
BEGIN
	DECLARE v_dept_no CHAR(4);
	DECLARE v_emp_no INT;
	DECLARE v_no_more_rows BOOLEAN DEFAULT FALSE;
	DECLARE v_dept_list CURSOR FOR SELECT sept_no FROM deparments;
	DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_no_more_rows := TRUE;

	OPEN v_dept_list;
	LOOP_OUTER: LOOP
		FETCH v_dept_list INTO v_dept_no;
		IF v_no_more_rows THEN
			CLOSE v_dept_list;
			LEAVE loop_outer;
		END IF;

		BLOCK_INNER: BEGIN -- // 내부 프로시저 블록 시작
			DECLARE v_emp_no INT;
			DECLARE v_no_more_employees BOOLEAN DEFAULT FALSE;
			DECLARE v_emp_list CURSOR FOR SELECT emp_no FROM dept_emp WHERE dept_no=v_dept_no LIMIT 1;
			DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_no_more_employees := TRUE;
			OPEN v_emp_list;
			LOOP_INNER: LOOP
				FETCH v_emp_list INTO v_emp_no;
				-- // 레코드를 모두 읽었으면 커서 종료 및 내부 루프를 종료
				IF v_no_more_rows THEN
					CLOSE v_emp_list;
					LEAVE loop_inner;
				END IF;
			END LOOP loop_inner;
		END block_inner;
	END LOOP loop_outer;
END ;;
```
각 커서에 대한 핸들러 처리가 보완된 스토어드 프로시저.