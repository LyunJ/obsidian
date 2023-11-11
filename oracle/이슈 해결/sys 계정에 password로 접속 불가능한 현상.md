sys 계정 원격 접속이 가능한지 확인하기 위해 **oracle이 설치된 시스템에서** net service name을 사용해 접속하였지만 접속 불가능한 현상이 지속되었다.

```bash
sqlplus sys/password@net_service_name as sysdba
```
```text
[ora12c@localhost config]$ sqlplus SYS/tibero as sysdba

SQL*Plus: Release 12.2.0.1.0 Production on Mon Sep 25 11:00:16 2023

Copyright (c) 1982, 2016, Oracle.  All rights reserved.

ERROR:
ORA-01017: invalid username/password; logon denied
```

이 때 net_service_name을 생략하고 접속하면 정상적으로 되는 것을 보고 리스너 문제라고 착각하게 되었다.
```bash
sqlplus sys/password as sysdba
```
```text
[ora12c@localhost config]$ sqlplus SYS/tibero as sysdba

SQL*Plus: Release 12.2.0.1.0 Production on Mon Sep 25 11:12:28 2023

Copyright (c) 1982, 2016, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production
```

리스너를 껐다 키고, 정적 설정이 아닌 동적 설정으로 해보며 시간을 보냈지만 해결이 되지 않았고, 결국 일반 사용자로는 정상적으로 로그인이 되는 것을 확인하여 리스너 문제가 아닌 sys 계정 문제인 것을 확인하였다.

