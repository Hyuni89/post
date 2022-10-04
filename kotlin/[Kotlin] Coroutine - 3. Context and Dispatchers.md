## Coroutine context and dispatchers

coroutine은 항상 `CoroutineContext` type에 있는 context 위에서 동작한다. coroutine context에는 어떤 thread 들에 coroutine을 동작시킬지 결정하는 coroutine dispatcher를 포함한다. 모든 coroutine builder인 `launch`나 `async` 모두 `CoroutineContext`를 옵셔널하게 인자로 받을 수 있다.

```kotlin
launch { // context of the parent, main runBlocking coroutine
    println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    delay(500)
    println("main runBlocking      : after delay in thread ${Thread.currentThread().name}")
}
launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
    println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    delay(500)
    println("Unconfined            : after delay in thread ${Thread.currentThread().name}")
}
launch(Dispatchers.Default) { // will get dispatched to DefaultDispatcher 
    println("Default               : I'm working in thread ${Thread.currentThread().name}")
    delay(500)
    println("Default               : after delay in thread ${Thread.currentThread().name}")
}
launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
    println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    delay(500)
    println("newSingleThreadContext: after delay in thread ${Thread.currentThread().name}")
}
```

위의 예제와 같이, `launch`에 context를 설정하지 않으면 현재 `CoroutineScope`의 context를 기본으로 사용한다. 따라서 `runBlocking`을 실행하는 main thread에서 동작한다. default dispatcher는 특정 dispatcher를 사용할 필요가 없을때 주로 사용되는데 background의 thread pool에서 thread를 공유하고 이 thread를 사용한다. `newSingleThreadContext`는 해당 scope를 위해 thread를 하나 생성하고 할당한다. 그래서 굉장히 비싼 작업이라고 볼 수 있다. 그래서 이를 사용하게 된다면 작업을 다 마쳤을 때 `close`를 호출해 resource를 반납하거나 최상위 변수에 할당해 앱 다른 부분에서 재사용할 수 있어야 한다. `Dispatchers.Unconfined`는 특별한 동작을 하는데, suspening function이 호출되는 suspension point까지는 coroutine을 호출한 thread, 위의 예제에선 main thread에서 실행된다. 그러나 다시 나머지 코드가 실행될 때는 다른 thread에서 이어서 실행되는 것을 알 수 있다. 그러므로 unconfined dispatcher는 CPU time을 적게 쓰거나 공유된 resource를 업데이트 하지 않는 coroutine에 적절하다고 볼 수 있다.

> unconfined dispatcher는 고급 기술로 나중에 실행되어야 할 coroutine이 필요 없는 경우거나 원치 않는 side-effect가 발생하는 특정 케이스에서 사용된다. 일반적인 경우엔 사용하지 않는 것을 권장한다.

### debuging tip

`-Dkotlinx.coroutines.debug` VM 옵션을 주고 thread name을 출력하면 coroutine 정보가 함께 표시된다.  
`CoroutineName` context를 이용하면 coroutine 이름을 명시적으로 지정할 수 있고 위와 같은 옵션으로 이름을 확인 할 수 있다.  
두 가지 이상의 context를 주입하려면 `+`를 사용해서 주입할 수 있다. `launch(Dispatchers.Default + CoroutineName("test")) { ... }`

### Children of a coroutine

structured concurrency에 의해 parent coroutine이 종료되면 children coroutine은 함께 종료된다고 지금까지 알고 있었다. 이는 children coroutine이 parent coroutine의 context를 상속 받아서이다. 그러나 parent-children간의 관계를 명시적으로 변경하는 두 가지 방법이 있다.

1.  coroutine을 실행할 때 전혀 다른 scope를 명시해서 실행하는 방법이 있다. 이를 통하면 parent context를 상속받지 않게 된다. (`GlobalScope.launch` 등을 이용)
2.  coroutine을 실행할 때 다른 `Job` 객체를 주면 parent scope를 덮어 써서 parent coroutine의 lifecycle을 따라가지 않게 된다.

