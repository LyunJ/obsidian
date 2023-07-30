디스크에 새로운 데이터를 쓰는 역할을 하는 세 가지 background log thread가 있다.

1. log writer - log buffer나 write-ahead buffer에서 OS buffer로 데이터 쓰기
2. log flusher - OS buffer에서 disk로 쓰기(fsyncs)
3. log write_notifier - disk에의 쓰기가 완료됐을 때 유저 스레드에 알려줌(wrtie_lsn이 진행됐을 때)
4. log flush_notifier - 유저 스레드에 완료된 fsyncs에 대한 알림(flushed_to_disk_lsn이 진행됐을 때)

체크포인트에 대한 역할을 하는 두 가지 background log thread가 있다(로그 파일의 공간을 회수)
1. Log checkpointer - subsect_redo_log_available_for_checkpoint_lsn을 결정하고 체크포인트를 씀

## Thread: log writer

이 스레드는 log buffer에서 디스크로 데이터를 쓰는 역할을 한다. 하지만 이 스레드는 fsync() call을 하지 않는다. 데이터를 시스템 버퍼에 복사한다. fsync()를 하는 것은 log flusher thread이다.

log writer thread에서 해결해야 하는 사항은 다음과 같다:
1. 다수의 유저에 의해 병렬적으로 채워진 log buffer에 있는 준비된 데이터가 얼마나 있는지 확인한다.
   
   log recent written buffer 안에는 log buffer에 완료된 모든 쓰기에 대한 링크를 유저 스레드에서 세팅을 해 놓았다. 각각의 그러한 링크는 strat_lsn부터 쓰여진 byte 수로 표현된다. 링크는 쓰기 작업의 start_lsn에 할당된 slot에 저장되어있다.
   
   log writer thread는 recent written buffer에 있는 링크를 추적하고 링크에 의해 생성된 연결된 path를 지나간다. 이 스레드는 나가는 링크가 유실되면 정지한다. 이러한 경우에는 로그 버퍼의 다음 fragment가 쓰여지고 있을 것이다(혹은 할당된 최대 lsn에 도달했을 것이다).
   
   또한 4kB 이상을 통과하는 즉시 중지되며, 이 경우 다음 쓰기 작업을 하는데 충분하다(우리가 log writer thread 내부에서 fsync를 다시 하겠다고 결정하지 않는다면). 링크를 통과하고 링크(recent written buffer 내부의)에 의해 점유된 slot을 비우고난 후, log writer thread는 log.buf_ready_for_write_lsn을 업데이트 한다.   
   
   ![[Pasted image 20230205135850.png]]
   **Example of links in the recent written buffer**
   
   > log buffer는 log.buf_ready_for_write_lsn까지 빈 공간이 없다(이보다 더 작은 lsn에 대한 병렬적인 쓰기는 모두 완료되었다).
   
   만약 순회할 링크가 더이상 없을 경우, log.buf_ready_for_write_lsn이 진행되지 않았으므로 log writer thread는 대기해야만한다. 그러한 경우에는 먼저 spin delay를 사용하고 그런 다음에는 writer_event의 발생을 기다리도록 대기전환한다.
