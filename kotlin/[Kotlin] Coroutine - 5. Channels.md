## Channels

Channel은 FIFO(First-In First-Out)이기에 BlockingQueue와 개념적으로 비슷하다. 한가지 중요한 차이점은 BlockingQueue에 item을 넣는 blocking `put` 대신에 Channel은 suspending `send`를 사용한다는 것과, BlockingQueue에서 item을 가져오는 blocking `take` 대신에 suspending `receive`를 사용한다는 것이다.  
```kotlin
fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) channel.send(x * x)
        channel.close()
    }
    for (y in channel) println(y)
    println("Done")
}
```
```
1
4
9
16
25
Done
```
`close()`를 사용함으로 Channel에 item이 더 이상 없다는 것을 명시적으로 나타낼 수 있고, `for` loop를 이용해 쉽게 Channel에 들어있는 item만을 받을 수 있다. 사실, 개념적으론 `close()`는 특별한 close token을 Channel에 보내는 것인데 iteration에서 해당 token을 받으면 즉시 종료되므로 위와 같은 동작이 가능한 것이다.  
Channel을 생성할 때 buffer 크기를 명시하면 (ex. `Channel<*>(4)`) buffered channel이 된다. buffered channel은 말 그대로 buffer가 있는 channel인데, producer에 의해 buffer가 가득 차게 되면 producer는 suspend 상태가 된다.  

### Pipelines

Coroutine이 일련의 item을 생성하는 것은 producer-consumer 패턴의 일부이다. 이를 편리하게 하기 위해 `producer`라는 coroutine builder가 있고, `CoroutineScope`의 확장 함수로 `consumeEach`라는 consumer가 있다. 이런 producer와 consumer를 이용해 pipeline을 꾸릴 수 있다.  
```kotlin
fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++)
}

fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}

fun main() = runBlocking {
    var cur = numbersFrom(2)
    repeat(10) {
        val prime = cur.receive()
        println(prime)
        cur = filter(cur, prime)
    }
    coroutineContext.cancelChildren()
}
```
```
2
3
5
7
11
13
17
19
23
29
```
`numbersFrom` 함수를 통해 `start`부터 1씩 증가하는 무한의 stream을 생성한다. `filter` 함수에선 Channel에서 받은 `numbers`를 `prime`으로 나누어 소수인지 확인하고, 소수이면 producing한다. 이 두가지를 엮어 소수를 생성하는 코드가 만들어졌다.  
2부터 1씩 증가하는 producer를 생성한다. 그리고 Channel에서 consuming을 해서 2를 얻어온다. 이 2는 prime 변수에 저장되고 `filter` 함수를 통해 2로 나누어 떨어지지 않는 수를 다시 producing하도록 한다. loop가 반복할 수록 `filter`가 더 겹치게 되고, 소수만 반환하게 된다. 결과적으로만 보면 소수가 출력되지만 사실은 pipeline이 계속 쌓이면서 (ex. numberFrom(2) -> filter(2) -> filter(3) -> filter(5) -> ...) 동작이 이루어지기 때문에 제일 처음에 있는 `numbersFrom`에서 producing하는 숫자는 모든 숫자이다.  

### Fan-in/out

여러 coroutine이 같은 Channel에서 item을 받을 수 있고 (fan-out), 여러 coroutine에서 같은 Channel로 item을 보낼 수 있다. (fan-in)  
```kotlin
suspend fun produceNumbers(channel: SendChannel<Int>, start: Int, wait: Long) {
    var s = start
    while (true) {
        delay(wait)
        channel.send(s++)
    }
}

fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }
}

fun main() = runBlocking {
    val channel = Channel<Int>()
    launch { produceNumbers(channel, 1, 200L) }
    launch { produceNumbers(channel, 400, 300L) }
    repeat(5) { launchProcessor(it, channel) }
    delay(950)
    channel.cancel()
}

```
```
Processor #0 received 1
Processor #1 received 400
Processor #2 received 2
Processor #3 received 3
Processor #4 received 401
Processor #0 received 4
Processor #1 received 402
```

### Ticker channels

마지막 consume 이후 정해진 매 시간마다 `Unit`을 producing하는 channel이다. 
```kotlin
fun main() = runBlocking {
    val tickerChannel = ticker(delayMillis = 100, initialDelayMillis = 0)
    var nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Initial element is available immediately: $nextElement")

    nextElement = withTimeoutOrNull(50) { tickerChannel.receive() }
    println("Next element is not ready in 50 ms: $nextElement")

    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() }
    println("Next element is ready in 100 ms: $nextElement")

    println("Consumer pauses for 150ms")
    delay(150)

    nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Next element is available immediately after large consumer delay: $nextElement")

    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() }
    println("Next element is ready in 50ms after consumer pause in 150ms: $nextElement")

    tickerChannel.cancel()
}
```
```
Initial element is available immediately: kotlin.Unit
Next element is not ready in 50 ms: null
Next element is ready in 100 ms: kotlin.Unit
Consumer pauses for 150ms
Next element is available immediately after large consumer delay: kotlin.Unit
Next element is ready in 50ms after consumer pause in 150ms: kotlin.Unit
```
100ms delay를 가진 `ticker`를 생성한다. `initialDelayMillis`가 0이므로 생성될 때 첫번째 `Unit`이 producing된다. 1ms 뒤에 consume하면 `Unit`이 있다. 50ms 뒤에 다시 consume하면 100ms가 되지 않았기 때문에 `null`이 반환된다. 60ms 뒤에 다시 consume 하면 100ms가 지났기 때문에 `Unit`이 producing된다. 150ms의 delay동안 `Unit`이 다시 producing 되고, 다음 producing까지 50ms가 남게 된다. 이때 만약 200ms를 delay하게 된다면 2개의 `Unit`이 producing되는 것이 아니라 이전것은 버리고 새로 생성된 `Unit`만을 가지고 있는다. 1ms 뒤에 consume 하면 150ms의 delay에서 이미 하나가 producing 되었으므로 존재하고, 마지막 60ms 뒤에는 이전에 150ms delay에서 `Unit`이 producing 된 후 50ms가 이미 지났었기 때문에 다시 producing 되는 것을 확인 할 수 있다.  
만약 고정 대기시간마다 `ticker`를 동작하게 하고 싶다면 mode에 `TickerMode.FIXED_DELAY` 옵션을 주면 된다.  