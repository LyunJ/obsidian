Prosync 프로세스는 config 파일을 읽고 rule db에 해당 설정을 넣고 기동된다.

그렇기 때문에 추가적으로 config 디렉토리의 설정을 변경해도 즉시 적용되지 않는다.

rule db에 직접 config에 입력한 설정을 적용시키기 위해선 `prs_config_loader.sh {config file name}`을 입력한다.

Prosync Manager는 Rule DB를 읽고 연결 설정을 하므로 만약 Prosync Manager에 필요한 추가적인 설정이 필요한 경우 위의 명령어로 설정을 적용시켜야 한다.