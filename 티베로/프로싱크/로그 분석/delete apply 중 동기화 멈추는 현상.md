apply.log
```
[10-12T11:09:21.577876] [REP-04] [T] recv ipc (type:PRS_IPC_REPLAY_TX, src_id:-1)

[10-12T11:09:21.577879] [REP-04] [I] recv tx (xid:281612415665396, type:1, tsn:180292812801, ctl:180287242241, stmt count:1000)

[10-12T11:09:21.577881] [REP-04] [D] execdir SQL (SELECT 1 FROM DUAL)

[10-12T11:09:21.578131] [REP-04] [T] replay delete (obj_id:75427, col_cnt:2)

[10-12T11:09:21.578140] [REP-04] [T] execute delete (DELETE FROM "TIBERO"."TEST_5" WHERE "C1" = :"C1" AND "C2" = :"C2" AND ROWNUM = 1)

[10-12T11:09:21.578141] [REP-04] [D] prepare SQL (DELETE FROM "TIBERO"."TEST_5" WHERE "C1" = :"C1" AND "C2" = :"C2" AND ROWNUM = 1)

[10-12T11:09:21.578146] [REP-04] [D] natparam #1 (type:1, len:4)

[10-12T11:09:21.578148] [REP-04] [D] 0000: 03 c2 8a 81                                      ....            

[10-12T11:09:21.578153] [REP-04] [D] natparam #2 (type:3, len:4)

[10-12T11:09:21.578154] [REP-04] [D] 0000: 73 6f 66 74                                      soft            

[10-12T11:09:21.594662] [REP-04] [T] replay delete (obj_id:75427, col_cnt:4)

[10-12T11:09:21.594709] [REP-04] [F] Internal Error with condition 'stmt->u.del.where_col_cnt <= dd_tbl->col_cnt' (PID=6858) (2 args) [prs_apply_dd.c:1690]
    stmt->u.del.where_col_cnt = 4 = 4 = 0x4
    dd_tbl->col_cnt = 2 = 2 = 0x2
[10-12T11:09:21.614200] [REP-04] [F] callstack dumped to /home/tibero/prosync4/var/o2t_ab/prosync.out.6858 [prs_signal.c:68]
[10-12T11:09:21.614231] [REP-04] [F] replay thread terminated abnormally. [prs_thread.c:214]
```

callstack
```
*** 2023/10/12 11:09:21.607 ***
    [00] 0x461978:0x7f3221635000 = prs_dump_callstack + 0x0068
    [01] 0x461cde:0x7f3221635230 = sig_abrt_term + 0x003e
    [02] 0x7f3222122630:0x7f3221635370 = __restore_rt + 0x0000
    [03] 0x450051:0x7f3221636230 = prs_replay_delete + 0x00b1
    [04] 0x4526c5:0x7f32216386f0 = prs_replay_dml + 0x0165
    [05] 0x4554c1:0x7f3221638810 = prs_replay_stmt + 0x05a1
    [06] 0x457b91:0x7f3221638bd0 = prs_replay_tx + 0x0391
    [07] 0x458e12:0x7f3221638d00 = prs_replay_tx_handler + 0x0232
    [08] 0x459a35:0x7f3221638d50 = prs_apply_replay_main + 0x0675
    [09] 0x7f322211aea5:0x7f3221638f10 = start_thread + 0x00c5
```