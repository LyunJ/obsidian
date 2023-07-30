mtr이 커밋 될 때, 데이터는 mtr의 내부 버퍼에서 redo log의 버퍼로 이동된다.

더 나은 병렬성을 위해, 로그 버퍼를 쓰는 과정은 다음의 단계를 포함하고 있다.
1. redo 공간 예약
2. 예약된 공간에 데이터 복사
3. 최근에 쓰여진 버퍼에 링크 추가

이후 mtr동안 수정된 페이지는 플러시 목록에 추가해야 한다. 왜냐하면 더티 페이지가 플러시 리스트에 추가되는 순서를 지켜줄 뮤텍스가 더이상 없기 때문이고, 추가적인 매커니즘이 체크포인트를 적절하게 결정해줄 lsn available을 보장해주기 위해 요구된다. 따라서 절차는 다음과 같은 단계로 구성된다.

1. 페이지를 더티 페이지로 마킹
2. 더티 페이지를 플러시 리스트에 추가
3. 최근에 닫힌 로그 버퍼에 링크 추가

## redo 공간 예약
lsn 값의 범위는 제공받은 데이터 바이트 수에 대해 예약된다. 예약된 범위는 로그 버퍼와 로그 파일의 데이터 공간을 직접적으로 지정한다.

lsn 값 범위를 예약하는데 사용되는 절차:
1. redo log에의 공유 엑세스 획득(공유된 rw_lock을 통해)
2. 우리가 써야할 데이터 바이트의 숫자에 의해 예약된 데이터 바이트(log.sn)의 global number를 증가시킨다.

이것은 원자적인 fetch_add 작업에 의해 수행된다.
```
start_sn = log.sn.fetch_add(len)
end_sn = start_sn + len
```

len은 우리가 써야하는 데이터 바이트의 값이다.

sn 값의 범위는 lsn 값의 범위로 해석된다.
```
start_lsn = log_translate_sn_to_lsn(start_sn)
end_lsn =  log_translate_sn_to_lsn(end_sn)
```
요구되는 해석은 간단한 계산에 의해 수행된다.
```
lsn = sn / LOG_BLOCK_DATA_SIZE * OS_FILE_LOG_BLOCK_SIZE + 
	sn % LOG_BLOCK_DATA_SIZE + LOG_BLOCK_HDR_SIZE
```

3. 예약된 범위가 로그 버퍼의 남는 공간에 전달될 때 까지 기다린다.

이 단계에서 우리는 log writer thread를 기다려야만 할 수 있고, 이 스레드는 시스템 버퍼에 데이터를 쓰는 로그 버퍼 안의 공간을 회수한다.

유저 스레드는 log buffer의 빈 공간에 예약된 lsn 값의 범위가 매핑될 때 까지 기다려야한다.
```
end_lsn - log.write_lsn <= log.buf_size
```

주의 :
대기 상태는 log_write_up_to(end_lsn - log.buf_size)를 호출함으로서 수행되고, log_write_up_to는 짧은 sleep을 동반한 반복문을 가지고 있다. 우리는 그 대기 상태가 실제로 필요할 것 같지 않다고 가정한다. MONITOR_LOG_ON_BUFFER_SPACE* 카운터는 대기 루프에서 소비된 반복 횟수를 추적한다. 이 값이 0 근처에 있지 않으면 DBA가 로그 버퍼 크기를 늘려야 한다.

> log writer thread는 write syscall을 기다릴 수 있지만, 이는 또한 더 작은 sn 값에 대한 로그 버퍼에 데이터 쓰기 작업을 완수해야하는 또다른 유저 스레드를 기다려야할 수도 있다. 이러한 사용자 스레드가 예약되지 않았기를 바란다. 만약 우리가 스케쥴링을 통제한다면(예를들면 우리가 fiber-based 접근을 사용했다면), 우리는 그런 문제들을 피할 수 있다.

4. 예약된 범위가 로그 파일의 빈 공간에 전달될 때 까지 기다린다

이 단계에서 우리는 페이지 클리너 스레드를 기다리거나 log checkpointer thread가 다음 체크포인터를 만들때까지 강제적으로 기다려야할 수도 있다.

