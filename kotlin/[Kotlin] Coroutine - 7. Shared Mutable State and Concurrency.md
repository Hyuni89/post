## Shared mutable state and concurrency

Coroutine에서도 여러 thread를 병렬적으로 사용할 수 있고, 이에 따라 여러 thread를 사용함으로서 발생하는 문제들을 동일하게 겪게 된다. 가장 큰 문제는 공유된 자원의 접근이 아닐수 없다. Coroutine에서 사용하는 방법은 일반적인 multi-thread 환경에서 사용하는 방법과 비슷한 것도, coroutine에만 있는 것도 있다.  

#### The Problem

```kotlin
suspend fun massiveRun(action: suspend() -> Unit) {
    val n = 100
    val k = 1000
    val time = measureTimeMillis {
        coroutineScope {
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")
}

var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter == $counter")
}
```

위와 같이 `counter`값을 증가시키는 동작을 100번 반복하는 coroutine을 1000개 실행시키는 코드가 있다. context가 `Dispatchers.Default`로, background thread pool에 있는 thread에서 돌게 되어 multi-thread 환경이 조성되었다. 이 코드를 실행하면 100000의 값이 `counter`에 남아있길 기대하지만 실제로 코드를 돌려보면 한참 못미친 값이 출력된다. 우리가 multi-thread 환경에서 자주 보았던 동기화 문제이다.  

이번엔 이런 상황을 해결한다고 오해하고 있는 `volatile`을 사용해 위의 코드를 동일하게 실행시켜 본다.  

```kotlin
@Volatile
var counter = 0
```

그러나 결과는 이전과 동일하게 기대했던 값이 출력되지 않는다. 사실 `volatile`은 선형적인 읽기와 쓰기를 보장하지만 원자성을 나타내진 않기 때문이다.  

#### Thread-safe data structures

가장 일반적이고 쉬운 방법으로 thread-safe한 데이터 구조를 사용하는 것이다. 여기에선 `AtomicInteger`를 사용해 `counter`를 계산 할 수 있을 것이다.  

```kotlin
var counter = AtomicInteger()

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.incrementAndGet()
        }
    }
    println("Counter == $counter")
}
```

#### Thread confinement fine-grained

Thread confinement는 다양한 thread에서 접근하는 개념을 변경해 제한된 하나의 thread에서만 자원에 접근하는 것을 의미한다.  

```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            withContext(counterContext) {
                counter++
            }
        }
    }
    println("Counter == $counter")
}
```

`counter`에 접근할 수 있는 단 하나의 `counterContext`를 생성하고, 이 context 안에서만 `counter`에 접근해서 값을 변경한다. 결과는 기대했던 값이 정확하게 나오는 것을 확인 할 수 있다. 그런데 한가지 문제가 있는데, 엄청 느리다는 것이다. 왜냐하면 multi-thread가 `counter`의 값을 변경하기 위해 매번 `counterContext`로 context를 변경해야 하는데 여기에서 드는 비용이 크기 때문이다.  

#### Thread confinement coarse-grained

사실, thread confinement는 비지니스 로직의 상태를 변경하는 큰 부분이라든지 거대한 덩어리 단위로 실행된다. 바로 직전의 예제에선 `counter` 값을 변경하는 것 자체를 하나의 덩어리로 보았지만 이번 예제는 전체 코드를 하나의 덩어리로 보고 하나의 thread로 실행시켰다.  

```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking {
    withContext(counterContext) {
        massiveRun {
            counter++
        }
    }
    println("Counter == $counter")
}
```

단일 thread이므로 속도도 빠르고 `counter`도 기대했던 값과 정확히 일치한다.  

#### Mutual exclusion

익숙한 mutual exclusion은 상태를 공유하고 그 상태에 따라 critical section에 진입 여부를 결정하므로 결코 동시에 실행 될 수 없다. 일반적으로 `synchronized`가 가장 익숙할텐데, coroutine에선 `Mutex`를 사용해 `lock`, `unlock`으로 critical section을 제한한다.  

```kotlin
val mutex = Mutex()
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            mutex.withLock {
                counter++
            }
        }
    }
    println("Counter == $counter")
}
```

`withLock`을 통해 `lock`과 `unlock`을 편하게 범위로 지정할 수 있다. 결과는 기대했던 그대로 출력되지만 critical section이 병목 효과를 일으켜 다소 속도면에서 효과가 좋지 못한 결과가 나오는 것을 확인할 수 있다.  

#### Actors

`actor`는 제한되고 캡슐화 된 coroutine 상태와 다른 coroutine과 통신할 수 있는 채널로 구성된 요소이다. (다시 말해, 상태와 채널의 조합)  

```kotlin
sealed class CounterMsg
object IncCounter : CounterMsg()
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg()

fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0
    for (msg in channel) {
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}

fun main() = runBlocking<Unit> {
    val counter = counterActor()
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.send(IncCounter)
        }
    }
    val response = CompletableDeferred<Int>()
    counter.send(GetCounter(response))
    println("Counter == ${response.await()}")
    counter.close()
}
```

잘 이해가 안되니 바로 코드를 보면, `CounterMsg`를 상속받은 `IncCounter`과 `GetCounter`라는 상태가 `channel`을 통해 전달되고, 그 상태에 맞춰 `actor`의 동작이 결정된다. 중요한 점은 `actor`가 어떤 context에서 실행되었어도 그것과 상관 없이 순차적으로 실행되어 결과가 올바르게 나오는 것을 보장한다.  

