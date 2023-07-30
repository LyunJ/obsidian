[Redo log source documentation](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG.html)


mtr_commit() : mini-transaction(mtr)에 있는 mtr_commit() 함수에 의해 데이터 페이지의 모든 변경점이 redo log에 기록된다.

보통 변경점들은 mlog_write_ulint()에 의해 수행된다. 몇몇 page-level 작업에서는 redo log 크기를 줄이기 위해 c함수의 코드번호와 매개변수만 기록된다.

하나의 mtr은 여러 페이지의 변경을 다룬다. 손상이 발생한 경우, mtr의 전체 변경사항 집합이 복구되거나 변경사항이 없어진다.

mtr의 생명주기동안, 로그의 변경점들은 mtr의 내부 버퍼에 저장된다. mtr의 내부 버퍼는 다수의 로그 레코드가 포함되어 있으며, 이 레코드는 서로 다른 수정된 페이지에 적용되는 변경 사항을 설명한다. mtr이 커밋될 때, 모든 로그 레코드는 로그 레코드의 단일 그룹 내의 로그 버퍼에 기록된다.

절차:
1. 로그 레코드의 크기가 계산된다(byte)
2. 로그 레코드를 위한 공간이 예약된다. lsn 값의 범위는 로그 레코드의 그룹을 위해 할당된다.
3. 로그 레코드들은 로그 버퍼의 예약된 공간에 쓰여진다.
4. 변경된 페이지들은 더티 페이지로 마킹되고 플러시 리스트로 이동된다. 모든 더티 페이지들은 동일한 범위의 lsn 값으로 표시된다.
5. 예약된 공간이 해제된다.

백그라운드 스레드는 로그 버퍼의 새로운 변경점을 로그 파일에 쓰는 작업을 한다. 로그 레코드를 위한 지속성을 요구하는 유저 스레드는 로그가 유저가 요구하는 지점까지 플러시 될 때 까지 기다려야 한다.

복구 중에는 오직 완전한 그룹의 로그 레코드만이 복구되고 적용된다. 예를들어, 트리에서 로테이션이 발생하여 3개의 노드(페이지)가 변경되면 전체 로테이션이 복구되거나 아무 것도 복구되지 않으므로 잘못된 구조의 트리로 끝나지 않는다.

redo 로그에 기록된 연속된 바이트값은 lsn 값으로 열거된다. 로그 버퍼에 기록된 각각의 바이트값은 1씩 증가한 현재 lsn값에 해당한다.

redo log에 있는 데이터는 512바이트의 연속적인 블록으로 구조화된다(OS_FILE_LOG_BLOCK_SIZE). 각각의 블록은 12 바이트의 헤더(LOG_BLOCK_HDR_SIZE)와 4바이트의 footer를 가진다(LOG_BLOCK_TRL_SIZE). 이 추가 바이트들은 또한 lsn 값으로 열거된다. 지금부터 언급되는 데이터 바이트는 헤더와 footer 바이트가 아닌 실제 로그 레코드를 의미한다. 

유저 트랜잭션이 커밋됐을 때 undo log와 관련된 추가적인 mtr이 커밋되고, 그리고 나서 유저 스레드는 redo log가 해당 mtr end의 로그 레코드가 있는 지점까지 플러시될 때까지 기다린다.

더티 페이지가 플러시 되고 있을 때, 플러시를 수행하는 스레드 먼저 redo log가 페이지의 가장 최신 변경점까지 플러시 될 때까지 기다려야한다. 그 후 페이지는 아마 플러시될 것이다. 손상이 발생했을 경우, 페이지의 가장 최신 버전으로 끝날 수 있으며 페이지의 이전 버전은 없을 것이다. 그렇다면 손상 이전의 잠재적으로 플러시 되지 않은 페이지들은 해당 버전으로 복구되어야 할 것이다. 이는 로그 레코드의 같은 그룹 내에서 변경된 페이지와 로그 레코드의 이전 그룹 내의 변경된 페이지에 적용된다.