유저 스레드는 lsn 값의 예약된 범위가 로그 파일의 빈 공간에 매핑될때까지 기다린다
```
end_lsn - log.last_checkpoint_lsn <= redo lsn capacity
```

주의:
대기 상태는 progressive sleep를 동반한 반복문에 의해 수행된다. MONITOR_LOG_ON_FILE_SPACE* 카운터는 대기 루프에서 발생한 반복 횟수를 추적한다. 만약 이 값이 0 근처에 있지 않으면, DBA는 더 많은 page cleaner thread를 사용해야하고, log 파일의 크기를 늘리거나 더 좋은 스토리지 하드웨어를 요청해야한다.

이 매커니즘은 **deadlock**으로 이어질 수 있다. 왜냐하면 mtr 커밋 중에 대기중인 사용자 스레드가 더티 페이지를 잠그면 플러시할 수 없기 때문이다. 만약 이 페이지가 매우 오래된 변경사항을 가지고 있다면, 플러시하지 않고 체크 포인트를 앞으로 이동시키는 것은 불가능할 것이다. 그러한 경우에는 log checkpointer thread가 로그 파일 내부의 공간을 회수할 수 없다.

이러한 문제를 피하기 위해서, 유저 스레드는 log_free_check()를 latch를 유지하지 않을 때 때때로 호출한다. 유저 스레드는 적어도 4개의 변경된 페이지마다 log_free_check()을 호출하고, 만약 로그 파일에 빈 공간이 얼마 없는 것을 파악하면 빈 공간이 회수될 때까지 (또한 래치를 가지고 있지 않아야한다) 대기한다.

> 다수의 유저 스레드는 래치를 가지고 있지 않아도 빈 공간을 체크한다음 쓰기를 진행할 수 있다. 게다가 이 매커니즘은 필요한 최소 여유 공간이 다음과 같은 가정을 기반으로하기 때문에 작동한다. 
- 동시 사용자 스레드의 최대 개수가 제한되었다
- 스레드 내에서 두 체크 간의 쓰기의 최대 크기가 제한된다

이 매커니즘은 concurrency가 제한되어있지 않으면 안전하지 않다. 그러한 경우 우리는 데드락이 발생할 수 있는 환경에서 최선을 다할 뿐이다.

## 예약된 공간에 데이터 복사
lsn 값의 범위가 예약된 이후에, 데이터는 lsn 값의 범위와 관련된 log buffer의 fragment에 복사된다.

log buffer는 ring buffer이고, lsn 값에 의해 직접적으로 주소가 할당된다. 이는 log buffer에 있는 데이터를 더이상 옮길 필요가 없다는 뜻이다. 지정된 lsn의 바이트가 버퍼의 lsn modulo size에 저장된다. 이는 log buffer에 더 높은 병렬성을 가질 수 있는데, 데이터의 이동은 베타적인 접근이 필요하기 때문이다.

> 하지만, log buffer의 wrapped fragment를 디스크에 쓸 때, 추가적인 IO 작업이 발생할 수 있다(왜냐하면 2개의 분리된 메모리 공간을 복사해야 하기 때문이다). 먼저, 이는 거의 발생하지 않는 상황이므로 전혀 문제되지 않는다. 또한 wrapped fragment는 시스템 버퍼에의 추가적인 쓰기에서 발생하므로 실제 IO 작업의 수는 그대로 유지될 수 있다.

다른 lsn 값의 범위를 쓰는 작업은 동기화 없이 병렬적으로 발생한다. 각각의 유저 스레드는 log buffer에 자체 로그 레코드 시퀀스를 작성하여 mtr의 내부 버퍼에서 복사하여 연속된 로그 블록의 header와 footer를 위한 공간을 남긴다.

>  다수의 유저 스레드가 같은 메모리 캐시 line에 쓰기 작업을 할 때 숨겨진 동기화 작업이 발생한다. 이는 같은 64바이트에 쓸 때 발생한다. 왜냐하면 lsn값의 작은 연속적인 범위를 예약했기 때문이다. 다행히도 각각의 mtr은 평균적으로 몇 바이트 이상을 차지하며, 이는 캐시 라인 내에서 만나는 사용자 스레드의 수를 제한한다.

