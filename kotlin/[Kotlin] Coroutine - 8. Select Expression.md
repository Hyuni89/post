## Select expression

Select expression은 실행중인 많은 suspending function을 동시에 대기하게 하고 가능한 요소를 선택해서 가져올 수 있게 만들어 준다.  

```kotlin
fun CoroutineScope.fizz() = produce<String> {
    while (true) {
        delay(300)
        send("Fizz")
    }
}

fun CoroutineScope.buzz() = produce<String> {
    while (true) {
        delay(500)
        send("Buzz!")
    }
}

suspend fun selectFizzBuzz(fizz: ReceiveChannel<String>, buzz: ReceiveChannel<String>) {
    select<Unit> {
        fizz.onReceive { println("fizz -> $it") }
        buzz.onReceive { println("buzz -> $it") }
    }
}

fun main() = runBlocking {
    val fizz = fizz()
    val buzz = buzz()
    repeat(7) {
        selectFizzBuzz(fizz, buzz)
    }
    coroutineContext.cancelChildren()
}
```
```
fizz -> Fizz
buzz -> Buzz!
fizz -> Fizz
fizz -> Fizz
buzz -> Buzz!
fizz -> Fizz
buzz -> Buzz!
```

`selectFizzBuzz` 함수를 보면 `select`에 대해 간단히 이해 할 수 있다. `fizz`란 채널과 `buzz`란 채널을 받아 그 두 채널에서 요소를 받아 선택적으로 코드를 실행할 수 있는 리스너 역할을 하는 것을 확인 할 수 있다. 위의 예제는 `onReceive`를 사용해서 채널에서 요소를 받는 것을 처리했는데 채널이 닫히게 되면 예외를 던지는데 이 예외를 처리하지 못한다. 이를 해결하기 위해 `onReceiveCatching`을 사용한다.  

```kotlin
suspend fun selectAorB(a: ReceiveChannel<String>, b: ReceiveChannel<String>): String =
    select<String> {
        a.onReceiveCatching {
            val value = it.getOrNull()
            if (value != null) {
                "a -> '$value'"
            } else {
                "Channel 'a' is closed"
            }
        }
        b.onReceiveCatching {
            val value = it.getOrNull()
            if (value != null) {
                "b -> '$value'"
            } else {
                "Channel 'b' is closed"
            }
        }
    }

fun main() = runBlocking {
    val a = produce<String> {
        repeat(4) { send("Hello $it") }
    }
    val b = produce<String> {
        repeat(4) { send("World $it") }
    }
    repeat(8) {
        println(selectAorB(a, b))
    }
    coroutineContext.cancelChildren()
}
```
```
a -> 'Hello 0'
a -> 'Hello 1'
b -> 'World 0'
a -> 'Hello 2'
a -> 'Hello 3'
b -> 'World 1'
Channel 'a' is closed
Channel 'a' is closed
```

`a`와 `b`에서 각각 4번씩 string을 producing하고 채널이 닫히는 것을 확인 할 수 있다. 그리고 닫힌 채널을 인지하고 예외가 처리됨을 알 수 있다. 여기서 보이는 특별한 점은 처음 예제와 다르게 producing 되는 부분에 delay가 존재하지 않아 동일하게 producing이 됨에도 불구하고 aabaab 순으로 `select`가 처리한다는 점이다. 이는 몇번을 재실행 해도 동일하게 실행됨을 알 수 있다. 그 이유로, `select`는 첫번째 구문(이 예제에선 `a.onReceiveCatching`)을 우선적으로 처리한다. 그럼 동시에 계속해서 producing이 일어난다면 `a`만 계속 처리될 것으로 보이기도 하지만 `a`가 `send`로 producing 될때 suspending 되면서 `b`도 역시 producing 할 기회를 갖게 된다. 결과적으로 `a`와 `b` 모두 처리가 되긴 하지만 `select`에 첫번째 구문이 우선되어 `a`가 더 많이 처리된다. 그리고 `onReceiveCatching`은 채널이 닫힌 것을 즉시 알아차려 예외를 처리한다. 그래서 `a`가 닫힌 이후로 `b`가 producing 되어도 `a`가 닫힌 예외를 먼저 처리하므로 `Channel 'a' is closed`만 계속해서 출력되는 것을 알수 있다.  

#### Selecting to send

Select expression은 `onSend`를 이용해 채널에 선택적으로 producing 할 수 있는 기능을 제공한다.  