2. 쓰기 위한 log block들을 준비 - 준비할 log block의 header와 footer를 업데이트한다.
   
   log writer thread는 log buffer 내부의 완료된 log block를 감지한다. 그러한 log block들은 더이상 쓰기를 받지 않을 것이다. 그러므로 그들의 header와 footer는 아주 쉽게 업데이트 될것이다(예를들어 체크섬이 계산된 경우).
   
   ![[Pasted image 20230205142620.png]]
   
   만약 완료된 블록이 하나라도 감지된다면 header와 footer를 업데이트한 이후에 log buffer로부터 직접적으로 disk에 쓰여질 것이다. 그 후 log writer thread는 다음 결정을 하기 전에 이전 단계를 재시도 한다. 각각의 쓰기가 하나 또는 하나 이상의 완료된 블록으로 구성되어 있으면 MONITOR_LOG_FULL_BLOCK_WRITES는 하나씩 증가된다.
   
   > 특수한 케이스가 있다 - write-ahead가 요구된다면, 데이터는 write-ahead buffer에 복사되고 완료되지 않은 마지막 블록 또한 복사되고 쓰여질 수 있다.
   
   특수한 케이스는 완료되지 않은 마지막 블록이다. log.buf_ready_for_write_lsn은 그러한 블록의 중간에 위치할 수 있음을 떠올려라. 그러한 경우, 로그 블록에 다음 쓰기작업이 들어올 가능성이 높다.
   
   ![[Pasted image 20230205144112.png]]
   
   성능상의 이유로 우리는 가끔씩 완료되지 않은 마지막 블록을 작성해야 하는 경우가 생긴다. 왜냐하면 우리가 유저 스레드를 가능한 빠르게 회수함으로써 다음 트랜잭션을 처리하고 다음 데이터를 제공할 수 있도록 노력해야한다는 것이 밝혀졌기 때문이다.
   
   그러한 경우:
   
   - 마지막 로그 블록은 log.buf_ready_for_write_lsn까지 전용 버퍼에 복사된다.
   - 전용 버퍼의 블록의 남은 부분은 0x00으로 채워진다.
   - header필드가 업데이트된다
   - 체크섬이 계산되고 블록의 footer에 저장된다.
   - 블록은 전용 버퍼로부터 쓰여진다
   - MONITOR_LOG_PARTIAL_BLOCK_WRITES는 하나씩 증가한다
     
   > write-ahead buffer는 완료되지 않은 마지막 블록의 쓰기를 위한 전용 버퍼로서 사용된다. 왜냐하면 우리가 다음 write-ahead가 필요할 때마다, write-ahead를 하는 동안 완료되지 않은 마지막 블록 또한 쓸 수 있기 때문이다. 전체 또는 부분적인 블록 쓰기에 대한 모니터 카운터는 writing ahead과 관련된 로직이 적용되기 전에 증가한다. 그러므로 부분적인 블록 쓰기의 카운터는 전체 블록 쓰기가 가능하면 증가되지 않는다(이 경우 write-ahead 요구사항만 불완전한 블록을 작성하는 이유가 될 수 있다).
   
   주의:
   log writer thread는 first_rec_group 필드를 업데이트하지 않는다. first_rec_group은 유저 스레드에 의해 블록이 쓰여지도록 허가받기 전에 세팅된다. 왜냐하면 오직 유저 스레드만이 log record 그룹의 바운더리를 알기 때문이다. first_rec_group으로 포인팅되어야할 쓰여질 데이터의 마지막 lsn을 가지고 있는 유저 스레드는 first_rec_group 필드를 세팅할 책임이 있다. log block의 정확히 마지막까지 쓴 유저 스레드는 다음 log block의 header 뒤의 lsn에서 끝나는 것으로 간주된다. 왜냐하면 그러한 쓰기 작업 후에는 log writer는 다음 빈 log block을 쓰는 것이 가능해 지기 때문이다(buf_ready_for_write_lsn은 그런 lsn을 가리킨다). first_rec_group 필드는 링크가 log recent written buffer에 추가되기 전에 업데이트 된다.
   
3. read-on-write 이슈를 피해라
   log writer thread는 또한 read-on-write problem을 피하기 위해 writing ahead에 대한 책임이 있다. 이것은 write ahead가 완료된 포인트까지 추적한다. 
   쓰기가 더 진행되는 경우 :
   - 만약 우리가 단일 write-ahead 구역의 크기이상을 쓰려고 한다면, 완료된 write-ahead 영역으로 제한하고 나중에 마지막 fragment를 쓰는 것을 연기한다(첫번째 단계를 재시도하며 buf_ready_for_write_lsn을 업데이트한다).
	> 만약 완료된 write-ahead 구역의 크기만큼 쓰고 싶다면, 그들은 log buffer에 준비되어 있고 그곳에서 직접적으로 쓰여질 수 있다. 그러한 쓰기는 read-on-write problem을 발생시키지 않을 것이다. 왜냐하면 쓰기의 크기는 write-ahead 영역별로 구분할 수 있기 때문이다.
	
 - 그렇지 않으면, 우리는 데이터를 특별한 write-ahead 버퍼에 복사하는데, 우리는 이 버퍼에  single write-ahead 의 전체 영역의 사이즈를 안전하게 쓸 수 있다. 데이터를 복사한 이후에, write-ahead 버퍼는 0x00 바이트로 완료된다.
> write-ahead buffer는 또한 불완전한 마지막 로그 블록을 복사할 때 쓰여진다.


4. write_lsn을 업데이트 한다.
   
   단일 쓰기를 한 다음(single log_data_blocks_write()), log writer thread는 log.write_lsn을 업데이트하고 메인 루프로 돌아간다. 쓰기 작업에 상당한 시간이 걸릴 수 있기 때문에 그 동안 더 많은 데이터가 준비될 수 있기 때문이다.
   
   그래서 log_data_blocks_write()를 한 뒤의 일반적인 규칙이 다음 호출 내에 얼마나 쓸지에 대한 다음 결정을 내리기 전에 log.buf_ready_for_write_lsn을 업데이트 해야한다이다.
   
