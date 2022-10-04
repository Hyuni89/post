## Asynchronous Flow

suspending function은 비동기로 단 하나의 값을 반환한다. 그러면 여러 값을 비동기로 반환 받으려면 어떻게 해야 할까? 여기서 `Flow`가 등장한다.  
반환값으로 `List<Int>`를 사용한다면 결과 값으로 **한번에** 리스트에 포함된 모든 값을 받겠다는 의미이다. 동기에서 stream 데이터를 반환하려면 `Sequence<Int>`를 사용했던 것처럼, 비동기로 stream 데이터를 사용하려면 `Flow<Int>`를 사용해야 한다.

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(1000)
        emit(i)
    }
}

fun main() = runBlocking {
    launch {
        for (k in 1..3) {
            println("I'm not blocked $k")
            delay(1000)
        }
    }

    simple().collect { println(it) }
}
```

```
I'm not blocked 1
1
I'm not blocked 2
2
I'm not blocked 3
3
```

`flow`란 builder로 `Flow`를 생성한다. 생성된 `flow`는 suspending이 가능하다. 그런데 `simple` 함수에는 `suspend` 키워드가 빠져있는 것을 볼 수 있다. `emit` 키워드로 stream을 반환하고 `collect`로 결과를 수집하는 것을 확인 할 수 있다. 결과값을 보게 되면 `launch`로 coroutine이 실행되어서 번갈아 가면서 출력이 되는 것을 볼 수 있다. 만약 여기서 `simple` 함수 내에 `delay`를 `Thread.sleep`으로 변경하게 되면 main thread가 block을 당해 전혀 다른 결과가 출력된다.

```
1
2
3
I'm not blocked 1
I'm not blocked 2
I'm not blocked 3
```

또 다른 특징으로 `collect`가 호출되지 않으면 `flow`는 실행되지 않는다. 그리고 동일한 결과를 반환하는 `flow`라도 매번 `collect`를 호출하면 매번 새로 실행된다. (메모제이션 이런건 없다)

### Intermediate flow operators

`Flow`는 collections이나 sequences와 같이 operator를 통해 변형될 수 있다. 이러한 과정들은 업스트림에서 다운스트림으로 향하고, 호출해야 실행된다. (`collect`가 호출되어야지만 `flow`가 실행되는 것처럼). 그리고 이러한 operator들은 그 자체로 suspending 함수가 아니다. 단지 빠르게 정의된대로 변형된 `flow`를 반환해준다. 또한 내부에서 suspending 함수를 호출도 가능하다. `map`, `filter`, `transform` 등 다양한 operator들이 있다.

```kotlin
suspend fun performRequest(request: Int): String {
    delay(1000)
    return "response $request"
}

suspend fun emittingRequest(request: String): String {
    delay(1000)
    return "emit $request"
}

fun main() = runBlocking {
    (1..3).asFlow()
        .filter { it % 2 != 0 }
        .map { performRequest(it) }
        .transform {
            emit("Making request $it")
            emit(emittingRequest(it))
        }
        .collect { println(it) }
}
```

```
Making request response 1
emit response 1
Making request response 3
emit response 3
```

1에서 3까지 sequence가 `flow`로 생성된다. 단순히 결과만 봤을땐 `[1, 2, 3]`이 filter를 거쳐 `[1, 3]`이 되고, 각각 두가지 `emit`으로 반환되어서 `collect` 되었다고 볼 수 있으나, `delay` 함수를 통해서 정확히 어떻게 코드가 실행되었는지 파악할 수 있다. 앞에서 언급했다 싶이 `flow`는 cold이기 때문에 `collect`가 호출되어야 1부터 반환된다. 1은 `filter`를 거치고 `map`에서 1초 후에 `response 1`을 반환한다. 그리고 `transform`에서 `Making request response 1`이 반환되어 출력되고, 또 1초 후에 `emit response 1`이 반환되어 출력된다. 이 후에 2가 반환되어 `filter`에서 걸려 제거되고, 바로 3이 반환되어 1과 같은 프로세스로 1초 후에 `Making request response 3`이 출력되고 또 1초 후에 마지막 `emit response 3`이 출력된다.

### Flow context

`Flow`는 언제나 coroutine을 실행한 context에서 동일하게 실행된다. 지금까지의 예제는 main thread에서 실행되었으니 `flow` 역시 main thread에서 동작했다. 그러나 가끔 이 작업이 리소스가 굉장히 많이 필요한 작업, 혹은 무거운 작업이 되어서 다른 context에서 실행 되어야 할 필요가 있는 경우가 있을 수 있다. 이럴때 이전에 사용했던 `withContext`를 사용해 다른 context 상에서 작업을 진행하고 `emit`으로 값을 반환받고 싶지만 context preservation property (context 보존 특성) 때문에 `flow`에서는 다른 context로 `emit`이 불가능하다. 이를 해결하기 위해 `flowOn`이라는 함수를 사용한다.

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        Thread.sleep(100)
        log("Emitting $i")
        emit(i)
    }
}.flowOn(Dispatchers.Default)

fun main() = runBlocking<Unit> {
    simple().collect { value ->
        log("Collected $value") 
    } 
}   
```

