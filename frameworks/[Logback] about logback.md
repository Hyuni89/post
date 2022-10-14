# Logback?

Logback은 java 진영에서 많이 쓰이는 logging framework로 log4j를 대체하기 위해 개발되었다.  

1. `Logger`
2. `Appender`
3. `Layout`

위와 같이 3개의 class로 구성되었다.  
`Logger`는 log 메시지를 생성하는 모듈, `Appender`는 log 메시지를 출력하는 모듈, `Layout`은 log 메시지의 포맷팅에 관련된 모듈이라고 볼 수 있다.  

# How to Use?

## setup

```groovy
implementation("ch.qos.logback:logback-classic:{version}")
```

## basic

logback 설정은 `logback.xml`에서 설정할 수 있다. 파일명은 `-Dlogback.configurationFile={filename}`을 이용해 원하는 파일로 변경할 수 있다.  

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="debug">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```
```java
Logger logger = LoggerFactory.getLogger("test");
logger.info("log info example {}", 1);
logger.error("log error example " + 2);
logger.trace("log trace example {}", 3);
```
```bash
15:56:28.735 [main] INFO test - log info example 1
15:56:28.737 [main] ERROR test - log error example 2
```

code에서 `Logger`를 이용해 로그를 생성하고 출력하면 `Appender`에서 지정한 타겟 (위에선 `STDOUT`)에 `pattern`에 맞추어 로그가 출력되는 모습을 확인 할 수 있다.  
log level에 따라 `Logger`에서 지정해서 로그를 생성할 수도 있다. log level은 간단하게 `ERROR -> WARN -> INFO -> DEBUG -> TRACE` 순으로 중요도가 나뉠 수 있다. 설정에서 level이 `debug`로 설정되어서 마지막 로그인 `trace` 로그는 출력되지 않는 것을 확인 할 수 있다.  

### Appender

위의 예제와 같이 console에 출력하는 `Appender` 외에 파일에 로그를 출력하는 `FileAppender`, 파일에 쓰여진 로그를 rolling을 통해 관리해 주는 `RollingFileAppender`가 있다. 특히 서비스를 운영하다보면 로그가 많이 쌓이게 될텐데 날짜에 따라, 로그 크기에 따라 자동적으로 로그를 관리해 주는 `RollingFileAppender`는 유용하게 쓰일 수 있을 것이다.  

```xml
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>test.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>%d{yyyy-MM-dd-HH-mm}/test.log</fileNamePattern>
        <maxHistory>10</maxHistory>
        <cleanHistoryOnStart>true</cleanHistoryOnStart>
    </rollingPolicy>

    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %-4relative --- [ %thread{10} ] %logger{35} - %msg%n</pattern>
    </encoder>
</appender>
```
```bash
2022-10-05 18:09:38.794 ERROR 1221 --- [ main ] logger - in error log
```

로그는 `test.log` 파일에 출력된다. 시간 기준의 rolling 정책을 가지고 있으므로 매 분마다 rolling이 일어나고 롤링된 파일은 원래 로그 위치에서 `{yyyy-MM-dd-HH-mm}/test.log`로 이동한다. `maxHistory`값이 10이므로 rolling된 로그는 최대 10개까지 가지고 있고, `cleanHistoryOnStart` 설정으로 앱이 시작할 때 이전 rolling 대상이었던 로그들을 삭제해 준다.  

아무런 로그가 출력되지 않았을땐 rolling이 일어나지 않는다. 그리고 위와 같이 특정 디렉토리로 로그를 rolling해도 디렉토리까지 rolling될 때 삭제된다. 다만 그 전에 로그를 먼저 지우거나 그러면 rolling 대상에서 제외되는지 디렉토리가 삭제되지 않는다.  

### Layout

위 예제들에선 `PatternLayout`을 이용했다.  

```xml
<pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %-4relative --- [ %thread{10} ] %logger{35} - %msg%n</pattern>
```

`%`로 시작되는 것들은 예약어라고 볼 수 있다. 자세한 내용은 공식 문서를 참조하면서 확인하는 것이 좋을 것 같다.  

# link

[logback](https://logback.qos.ch/)