5. log writer_notifier_event에 os_event_set을 사용하여 log writer_notifier thread에 알린다.
6. flusher_event에 os_event_set()을 사용하여 log flusher threa에 알린다.

## Thread: log flusher
log flusher thread는 로그 파일의 fsync()를 실행시키는 역할을 한다

fsync() call이 완료되면 log flusher thread는 log.flushed_to_disk_lsn을 업데이트하고 flush_notifier_event의 os_event의 os_event_set을 사용하여 log flush_notifier thread에 알린다.

주의
작은 최적화가 적용되었다 - 이전 플러시 이후 단일 로그 불록만 플러시된 경우, log flusher thread는 유저 스레드에 바로 알려주게 된다(log flush_notifier thread를 거치지 않고). 최적화의 효과는 몇몇 경우는 긍정적이지만 또 다른 경우는 부정적이기 때문에 계속 지켜봐야한다. 하지만, 논리적으로 봤을 땐 합리적이기 때문에 보존되었다.

만약 log flusher thread가 어떤 조건도 만족시키지 않음을 감지하면, 이 스레드는 단순히 기다리거나 재검사를 한다. initial spin delay 이후에 flusher_event를 기다린다.

## Thread: log flush notifier
log flush_notifier thread는 log.flused_to_disk_lsn >= lsn을 만족시키는 것을 대기하는 모든 유저 스레드에게 조건이 만족되었을 때 알려주는 역할을 한다.

주의:
	log flush-notifier thread는 만족할 가능성이 매우 높은 시기 또한 알려준다(lsn 값이 같은 log block에 있을 때). 이는 실수를 유발할 수 있고 flushed_to_disk_lsn이 충분히 진행됬음을 알림을 받은 유저 스레드가 직접 보장해야한다.

log flush_notifier thread는 flush_notifier_event에서 os_event_wait_time_low()를 사용하여 반복 중에 증가하는 flushed_to_disk_lsn이 충분히 진행됨을 기다린다. log flush_notifier thread는 log flusher에 의해 알림을 받았을 때 flushed_to_disk_lsn이 진행되었음(새로운 single byte면 충분)을 보장한다.

이는 다음 두 항목 사이의 모든 이벤트에서 대기 중인 유저 스레드에 알린다
- flushed_to_disk_lsn 이전 값이 포함된 블록의 이벤트
- flushed_to_disk_lsn의 새로운 값을 포함하는 블록의 이벤트

이벤트는 매핑을 사용하여 원형 이벤트 배열의 블록별로 할당된다:
```
event_slot = (lsn-1) / OS_FILE_LOG_BLOCK_SIZE % S
```
S는 배열의 크기이다(이벤트가 있는 슬롯 수). 각각의 슬롯은 하나의 이벤트를 가지고 있고, 동일한 로그 블록(또는 S * i보다 큰 숫자의 로그 블록) 내에서 최대 lsn까지 플러시를 대기하는 모든 유저 스레드를 그룹화한다.

![[Pasted image 20230207181653.png]]

이벤트의 내부 뮤텍스는 알림을 놓칠 경우를 피하기 위해 사용된다(이는 잘못된 알림보다 위험하다).

하지만, 이벤트 대기에 대한 최대 timeout값이 정의되어있다. timeout에 도달하면(기본값: 1ms), flushed_to_disk_lsn은 유저 스레드에서 다시 체크된다.

> 플러시는 로그 블록의 가운데에 세팅된 log.write_lsn에 대해서 가능하기 때문에 같은 블록에 대한 같은 슬롯에 대해 여러번 알림이 갈 수 있다. 우리는 마지막 블록에 대해 알림을 지연시키는 방법을 시도해봤지만 결과는 더 좋지 않았다. 여기서는 레이턴시가 더 중요하다는 것을 알게 되었다.


## Thread: log checkpointer
log checkpointer thread는 다음과 같은 역할을 한다
1. checkpoint 쓰기가 요구되는지 체크한다(checkpoint age가 너무 커지기 전에 감소시키기 위해)
2. redo log의 공간 또는 가장 오래된 페이지의 age때문에 page cleaner thread에서 더티 페이지의 동기 플러시를 강제로 수행해야하는지 확인한다.
3. 체크포인트 쓰기(log checkpointer thread만이 쓸 수 있음)