```
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 1
[main @coroutine#1] Collected 1
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 2
[main @coroutine#1] Collected 2
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 3
[main @coroutine#1] Collected 3
```

`emit`과 `collect`가 다른 thread, 다른 coroutine에서 동작하는 것을 확인할 수 있다.

### Buffering

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

val time = measureTimeMillis {
    simple()
        .buffer()
        .collect { value -> 
            delay(300)
            println(value) 
        } 
}   
println("Collected in $time ms")
```

```
1
2
3
Collected in 1071 ms
```

모든 operator는 순차적으로 실행된다. 위의 예제에서 `buffer()`가 없었다면 100ms 후에 1이 반환되고, 300ms 뒤에 반환된 1이 출력되고, 모든 출력이 이루어지기까지 1200ms 이상이 걸렸을 것이다. `buffer`는 이후 sequence가 suspending 될때 앞에 `flow`를 모으는 역할을 한다. 그래서 100ms 후에 1이 반환되고, 300ms를 suspending 할때 2와 3을 100ms씩 기다려 반환한다. 그리고 순차적으로 300ms씩 기다리며 출력을 해서 100ms + 300ms + 300ms + 300ms = 1000ms 정도의 시간이 소요되는 것을 확인 할 수 있다. 이와 비슷한 함수로 `conflate()`가 있는데 이는 `buffer()`와 다르게 뒤의 sequence가 실행될 때 앞에 반환되는 요소가 쌓이면 버린다. (마지막 하나만 들고 있다). 위의 예제에서 `buffer()`만 `conflate()`로 변형되면 1이 출력되는 300ms를 기다리는 동안 2와 3이 반환되었지만 2는 버리고 3만 처리해서 1과 3만 출력된다. 이것과 또 유사한 것으로 `collectLatest`가 있는데 이것은 말 그대로 제일 마지막 요소만 처리한다.

### 그 외

`zip`, `combine`, `flatMapConcat`, `flatMapMerge`, `flatMapLatest` 등이 소개되었다.

### Flow exceptions

우리가 잘 아는 `try / catch` 구문으로 충분히 예외 처리가 잘 이루어진다. `flow` 내의 `catch()` 함수를 사용해서 예외를 처리 할 수 있다. 이때 예외는 업스트림에서 발생한 예외만 처리 가능하다. `catch` 구문의 다운스트림에서 발생한 예외는 처리가 불가능하다.

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking {
    simple()
        .catch { println("First Caught $it") }    // 발생된 예외가 다운스트림에 있어서 예외를 잡지 못함
        .onEach {
            check(it <= 1) { "Collected $it" }    // 예외 발생
            println(it)
        }
        .catch { println("Second Caught $it") }    // 발생된 예외가 업스트림에 있기 때문에 예외를 잡아낼 수 있다
        .collect()
}
```

```
Emitting 1
1
Emitting 2
Second Caught java.lang.IllegalStateException: Collected 2
```

### Flow completion

`finally`로 block이 종료될 때 완료 동작을 취할 수 있다. `Flow`에서 `onCompletion`을 이용해서 동일하게 완료 동작을 취할 수 있다. `onCompletion`의 장점으론 `Throwable` 파라미터로 해당 block이 정상적으로 종료되었는지 혹은 예외에 의한 종료인지 확인이 가능하다. 다만 예외 처리를 할 수 없고 해당 예외는 다운스트림으로 전달된다는 점에서 `catch()`와 다른점이 있다. 또한 `catch()`는 다운스트림에서 발생한 예외를 알지 못하지만 `onCompletion`은 다운스트림에서 발생한 예외도 알 수 있다.

```kotlin
fun main() = runBlocking<Unit> {
    simple()
        .onCompletion { cause -> if (cause != null) println("Flow completed exceptionally") }
        .catch { cause -> println("Caught exception") }
        .collect { value -> println(value) }
}     
```

```
예외 발생 시:
Flow completed exceptionally
Caught exception
```