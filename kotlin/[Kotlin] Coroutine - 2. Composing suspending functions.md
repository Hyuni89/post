## Composing suspending functions

coroutine에서는 기본적으로 순차적으로 코드가 동작한다. 그러므로 A와 B라는 함수가 있고 A와 B가 순차적인 호출이 필요하다면 평범하게 A를 호출하고 B를 호출하면 순차적으로 호출 될 것이다. 만약 순차적인 호출이 필요하지 않고 A와 B가 독립적이고 동시적으로 동작하게 하고 싶다면 `async`를 사용할 수 있다. `async`는 `launch`와 마찬가지로 구별된 coroutine을 실행한다. 차이점이라면 `launch`는 동작에 대한 어떠한 결과값도 포함하고 있지 않은 `Job`을 반환하고, `async`는 `Deferred`(결과를 반환하는 light-weight non-blocking furue)를 반환한다. (`Deferred` 역시 `Job`이어서 취소 가능하다). 반환된 `Deferred`는 `await()`를 호출함으로써 결과값을 가져올 수 있다. `async`를 실행하면서 `start` 옵션을 설정할 수 있는데, `CoroutineStart.LAZY`를 설정하게 되면 `async`를 호출하는 시점에 동작이 시작되는 것이 아니라 `await()`로 값을 가져오길 원하는 곳에 이르거나 `start()`로 명시적으로 동작을 시작하라고 호출하게 되면 동작하게 된다.

### Async-style functions

`async` coroutine builder를 사용해 아래와 같은 비동기 함수를 만들수 있다

```kotlin
@OptIn(DelicateCouroutinesApi::class)
fun oneAsync() = GlobalScope.async {
    doOne()
}

@OptIn(DelicateCoroutinesApi::class)
fun twoAsync() = GlobalScope.async {
    doTwo()
}

val one = oneAsync()
val two = twoAsync()
runBlocking {
    println("${one.await()}, ${two.await()}")
}
```

`xxxAsync` 함수는 suspending function이 아니다. 그래서 이 함수는 어디서나 호출 될 수 있다. 그러나 함수가 동작하기 위해선 비동기 블록이 반드시 필요하다. 위의 예를 보면 coroutine 블럭이 아님에도 `xxxAsync` 함수가 호출 된 것을 알 수 있다. 그러나 그 값을 실제로 받기 위해선 `runBlocking`에서 `await()`로 값을 받는 비동기 블록이 반드시 필요하다는 의미이다.  
위와 같은 형태의 코드는 문제가 있는데 비동기 동작을 실행하는 `xxxAsync` 함수를 호출하는 부분과 그 결과를 받는 `await` 사이에 에러가 발생해서 예외처리 루틴을 타게 되었을 때 백그라운드에서 `xxxAsync`는 계속해서 동작하게 된다. 이런 문제점이 발생하지 않기 위해선 구조화된 동시성(structured concurrency)을 활용해야 한다. (자녀 coroutine은 부모 coroutine에 종속되어야 한다)

### Structured concurrency with async

```kotlin
suspend fun failedConcurrentSum(): Int = coroutineScope {
    val one = async<Int> {
        try {
            delay(Long.MAX_VALUE)
            42
        } finally {
            println("First child was cancelled")
        }
    }
    val two = async<Int> {
        println("Second child throws ans exception")
        throw ArithmeticException()
    }
    one.await() + two.await()
}

fun main() = runBlocking<Unit> {
    try {
        failedConcurrentSum()
    } catch(e: ArithmeticException) {
        println("Computation failed with ArithmeticException")
    }
}
```

suspending function인 `failedConcurrendSum` 안에 `Long.MAX_VALUE`값, 사실상 무한의 시간을 대기하는 `async` coroutine과 `ArithmeticException` 예외를 던지는 `async` coroutine이 있다. 두번째 coroutine이 실행되며 바로 예외를 던질 것이고 이 예외를 통해 coroutine은 모두 종료될 것이다. 이 때, 메인 coroutine이 종료하면서 자식 coroutine인 `val one = async<Int> { ... }`도 종료시키는데, structured concurrency를 만족하므로 이것이 가능하다. 위에서 설명했던 async-style function이었다면 첫번째 coroutine이 종료되지 않고 계속 실행되는 상태가 되었을 것이다.  
  
  
실행 결과

```
Second child throws an exception
First child was cancelled
Computation failed with ArithmeticException
```