mtr_commit()이 로그 레코드 그룹 쓰기를 마치면 start_lsn이 속한 블록과 동일한 블록이(start_lsn에서 끝나는 사용자가 업데이트를 책임지는 경우) 아닌 한 end_lsn이 속한 블록의 헤더에 있는 first_rec_group 필드를 업데이트합니다.

## 최근에 쓰여진 버퍼에 링크 추가하기

현재 lsn에 가까운 log buffer의 fragment는 여러 유저 스레드에 의해 동시에 작성될 가능성이 매우 높다. 그러한 동시 쓰기가 완료되는 순서에는 제한이 없다. 쓰기 작업을 완료한 각각의 유저 스레드는 다른 유저 스레드를 기다리지 않고 계속 진행한다.

![[Pasted image 20230204132242.png]]

> 유저 스레드가 쓰기 작업을 완료했을 때, 아직 다른 유저 스레드는 그들의 데이터를 더 작은 lsn 값에 쓰려고 할 수 있다. 로그 버퍼의 완전한 fragment만 기록하는 것은 log writer thread이기 때문에 그래도 괜찮다. 이를 위해서는 우리는 완료된 쓰기 작업에 대한 정보가 필요하다.

log recent written buffer는 로그 버퍼에의 병렬적인 완료된 쓰기를 추적해야한다. 이는 log writer thread가 쓰기 작업을 위해 다음 완료된 fragment를 찾게해주는 log.buf_ready_for_write_lsn을 업데이트하도록 허락해준다. 이는 우리가 log.buf_ready_for_write_lsn까지 모든 쓰기가 완료되었다는 것을 알고 있기 때문에 최근 쓰기만 추적해도 충분하다. 그러므로 이 lsn 값은 주어진 시간 내에 recent written buffer로 표현되는 lsn 범위의 시작을 정의한다. recent written buffer는 ring buffer이고, lsn 값에 주소가 직접적으로 할당된다. 버퍼에 공간이 없으면 유저 스레드는 기다려야한다.

> 최근 쓰여진 로그 버퍼의 크기는 제한되어있어, 최근 쓰여진 버퍼가 너무 작다면 병렬성이 제한될 수 있고 유저 스레드는 서로를 기다려야한다(간접적으로 log writer thread에 의해 최근 쓰여진 버퍼에서 회수단 공간을 대기함으로써)

링크를 추가할 때 사용되는 절차를 알아보자.

유저 스레드가 mtr의 log record를 `tmp_start_lsn ... tmp_end_lsn` 의 lsn 값의 범위에 방금 기록했다고 가정하자.
1. 유저 스레드는 아래 수식이 만족될 때까지 recent written buffer의 빈 공간을 기다린다.
```
tmp_end_lsn - log.buf_ready_for_write_lsn <= S
```
S는 log recent written buffer 안에 있는 slot의 개수이다.

2. 유저 스레드는 tmp_start_lsn에 대한 slot의 값을 세팅함으로서 링크를 추가한다
```
log.recent_written[tmp_start_lsn % S] = tmp_end_lsn
```

이 값은 링크를 통과할 때 lsn을 얼마나 진행해야 하는지에 대한 정보를 제공한다.

> tmp_end_lsn < end_lsn일 수 있다. 이러한 경우에는 mtr의 다음 log record의 쓰기가 tmp_end_lsn에서 시작될 것이다. 모든 log record가 쓰여진 뒤에, tmp_end_lsn은 예약된 end_lsn과 같아야한다(우리가 써야하는 것 보다 더 많은 바이트를 예약해서는 안된다).

log writer thread는 추가된 링크에 의해 생성된 path를 따라가고, log.buf_ready_for_write_lsn을 업데이트하고 lsn보다 S가 큰 경우 재사용하기 위해 link를 지운다.

링크가 추가되기 전에, release barrier가 필요한데, 이는 컴파일 타임을 피하거나 로그 버퍼와 recent written buffer에의 쓰기의 메모리 재정렬을 피하기 위함이다. 로그 버퍼에 쓰기가 recent written buffer에 쓰기보다 우선하는지 확인하는 것이 중요하다.

log writer thread의 읽기에도 동일하게 적용되므로 log writer thread는 최근 작성된 버퍼에서 링크를 읽은 후 링크와 관련된 로그 버퍼 조각에서 적절한 데이터를 읽을 수 있다.

