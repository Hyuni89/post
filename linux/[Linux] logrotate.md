# Logrotate

서비스를 운영하다보면 쌓여가는 로그를 볼 수 있다. logrotate는 이렇게 쌓여가는 로그를 정리가 용이하게 지원해주는 리눅스 툴이라고 볼 수 있다.  
logrotate가 하는 역할은 크게,  

1. rotate, delete  
2. compression  
3. mailing  

이렇게 정리할 수 있다.  
사실 한번만 잘 세팅해 두면 변경할 일이 크게 없는 설정이라서 간단하게 많이 사용하는 옵션 위주로 정리했다.  

## config

크게 전체 logrotate 설정을 통합으로 설정하는 파일이 있고, 개별 로그에 적용하는 설정이 있다.  

```bash
# 전체 설정
/etc/logrotate.conf

# 개별 설정
/etc/logrotate.d/{files}
```

## options

```bash
daily                           # logrotate 주기 (hourly, weekly, monthly, yearly)
rotate {count}                  # rotate 해서 보관할 파일 갯수. -1일 경우 영구 보관
size {size}                     # 지정한 size보다 파일이 커지면 rotate 실행  
missingok                       # rotate할 로그 파일이 없어도 에러 없이 진행 (<-> nomissingok)  
minage {count}                  # rotate를 진행하지 않을 횟수  
maxage {count}                  # 지정된 날 후에 rotate된 파일 삭제  
create {mode} {owner} {group}   # rotate후 새로운 파일 생성 (<-> nocreate)
compress                        # rotate시 gzip으로 압축 여부 (<-> nocompress)
dateext                         # rotate시 이전 로그에 yyyyMMdd 형식의 날짜를 부여 (<-> nodateext)  
dateformat {format}             # dateext의 포맷 변경  
mail {address}                  # rotate된 로그 발송 (<-> nomail)  
prerotate
    {script}                    # rotate 실행 전 script 실행  
endscript
postrotate
    {script}                    # rotate 실행 후 script 실행
endscript
preremove
    {script}                    # 로그 삭제 전 script 실행
endscript
```

이 외에도 다양한 옵션이 많이 있으므로 공식 문서를 참고하거나 `man logrotate`를 툥해 확인 할 수 있다.  

## command

```bash
logrotate -d /etc/logrotate.d/{file}    # 디버그 모드로 logrotate 실행 (실제 logrotate 되지 않는다)
logrotate /etc/logrotate.d/{file}       # logrotate 실행
logrotate -f /etc/logrotate.d/{file}    # logrotate 강제 실행 (rotate가 될 필요 없는 파일도 모두 rotate 된다)
```

## history

logrotate가 최근 언제 실행되었는지 간단히 알 수 있는 방법으로 status 파일을 확인할 수 있다.  
```bash
cat /var/lib/logrotate/logrotate.status
```

## example

```bash
# /etc/logrotate.conf
weekly                      # 매주 logrotate 실행
rotate 8                    # rotate 파일은 8개 까지 생성
create                      # rotate된 후 파일 생성
dateext                     # rotate 될 때 날짜 확장자를 붙임
include /etc/logrotate.d    # 개별 설정 파일 import

/var/log/wtmp {             # /var/log/wtmp 로그에 대해 아래와 같은 설정 적용
    monthly                 # 매달 logrotate 실행
    create 0664 root utmp   # rotate 후 파일 생성 mode 0664, owner root, group utmp
    minsize 1M              # 정해진 rotate 주기때 1M보다 커지면 rotate 실행
    rotate 1                # rotate 파일 1개
}

/var/log/btmp {             # /var/log/btmp 로그에 대해 아래와 같은 설정 적용
    missingok               # 해당 파일이 없어도 에러없이 진행
    monthly                 # 매달 logrotate 실행
    create 0600 root utmp   # rotate 후 파일 생성 mode 0600, owner root, group utmp
    rotate 1                # rotate 파일 1개
}
```
```bash
# /etc/logrotate.d/{file}
{target}/{logfile}/{path}/app.log* {    # rotate를 적용할 로그 파일 (app.log를 prefix로 가지는 모든 로그)
    daily                               # 매일 rotate 실행
    rotate 0                            # rotate 파일은 0개. rotate가 실행되면 바로 파일 삭제를 의미
    nocreate                            # rotate시 새로운 파일은 생성하지 않기
    minage 5                            # 5일동안 rotate를 시키지 않음
    missingok                           # 로그 파일이 없어도 에러없이 진행
}
```

---

더 자세한 내용은 command를 이용해 `help`를 확인하는 것이 가장 좋은 방법

## link

[Logrotate Git](https://github.com/logrotate/logrotate)