```kotlin
fun CoroutineScope.produceNumbers(side: SendChannel<Int>) = produce<Int> {
    for (num in 1..10) {
        delay(100)
        select<Unit> {
            onSend(num) {}
            side.onSend(num) {}
        }
    }
}

fun main() = runBlocking {
    val side = Channel<Int>()
    launch {
        side.consumeEach { println("Side channel has $it") }
    }
    produceNumbers(side).consumeEach {
        println("Consuming $it")
        delay(250)
    }
    println("Done consuming")
    coroutineContext.cancelChildren()
}
```
```
Consuming 1
Side channel has 2
Side channel has 3
Consuming 4
Side channel has 5
Side channel has 6
Consuming 7
Side channel has 8
Side channel has 9
Consuming 10
Done consuming
```

`select`의 특성을 따라 첫번째 `onSend`가 호출 되었으나 consume 후에 250ms의 delay로 인해 `side.onSend`가 두번 연속 호출된다.  

#### Selecting deferred values

```kotlin
fun CoroutineScope.asyncString(time: Int) = async {
    delay(time.toLong())
    "Waited for $time ms"
}

fun CoroutineScope.asyncStringList(): List<Deferred<String>> {
    val random = Random(3)
    return List(12) { asyncString(random.nextInt(1000)) }
}

fun main() = runBlocking {
    val list = asyncStringList()
    val result = select<String> {
        list.withIndex().forEach { (index, deferred) ->
            deferred.onAwait {
                "Deferred $index produced answer '$it'"
            }
        }
    }
    println(result)
    val countActive = list.count { it.isActive }
    println("$countActive coroutines are still active")
}
```
```
Deferred 6 produced answer 'Waited for 43 ms'
11 coroutines are still active
```

12개 크기의 list를 생성하고 각각 index에 임의의 delay가 지난 후에 `Waited for $time ms`의 string이 들어간다. `select`를 이용해 deferred 값을 가져오게 되면 제일 먼저 생성된 값이 반환되는 것을 확인 할 수 있다. 나머지 값을 채우는 coroutine은 여전히 동작 중이어서 마지막에 11개의 coroutine이 동작중임을 알 수 있다.  

#### Switch over a channel of deferred values

`select`를 이용해 이전에 확인했던 채널을, 혹은 deferred values를 선택적으로 취할 수 있다.  

```kotlin
fun CoroutineScope.switchMapDeferreds(input: ReceiveChannel<Deferred<String>>) = produce<String> {
    var current = input.receive()
    while (isActive) {
        val next = select<Deferred<String>?> {
            input.onReceiveCatching {
                it.getOrNull()
            }
            current.onAwait {
                send(it)
                input.receiveCatching().getOrNull()
            }
        }
        if (next == null) {
            println("Channel was closed")
            break
        } else {
            current = next
        }
    }
}

fun CoroutineScope.asyncString(str: String, time: Long) = async {
    delay(time)
    str
}

fun main() = runBlocking {
    val chan = Channel<Deferred<String>>()
    launch {
        for (s in switchMapDeferreds(chan)) {
            println(s)
        }
    }
    chan.send(asyncString("BEGIN", 100))
    delay(200)
    chan.send(asyncString("Slow", 500))
    delay(100)
    chan.send(asyncString("Replace", 100))
    delay(500)
    chan.send(asyncString("END", 500))
    delay(1000)
    chan.close()
    delay(500)
}
```
```
BEGIN
Replace
END
Channel was closed
```

100ms 뒤에 `BEGIN`이라는 값을 반환하는 deferred가 producing된다. `input`에 값을 받았으므로 `currnet`에 deferred가 저장되고 `while` loop를 돌면서 대기한다. 100ms 뒤에 `select`에선 `current`의 `onAwait`로 값을 받아 다시 producing하고 다음 `input`을 기다린다. 100ms 뒤에 500ms 뒤에 `Slow`라는 값을 반환하는 deferred가 producing된다. `current`에 deferred가 저장되고 `while` loop를 돌면서 대기한다. 다시 100ms 뒤에 100ms 뒤에 `Replace`를 반환하는 deferred가 producing된다. 이때, `input`으로 받게 되어 `current`의 deferred가 새로 받은 이 deferred로 교체되게 된다. 100ms 뒤에 `current`의 `onAwait`로 값을 받아 producing하고 다음 `input`을 기다린다. 400ms 뒤에 500ms 뒤에 `END`를 반환하는 deferred가 producing된다. `current`에 deferred가 저장되고 `while` loop를 돌면서 대기한다. 500ms 뒤에 `current`의 `onAwait`를 통해 값을 받아 다시 producing한다. 다시 500ms 뒤에 채널이 닫히게 되고 `next`는 `null` 값을 가지게 되고 종료되게 된다.  