이 스레드는 마지막에 도입되었다. 이 스레드는 성능을 위해 요구되지 않지만 다른 로그 스레드를 도입한 후에는 일관성 있는 설계를 할 수 있게 한다. 이는 유저 스레드가 로그파일에 대한 쓰기 작업을 하지 않기 때문이다. 이전에는 유저 스레드간의 동기화가 필요할 때마다 체크포인트를 직접 썼었다.

log checkpointer thread는 다음과 같은 수식으로 log.available_for_checkpoint_lsn을 업데이트한다.
```
min(log.buf_dirty_pages_added_up_to_lsn, max(0, oldest_lsn - L))
```
- oldest_lsn = min(각각의 flush list 에서의 가장 앞에 있는 페이지의 가장 오래된 변경)
- L은 log recent closed buffer의 slot의 개수이다.

특이한 상황은 flush list에 더티 페이지가 없는 상황이다 - 이는 기본적으로 log.buf_dirty_pages_added_up_to_lsn에 세팅되어 있다.

> 이전에는 모든 유저 스레드가 이 lsn을 동시에 계산하려고 시도하여 flush_list 뮤텍스에서 경합을 일으켰다. 이 뮤텍스는 가장 초기에 추가된 페이지의 가장 오래된 변경사항을 읽어야 한다. 이제 lsn은 단일 스레드로 업데이트된다.

## log가 disk에 쓰여질 때까지 대기
유저는 유저에 의해 사용된 lsn >= end_lsn의 log range에 대해 로그 버퍼에서 디스크로 데이터를 log writer thread가 쓸 때까지 대기해야한다.
```
write_lsn >= end_lsn
```

log.write_lsn은 log writer thread에 의해 업데이트된다.

대기상태는 이벤트의 배열을 사용해 해결된다. 주어진 lsn을 기다리는 유저 스레드는 해당 포지션에 발생할 이벤트를 기다린다:
```
slot = (end_lsn - 1) / OS_FILE_LOG_BLOCK_SIZE % S
```
S는 배열 내의 엔트리의 개수를 나타낸다. 따라서 이벤트는 end_lsn을 포함하는 로그 블록에 해당한다.

log writer_notifier thread는 어떻게 log.write_lsn이 진행되는지를 추적하고 유저 스레드에 연속적인 슬롯에 대해 사용자 스레드에 알린다.

주의:
write_lsn이 로그 블록의 중간에 있을 때, 대한 lsn이 있는 블록 전체에 대한 대기 중인 유저 스레드는 알림을 받는다. 유저 스레드가 알림을 받으면, write_lsn의 현재 값이 자신의 lsn값에 대해 만족하는지 확인하고 아니라면 다시 대기상태로 돌아간다. 알림의 유실을 막기 위해 이벤트의 내부 뮤텍스가 사용된다.

## 로그가 디스크에 플러시 될 때까지 대기
만약 유저가 손상(트랜잭션에 대한 commit)에 대한 log의 보존을 보장받고 싶다면, log flusher가 유저에 의해 사용된 lsn >= end_lsn의 log range에 대한 로그파일의 디스크로의 플러시할 때까지 기다려야한다:
```
flushed_to_disk_lsn >= end_lsn
```
log.flushed_to_disk_lsn은 log flusher thread에 의해 업데이트된다.

대기상태는 이벤트 배열에 의해 해결된다. 주어진 lsn에 대해 대기하고 있는 유저 스레드는, 해당 포지션의 이벤트를 사용해 대기한다.
```
slot = (end_lsn - 1) / OS_FILE_LOG_BLOCK_SIZE % S
```
S는 배열내의 엔트리의 개수를 나타낸다. 그러므로 이벤트는 end_lsn을 포함하는 로그 블록에 해당한다.

log flush_notifier thread는 log.flushed_to_disk_lsn이 어떻게 진행되는지 추적하고 유저 스레드에 연속적인 슬롯에 대해 알린다.

주의:
write_lsn이 로그 블록의 중간에 있을 때, 대한 lsn이 있는 블록 전체에 대한 대기 중인 유저 스레드는 알림을 받는다. 유저 스레드가 알림을 받으면, write_lsn의 현재 값이 자신의 lsn값에 대해 만족하는지 확인하고 아니라면 다시 대기상태로 돌아간다. 알림의 유실을 막기 위해 이벤트의 내부 뮤텍스가 사용된다.