데이터 복사와 링크 추가는 mtr 내의 log record 그룹 내의 연속적인 log record에 대해 루프 방식으로 수행된다.

> 다른 유저가 모든 log record 쓰기를 완료하기 전까지 다른 유저 스레드에 의한 더 큰 lsn에 대한 log buffer에 기록된 모든 로그 레크드를 디스크에 기록할 수 없다. log writer thread는 log recent written buffer의 사라진 링크에서 멈출 것이고 대기할 것이다. 이것은 연결된 링크만 따라간다.

## 더티 페이지 마킹
lsn 값의 범위인 `start_lsn ... end_lsn`은 공간 예약할 동안 필요하고, 모든 log record 그룹을 대표한다. 이것은 mtr 내부의 모든 페이지를 더티 페이지로 마킹한다.

> 복구하는 동안 전체 mtr을 복구하거나 건너뛰어야한다. 그러므로 우리는 페이지를 마킹할 때 lsn 값의 디테일한 범위가 필요가 없다.

mtr에서 수정된 각 페이지는 잠겨있으며 가장 오래된 수정은 이것이 첫 번째 수정인지 또는 이 mtr에서 수정이 시작될 때 페이지가 이미 수정되었는지 확인하기 위해 검사된다.

처음 변경된 페이지는 다음과 같이 업데이트된다.
-   _oldest_modification_ = _start_lsn_
-   _newest_modification_ = _end_lsn_

그리고 해당 버퍼 풀에 대한 플러시 목록에 추가됩니다 (버퍼 풀은 page_id에 의해 샤딩됩니다).  

다른 페이지의 경우 newest_modification 필드만 업데이트됩니다 (end_lsn 포함).

> 몇몇 페이지들은 이미 이전 mtr에 의해 변경되어 flush 되지 않았을 수 있다. 그런 페이지들은 이 페이즈동안 oldest_modification != 0을 만족하고 이미 flush list에 속한다. 그러므로 newest_modification만 업데이트해도 충분하다.

## 더티 페이지를 플러시 리스트에 추가

mtr_commit() 내부의 모든 log record의 쓰기가 완료된 이후에 더티 페이지는 flush list로 이동해야한다. 다행히도 잠깐 후에 페이지는 플러시 될 것이고 log file의 공간은 재할당될 것이다.

다음은 페이지를 flush list로 추가하는 과정이다
1. end_lsn을 커버링하는 recent colsed buffer를 기다린다
페이지를 이동시키기 전에, 유저 스레드는 recent closed buffer 내의 start_lsn에서 end_lsn을 가리키는 링크를 위한 빈 공간이 있을 때 까지 대기하게 된다. 빈 공간은 다음과 같은 수식을 만족할 때 유효하게 된다.
```
end_lsn - log.buf_dirty_pages_added_up_to_lsn < L
```
L은 log recent closed buffer 내의 slot의 개수이다.

이렇게 하면 플러시 목록의 최대 지연이 L에 의해 제한된다는 보장이 있다. 왜냐하면 우리는 더 작은 lsn 값(L보다 작은 것)을 가지는 페이지가 추가될 때까지 너무 높은 lsn 값을 가지는 더티 페이지를 추가하는 것을 불허한다.

2. 더티 페이지를 flush list에 추가
이 단계 동안 페이지는 잠기고 더티 마크가 생긴다.

다수의 유저 스레드는 페이지 순서와 상관없이 수행할 수 있다. 그러므로 flush list의 더티 페이지의 순서는 그들의 oldest modification lsn과 같은 순서가 아니다.

![[Pasted image 20230204154416.png]]
**Relaxed order of dirty pages**

> 그래도 subsect_redo_log_buf_dirty_pages_added_up_to_lsn은 start_lsn보다 더 진행할 수 없다. 왜냐하면 start_lsn에서 end_lsn까지의 링크는 이 단계에선 추가되지 않았기 때문이다.

## 링크를 log recent closed buffer에 추가

모든 더티 페이지가 플러시 리스트에 추가된 이후에, start_lsn에서 end_lsn까지의 포인팅 링크는 log recent closed buffer에 추가된다.