```kotlin
// launch a coroutine to process some kind of incoming request  
val request = launch {  
	// it spawns two other jobs  
	launch(Job()) {  
		println("job1: I run in my own Job and execute 	independently!")  
		delay(1000)  
		println("job1: I am not affected by cancellation of the request")  
	}  
	// and the other inherits the parent context  
	launch {  
		delay(100)  
		println("job2: I am a child of the request coroutine")  
		delay(1000)  
		println("job2: I will not execute this line if my parent request is cancelled")  
	}  
}  

delay(500)  
request.cancel() // cancel processing of the request  
println("main: Who has survived request cancellation?")  
delay(1000) // delay the main thread for a second to see what happens
```
```
output:  
job1: I run in my own Job and execute independently!  
job2: I am a child of the request coroutine  
main: Who has survived request cancellation?  
job1: I am not affected by cancellation of the request
```

위와 같이 `launch(Job()) { ... }`으로 실행한 coroutine은 parent coroutine인 `val request = launch { ... }`가 `cancel()`로 종료될 때에도 종료되지 않고 마지막 `println()`을 한 것을 알 수 있다. 이와 달리 아무것도 지정하지 않은 두번째 `launch`는 마지막 `println()`을 출력하지 못하고 종료된 것을 볼 수 있다.

### Thread-local data

가끔 thread-local data를 사용해 데이터를 넘기고 싶은 니즈가 있을 수 있다. 그러나 coroutine에선 어떤 thread가 사용될지 몰라 데이터를 정확하게 전달하거나 가져오는 것이 매우 어렵다. 이를 해결하기 위해 `ThreadLocal`에 확장 함수로 구현된 `asContextElement`를 이용하면 이를 쉽게 해결해 준다.  
  

```kotlin
threadLocal.set("main")
println("Pre-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
val job = launch(Dispatchers.Default + threadLocal.asContextElement(value = "launch")) {
    println("Launch start, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    threadLocal.set("this")
    println("before yield, current thread: ${Thread.currentThread().name}, thread local value: '${threadLocal.get()}'")
    yield()
    println("After yield, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
}
job.join()
println("Post-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
```
```
output:
Pre-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
Launch start, current thread: Thread[DefaultDispatcher-worker-1 @coroutine#2,5,main], thread local value: 'launch'
before yield, current thread: Thread[DefaultDispatcher-worker-1 @coroutine#2,5,main], thread local value: 'this'
After yield, current thread: Thread[DefaultDispatcher-worker-2 @coroutine#2,5,main], thread local value: 'launch'
Post-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
```

먼저 구성을 보면 main thread에서 main이라는 값이 저장되고 중간에 `launch`에서 `Dispatchers.Default`를 사용해 background thread pool을 사용해 main thread와 다른 thread에서 해당 scope이 동작하게 된다. 이때 `threadLocal.asContextElement(value = "launch")` context를 인자로 넘긴 것을 확인 할 수 있다. 해당 scope에서 thread-local 값은 인자로 넘긴 launch인 것을 확인 할 수 있다. 해당 scope에 thread-local 값이 잘 주입이 된 것이다. 다시 thread-local의 값을 변경한 후, `yield`로 thread를 잠시 놓았다가 다시 실행했을때 thread가 다시 바뀐 것을 확인 할 수 있다. 그리고 변경한 thread-local 데이터는 적용되지 않고 scope가 처음 시작할 때 넘어온 launch 값은 변함이 없었다. 모든 coroutine이 종료되고 `threadLocal`의 값을 확인하니 맨 처음 저장했던 main이란 값이 나왔다. 이를 통해 thread와 상관 없이 coroutine scope 내에선 context로 넘어온 thread-local 데이터를 유지할 수 있고 값은 수정할 수 없다는 것을 확인 할 수 있었다.