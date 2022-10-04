## Coroutine exceptions handling

Coroutine builder는 다음과 같은 두가지 방식으로 예외를 처리한다.

1. 예외를 자동적으로 전파한다. (`launch`, `actor`)  
2. 사용자에게 예외를 보여준다. (`async`, `produce`)

1번의 경우 root coroutine에서 발생한 예외일 경우 **uncaught** 예외로 취급하고, 2번의 경우 사용자가 최종으로 예외를 처리해야 한다.  

```kotlin
@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val job = GlobalScope.launch {
        println("Throwing exception from lauch")
        throw IndexOutOfBoundsException()
    }
    job.join()
    println("Joined failed job")
    val deferred = GlobalScope.async {
        println("Throwing exception from async")
        throw ArithmeticException()
    }
    try {
        deferred.await()
        println("Unreached")
    } catch (e: ArithmeticException) {
        println("Caught ArithmeticException")
    }
}
```
```
Throwing exception from lauch
Exception in thread "DefaultDispatcher-worker-1" java.lang.IndexOutOfBoundsException
	at me.hyuni.exceptions001.Exceptions_001Kt$main$1$job$1.invokeSuspend(exceptions_001.kt:9)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:106)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:570)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:749)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:677)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:664)
	Suppressed: kotlinx.coroutines.DiagnosticCoroutineContextException: [StandaloneCoroutine{Cancelling}@36ad06d2, Dispatchers.Default]
Joined failed job
Throwing exception from async
Caught ArithmeticException
```

`GlobalScope`로 실행된 `launch`는 root coroutine에서 예외가 발생했고, 1번이므로 uncaught 예외로 위와 같은 예외가 출력된다. 이와 다르게 `async`는 2번이므로 `catch`를 통해 사용자가 예외를 처리함을 볼 수 있다. 위와 같이 uncaught 예외를 커스텀 처리 할 수 있는데, `CoroutineExceptionHandler`를 사용하면 된다. (`CoroutineExceptionHandler`는 **uncaught exception**에만 동작한다)

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    println("CoroutineExceptionHandler got $exception")
}
val job = GlobalScope.launch(handler) {
    throw AssertionError()
}
```

coroutine의 취소는 예외와 관련이 깊다. coroutine을 취소하면 `CancellationException`이 발생되기 때문이다. 그러나 이 예외는 모든 예외 handler에서 무시된다. 따라서 자식 coroutine이 취소된다면 이 예외는 부모 coroutine까지 전파되지 않는다. 그러나 `CancellationException`을 제외한 다른 예외들을 만나게 된다면 자식 coroutine의 예외가 부모 coroutine까지 전파되어 모든 coroutine이 종료되게 된다.  

만약 다수의 자식 coroutine에서 동시다발적으로 예외가 발생하게 된다면, 기본적으로 가장 먼저 발생한 예외가 전파된다. 뒤의 예외는 첫번째 예외의 `suppressed`에 붙는다.  