## redo log의 구조
### 데이터 레이어
Redo log는 아래와 같은 데이터 레이어들로 구성된다.
1. Log files(형식적으로 4 - 32 GB) : 디스크에 저장된 물리적인 redo file
2. Log buffer (디폴트 64 MB) : 데이터를 그룹화하여 로그 파일에 기록하고 적절한 방식으로 데이터 형식을 지정: 로그 블록의 headers와 footers 포함, 체크섬 계산, 레코드 그룹의 범위 관리.
3. Log recent written buffer (4MB) : 최근에 쓰여진 로그 버퍼 추적. 로그 버퍼의 동시 쓰기를 허용하고 쓰기가 완료된 lsn까지 추적.
4. Log recent closed buffer(4MB) : 최근에 쓰여진 데이터와 일치하는 더티 페이지 중 이미 플러시 리스트에 추가된 것들을 추적. 더티 페이지가 플러시 리스트에 추가되어야할 순서를 완화하고, 추가된 더티 페이지의 lsn까지 추적. 몇몇 플러시 리스트에 추가되지 않은 더티페이지의 oldest_modification보다 큰 lsn에 체크포인트를 만들지 않아야한다(왜냐하면 유저 스레드는 scheduled out되었기 때문).
5. Log write ahead buffer(4kB) : redo file에 더 많은 바이트값을 먼저 쓰기 위해 사용하고, 또한 read-on-write 문제를 피하기 위해 사용된다. 이 버퍼는 또한 동시적으로 다음 유저 스레드에서 추가적인 데이터를 받아야할 완전하지 않은 로그 블록을 쓸 때 필요하다. 이러한 경우 우리는 먼저 불완전한 블록을 write ahead buffer에 복사한다.


## General rules