이는 유저 스레드에 의해 수행되고, start_lsn에 대한 slot의 값을 다음과 같이 세팅한다
```
log.recent_closed[start_lsn % L] = end_lsn
```
L은 log recent closed buffer의 크기이다.  이 값은 링크를 통과할 때 lsn을 얼마나 전진시킬지에 대한 정보를 제공한다.

링크가 추가된 이후에, 로그 버퍼에의 샤딩된 접근이 가능해진다. 이는 배타적인 접근을 기다리고 있던 스레드에 대해 진행을 허가해준다.

## redo log의 공간을 해제함
복구 작업은 항상 마지막으로 쓰인 lsn의 체크포인트에서 시작한다는 것을 떠올려라. 게다가 log.last_checkpoint_lsn은 로그 파일의 시작을 정의한다.  로그 파일의 크기가 고정되어 있기 때문에 lsn 값의 지정된 범위가 로그 파일의 사용 가능한 공간에 해당하는지 여부를 쉽게 확인할 수 있다(이 경우 더 작은 lsn에 대해 redo 로그의 tail을 덮어쓴다).

로그 파일의 공간은 더 높은 lsn에 대한 체크포인트를 작성하여 회수된다. 이는 더 많은 더티 페이지가 플러시됐을 때 가능하다. 체크포인트는 어떤 더티 페이지의 oldest_modification 보다 더 높은 lsn에 쓸 수 없다(그렇지 않으면 손상이 발생했을 때 그 페이지의 변경사항을 잃게 된다). 안전한 다음 체크포인트(subsect_redo_log_available_for_checkpoint_lsn)를 위한 lsn을 지정하고 체크포인트를 쓰는 작업은 log checkpointer thread이 하게된다. 로그 버퍼에 쓰기를 수행하는 유저 스레드는 더 이상 뮤텍스를 보유하지 않으므로 이러한 lsn을 결정하고 체크포인트를 쓰는 것을 허용하지 않는다.

유저 스레드가 방금 로그 버퍼에 쓰는 것을 완료했다고 가정하자, 그리고 이는 방금 더티 페이지를 플러시 리스트에 추가하기 직전이지만 갑자기 스케쥴 아웃 당했다. 이제 log checkpointer thread가 들어와서 다음 체크포인트에 사용할 수 있는 lsn을 결정하려고 시도한다. 만약 우리가 스레드에게 플러시 리스트의 더티 페이지의 최소 oldest_modification을 가지게 허가하고, 그 lsn 값에 체크포인트를 쓰게 허가한다면, 우리는 논리적으로 그 값보다 작은 lsn 값의 log record들을 모두 지우게 된다. 하지만 유저 스레드가 플러시 리스트에 추가하려고하는 더티페이지는 oldest_modification보다 더 작은 값을 가질 수 있다. 그러면 변경을 보호하고 있는 로그 레코드는 손상이 발생했을 때 논리적으로 삭제되고 복구할 수 없게 된다.

그렇기 때문에 관련된 더티 페이지가 플러시 목록에 추가될때까지 논리적으로 redo log에 기록된 데이터를 지우는 lsn값에서 체크포인트를 수행하는 것을 방지해야한다.

유저 스레드가 `start_lsn .... end_lsn`과 관련된 모든 더티 페이지를 추가했을 때, log recent closed buffer에 start_lsn에서 end_lsn까지의 링크를 생성한다. log closer thread는 recent closed buffer에 있는 링크를 추적하고 slot을 비우고 subsect_redo_log_buf_dirty_pages_added_up_to_lsn을 업데이트하여 최근 닫힌 버퍼의 공간을 회수하고 잠재적으로 체크포인트를 더 진전시킬 수 있다.

플러시 리스트에 추가된 페이지의 순서는 relaxed되어 주어진 플러시목록에 가장 먼저 추가된 페이지의 lsn에 직접적으로 의존할 수 없다. 플러시 리스트에 추가된 페이지는 더이상 최소 oldest_modification을 가진다고 보장할 수 없게된다. 하지만 이들이 가지는 oldest_modification이 최소값보다 L 이상 높은 oldest_modification을 가지지 않는다고 보장한다. 그러므로 우리는 이 값에서 L을 빼고 주어진 플러시 리스트에 따른 체크포인트를 활성화시킬 lsn값으로 사용한다.