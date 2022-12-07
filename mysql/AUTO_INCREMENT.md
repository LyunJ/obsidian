
# innodb_autoinc_lock_mode
| innodb_autoinc_lock_mode | Effect |
| -- | -- |
| 0 | 모든 INSERT 문장이 AUTO_INCREMENT LOCK에 걸린다 |
| 1 | 서버가 INSERT 개수를 예측할 수 있으면 MUTEX를 사용하고, 예측할 수 없으면 AUTO_INCRMENT LOCK을 사용한다 |
| 2 | 모든 INSERT 문장이 MUTEX를 사용한다. 단 이 때 바이너리 로그 포맷은 ROW여야한다 |