1.  User threads write their redo data only to the log buffer.
2.  User threads write concurrently to the log buffer, without synchronization between each other.
3.  The log recent written buffer is maintained to track concurrent writes.
4.  Background log threads write and flush the log buffer to disk.
5.  User threads do not touch log files. Background log threads are the only allowed to touch the log files.
6.  User threads wait for the background threads when they need flushed redo.
7.  Events per log block are exposed by redo log for users interested in waiting for the flushed redo.
8.  Users can see up to which point log has been written / flushed.
9.  User threads need to wait if there is no space in the log buffer.
    
    ![](https://dev.mysql.com/doc/dev/mysql-server/latest/dia_arch_writing.png)
    
    Writing to the redo log
    
10.  User threads add dirty pages to flush lists in the relaxed order.
11.  Order in which user threads reserve ranges of lsn values, order in which they write to the log buffer, and order in which they add dirty pages to flush lists, could all be three completely different orders.
12.  User threads do not write checkpoints (are not allowed to touch log files).
13.  Checkpoint is automatically written from time to time by a background thread.
14.  User threads can request a forced write of checkpoint and wait.
15.  User threads need to wait if there is no space in the log files.
    
  ![](https://dev.mysql.com/doc/dev/mysql-server/latest/dia_arch_deleting.png)
    
    Reclaiming space in the redo log
    
16.  Well thought out and tested set of _MONITOR_ counters is maintained and documented.
17.  All settings are configurable through server variables, but the new server variables are hidden unless a special _EXPERIMENTAL_ mode has been defined when running cmake.
18.  All the new buffers could be resized dynamically during runtime. In practice, only size of the log buffer is accessible without the _EXPERIMENTAL_ mode.
    
    Note
    
    This is a functional change - the log buffer could be resized dynamically by users (also decreased).
    

# Glossary of lsn values

Different fragments of head of the redo log are tracked by different values:

-   [log.write_lsn](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG.html#subsect_redo_log_write_lsn),
-   [log.buf_ready_for_write_lsn](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG.html#subsect_redo_log_buf_ready_for_write_lsn),
-   [log.sn](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG.html#subsect_redo_log_sn).

Different fragments of the redo log's tail are tracked by different values:

-   [subsect_redo_log_buf_dirty_pages_added_up_to_lsn](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG.html#subsect_redo_log_buf_dirty_pages_added_up_to_lsn),
-   [subsect_redo_log_available_for_checkpoint_lsn](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG.html#subsect_redo_log_available_for_checkpoint_lsn),
-   [log.last_checkpoint_lsn](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG.html#subsect_redo_log_last_checkpoint_lsn).

## log.write_lsn

Up to this lsn we have written all data to log files. It's the beginning of the unwritten log buffer. Older bytes in the buffer are not required and might be overwritten in cyclic manner for lsn values larger by _log.buf_size_.

Value is updated by: [log writer thread](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG_THREADS.html#sect_redo_log_writer).

## log.buf_ready_for_write_lsn

Up to this lsn, all concurrent writes to log buffer have been finished. We don't need older part of the log recent-written buffer.

It obviously holds:

    log.buf_ready_for_write_lsn >= log.write_lsn

Value is updated by: [log writer thread](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG_THREADS.html#sect_redo_log_writer).

## log.flushed_to_disk_lsn

Up to this lsn, we have written and flushed data to log files.

It obviously holds:

    log.flushed_to_disk_lsn <= log.write_lsn

Value is updated by: [log flusher thread](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG_THREADS.html#sect_redo_log_flusher).

## log.sn

Corresponds to current lsn. Maximum assigned sn value (enumerates only data bytes).

It obviously holds:

    log.sn >= log_translate_lsn_to_sn(log.buf_ready_for_write_lsn)

Value is updated by: user threads during reservation of space.

## subsect_redo_log_buf_dirty_pages_added_up_to_lsn

log.buf_dirty_pages_added_up_to_lsn

Up to this lsn user threads have added all dirty pages to flush lists.

The redo log records are allowed to be deleted not further than up to this lsn. That's because there could be a page with _oldest_modification_ smaller than the minimum _oldest_modification_ available in flush lists. Note that such page is just about to be added to flush list by a user thread, but there is no mutex protecting access to the minimum _oldest_modification_, which would be acquired by the user thread before writing to redo log. Hence for any lsn greater than _buf_dirty_pages_added_up_to_lsn_ we cannot trust that flush lists are complete and minimum calculated value (or its approximation) is valid.

Note

Note that we do not delete redo log records physically, but we still can delete them logically by doing checkpoint at given lsn.

It holds (unless the log writer thread misses an update of the [log.buf_ready_for_write_lsn](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG.html#subsect_redo_log_buf_ready_for_write_lsn)):

    log.buf_dirty_pages_added_up_to_lsn <= log.buf_ready_for_write_lsn.

## subsect_redo_log_available_for_checkpoint_lsn

log.available_for_checkpoint_lsn

Up to this lsn all dirty pages have been flushed to disk. However, this value is not guaranteed to be the maximum such value. As insertion order to flush lists is relaxed, the buf_pool_get_oldest_modification_approx() returns modification time of some page that was inserted the earliest, it doesn't have to be the oldest modification though. However, the maximum difference between the first page in flush list, and one with the oldest modification lsn is limited by the number of entries in the log recent closed buffer.

That's why from result of buf_pool_get_oldest_modification_approx() size of the log recent closed buffer is subtracted. The result is used to update the lsn available for a next checkpoint.

This has impact on the redo format, because the checkpoint_lsn can now point to the middle of some group of log records (even to the middle of a single log record). Log files with such checkpoint are not recoverable by older versions of InnoDB by default.

Value is updated by: [log checkpointer thread](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG_THREADS.html#sect_redo_log_checkpointer).

See also

[Adding dirty pages to flush lists](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG_BUF.html#sect_redo_log_add_dirty_pages)

## log.last_checkpoint_lsn

Up to this lsn all dirty pages have been flushed to disk and the lsn value has been flushed to the header of the redo log file containing that lsn.

The lsn value points to place where recovery is supposed to start. Data bytes for smaller lsn values are not required and might be overwritten (log files are circular). One could consider them logically deleted.

Value is updated by: [log checkpointer thread](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG_THREADS.html#sect_redo_log_checkpointer).

It holds:

    log.last_checkpoint_lsn
    <= log.available_for_checkpoint_lsn
    <= log.buf_dirty_pages_added_up_to_lsn.

Read more about redo log details:

-   [Redo log buffer](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG_BUF.html)
-   [Background redo log threads](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG_THREADS.html)
-   [Format of redo log](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_INNODB_REDO_LOG_FORMAT.html)