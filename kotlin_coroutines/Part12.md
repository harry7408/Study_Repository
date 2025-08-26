# Dispatcher
- 코틀린 코루틴 라이브러리가 제공하는 기능 중 하나는 코루틴이 실행되어야 할 스레드를 결정할 수 있다는 점이다
- CoroutineContext의 Dispatcher를 이용해 스레드 풀을 정할 수 있다

</br>

## 기본 디스패처 (Dispatchers.Default)

- Dispatcher를 설정하지 않으면 기본적으로 **`Dispatchers.Default`**로 설정된다
    - 현재 실행 중인 컴퓨터의 CPU 코어 개수와  동일 한 수의 스레드 Pool을 가지고 있다 (이론적인 최적의 스레드 수)

```kotlin
suspend fun main() = coroutineScope {
    repeat(1000) {
        launch {
            List(1000) { Random.nextLong()}.maxOrNull()

            val threadName=Thread.currentThread().name
            println("Running on Thread $threadName")

        }
    }
}
----------------------------------------
Running on Thread DefaultDispatcher-worker-1
Running on Thread DefaultDispatcher-worker-4
Running on Thread DefaultDispatcher-worker-3
Running on Thread DefaultDispatcher-worker-6
Running on Thread DefaultDispatcher-worker-5
...
(1~8까지 번갈아 가며 1000개가 출력됨) -> 8코어 Thread
```

- runBlocking은 Dispatcher가 설정되어 있지 않으면 자신만의 Dispatcher 사용 (main에서 실행)

</br>

## 기본 Dispatcher 제한

- Dispatcher.Default의 스레드를 다 써버려서 같은 Dispatcher를 사용하는 다른 코루틴이 실행될 기회가 제한될 수 있다
    - Dispatcher.Default의 `limitedParallelism`을  사용하면 같은 시간에 특정 수 이상의 스레드를 사용하지 못하도록 제한할 수 있다
    
    ```kotlin
    private val dispatcher=Dispatcher.Default
    	.limitedParallelism(5) // 5개의 스레드 사용 제한
    ```
    
    **⭐ limitedParallelism은 다른 Dispatcher에서도 유용하게 사용된다 (1.6버전 이후 제공)**
    
</br>

## Main Dispatcher

- 안드로이드를 포함한 Application 프레임워크는 Main 또는 UI 스레드 개념을 가지고 있다
- Main 스레드에서 코루틴을 실행하려면 Dispatchers.Main 을 사용하면 된다
    - **Main 스레드는 사용자와 상호작용 시 사용되므로 이를 Block 해서는 안된다 (ANR 발생의 원인)**
- `kotlinx-coroutines-android` 의존성이 있어야 활용 가능하다

</br>

## IO Dispatcher

- 파일을 읽고 쓰는 경우, 블로킹 함수를 호출하는 경우, 네트워크 작업을 하는 경우
    - 즉 Main Thread를 막지 않고 장기간 작업을 하기 위해 설계
- 코어 수에 따라 다르지만 일반적으로 64개의 스레드 Pool로 제한된다

```kotlin
suspend fun main() {
    val time= measureTimeMillis {
        coroutineScope {
            repeat(50) {
                launch(Dispatchers.IO) {
                    Thread.sleep(1000)
                }
            }
        }
    }
    println(time)
}
-------------------
약 1초 소요 (1050 정도 출력)
```

- Dispatchers.IO와 Dispatchers.Default는 같은 Thread Pool을 공유한다
- Dispatchers.Default에서 withContext로 Dispatchers.IO로 스레드가 바뀌면 Thread Pool의 한도가 IO의 한도로 적용 (대부분 같은 스레드에서 실행되나 한도는 IO의 한도로 적용)
- Dispatchers.Default와 Dispatchers.IO를 모두 최대치로 사용하는 경우 → 두 개의 스레드 한도를 모두 합친 것과 같다 (일반적으로 64 + 8개)

</br>

## 커스텀 Thread Pool을 사용하는 IO Dispatchers

- Dispatchers.IO에 limitedParallelism 함수를 적용하여 원하는 만큼 많은 수의 스레드 수를 정할 수 있다
    - limitedParallelism은 독립적인 스레드 Pool을 가진 새로운 Dispatcher를 만들기 때문

```kotlin
suspend fun main(): Unit = coroutineScope {
    launch {
        printCoroutinesTime(Dispatchers.IO)
    }
    
    launch {
        val dispatcher = Dispatchers.IO
            .limitedParallelism(100)
        printCoroutinesTime(dispatcher)
    }
}

suspend fun printCoroutinesTime(
    dispatcher: CoroutineDispatcher
) {
    val test = measureTimeMillis {
        coroutineScope {
            repeat(100) {
                launch(dispatcher) {
                    Thread.sleep(1000)
                }
            }
        }
    }
    println("$dispatcher took: $test")
}
-----------------------------
// 매번 실행 시 다른 결과가 나옴
LimitedDispatcher@6c9723f4 took: 1024
Dispatchers.IO took: 2018
```

- 기본 Dispatchers.IO는 64개로 제한되어 나머지 46개가 실행될 때 스레드 수가 모자라 대기하는 시간이 생기는 반면 limitedParallelism이 적용된 작업 결과는 1초정도 소요되었다
- Executors Class를 사용하여 Thread Pool을 만든 후 asCoroutineDispatcher 함수를 사용해 Dispatcher로 변형
    - close()로 닫아줘야 한다 (닫지 않으면 메모리 누수 발생)

<img width="298" height="318" alt="Image" src="https://github.com/user-attachments/assets/beadf295-3956-4133-acd5-ef26245e404c" />

⭐ 너무 많은 스레드는 자원을 비효율적으로 사용하고 너무 적은 스레드는 사용 가능한 스레드를 기다리게 되므로 적절히 판단하여 사용하는 것이 중요하다

</br>

## 정해진 수의 스레드 Pool을 가진 Dispatcher

- Executor 클래스를 사용하여 스레드의 수가 정해져 있는 스레드 풀이나 캐싱된 스레드 풀을 만들 수 있다
- ExecutorService, Executor 인터페이스 구현 및 asCoroutineDispatcher 함수를 이용하여 Dispatcher로 변형 가능

> Ex) Executor 사용하여 사용 스레드 풀 직접 관리하는 방법
> 

```kotlin
val NUMBER_OF_THREADS = 20
val dispatcher = Executors
			.newFixedThreadPool(NUMBER_OF_THREADS)
			.asCoroutineDispatcher()
```

- 이렇게 Dispatcher를 만들면 close 메서드로 닫아야 메모리 누수가 발생하지 않는다
- 최적의 스레드 수를 정하기는 어렵기 때문에 정해진 수의 스레드 풀을 만드는 것은 좋지 않을 수 있다

## 싱글 스레드로 제한된 Dispatcher

- 다수의 스레드 사용 시 `Race Condition`에 의해 원하는 결과를 얻지 못하는 경우가 생긴다
    - 동시간에 다수의 스레드가 공유 상태를 변경하면 아래와 같은 문제가 발생

```kotlin
var i = 0

suspend fun main(): Unit = coroutineScope {
    repeat(10_000) {
        launch(Dispatchers.IO) { 
            i++
        }
    }
    delay(1000)
    println(i) 
}
-----------------
실행 시 마다 10000보다 작은 숫자 출력
```

- 하나의 스레드를 가지는 Dispatcher를 사용하면 Race Condition 문제를 해결할 수 있다

```kotlin
// 대표적인 방법
val dispatcher=Executors.newSingleThreadExecutor()
		.asCoroutineDispatcher()
		

val dispatcher=Dispatchers.Default
	.limitedParallelism(1)
```

```kotlin
var i = 0

suspend fun main(): Unit = coroutineScope {
    val dispatcher = Dispatchers.Default
        .limitedParallelism(1)
    
    repeat(10000) {
        launch(dispatcher) {
            i++
        }
    }
    delay(1000)
    println(i) 
}
---------------
10000
```

- 허나 이 방법은 Race Condition을 해결할 수 있으나 몇가지 문제점이 존재
    - 더 이상 사용되지 않을 때는 스레드를 반드시 닫아줘야한다
    - 하나의 스레드만 가지고 있기 때문에 Blocking 되면 병렬적으로 작업이 처리될 수 없다
    
    ```kotlin
    suspend fun main(): Unit = coroutineScope {
        val dispatcher = Dispatchers.Default
            .limitedParallelism(1)
    
        val launch = launch(dispatcher) {
            repeat(5) {
                launch {
                    Thread.sleep(1000)
                }
            }
        }
        val time = measureTimeMillis { launch.join() }
        println("Took $time") // Took 5006
    }
    --------------------
    약 5초 이상 소요
    limitedParallelism의 인자로 5를 넣으면 약 1초정도 소요
    -> 1개의 스레드르 사용하고 있기 때문에 Sleep이 5번 반복되는 동안 순차적으로 실행
    ```

</br>

## 프로젝트 룸의 가상 스레드 사용하기

- JVM에서 Project Loom이라는 일반적인 스레드보다 훨씬 가벼운 가상의 스레드를 도입 (Blocking 비용이 적게 소모)
- JVM 19 이상 버전을 사용해야하고 코루틴을 대체할만한 요소는 아닌 것 같음 (Dispatcher.IO 가 막히는 경우가 아니라면)

> Loom Dispatcher 만드는 방법
> 

```kotlin
val LoomDispatcher = Executors
		.newVirtualThreadPerTaskExecutor()
		.asCoroutineDispatcher()
		
---------------------------
object LoomDispatcher : ExecutorCoroutineDispatcher() {

	override val executor: Executor = Executor { command -> 
		Thread.startVirtualThread(command)
	}
	
	override fun dispatch(context: CoroutineContext, block: Runnable) {
		executor.execute(block)
	}
	
	override fun close() {
		error("Cannot be invoked on Dispatchers.LOOM")
	}
}

--------------------
// 확장 프로퍼티 정의해야 위의 구현체를 사용할 수 있다
val Dispatchers.Loom = CoroutineDispatcher
	get() = LoomDispatcher
```

</br>

## 제한받지 않는 디스패처 (Dispatchers.Unconfined)

- Thread를 관리하는 Dispatcher pool이 변경되지 않는 다른 Dispatcher와 달리 구속되지 않는다
    - 현재 스레드에서 바로 실행이 된다
    - 재게 된다면 재개를 유발한 스레드에서 실행된다
- 단위 테스트 및 디버깅에만 사용하고 현업에서 잘 사용되지 않는 Dispatcher이다

```kotlin
suspend fun main(): Unit =
    withContext(newSingleThreadContext("Thread1")) {
        var continuation: Continuation<Unit>? = null
        
        launch(newSingleThreadContext("Thread2")) {
            delay(1000)
            continuation?.resume(Unit) // 재게되는 시점 (Thread 2)
        }
        
        launch(Dispatchers.Unconfined) {
            println(Thread.currentThread().name) 

            suspendCancellableCoroutine<Unit> {
                continuation = it // 일시 중단 후 continuation 객체에 할당
            }
            
            println(Thread.currentThread().name) 

            delay(1000) // 코루틴 Block
            
            println(Thread.currentThread().name)
        }
    }
 ----------------------
 Thread1
 Thread2
```

- 성능 적인 측면에서는 Dispatchers.Default가 Thread 스위칭을 일으키지 않아 비용이 가장 저렴하다
- 실행되는 스레드에 대하여 전혀 신경쓰지 않아도 될 때 선택해도 괜찮으나 현업에서는 권장 X (Main 스레드 활용 시 주의가 필요하기 때문)

</br>

## Main 디스패처로 즉시 옮기기

- withContext가 호출되면 코루틴은 중단되고 Queue에서 기다리다가 재게 된다
- 이미 실행되고 있는 코루틴이더라도 다시 배정하게 된다면 비용 소모가 든다
- Main Dispatchers를 사용해야할 때 반드시 필요할 때문 Dispatcher를 배정하여 비용을 줄이는 `Dispatchers.Main.immediate`가 존재하여 스레드 배정 없이 즉시 실행된다
    - 이미 Main Dispatchers에서 실행되고 있는 상황에서 조금이라도 Cost를 줄이고자할 때 활용

```kotlin
suspend fun showUser(user: User) = 
	withContext(Dispatchers.Main.immediate) {
		userNameElement.text= user.name
	}
```

</br>

## 컨티뉴에이션 인터셉터

- `ContinuationInterceptor`는 **코루틴 실행의 “재개(resume)” 시점을 가로채서, 실행이 어떤 스레드·컨텍스트에서 이어질지 제어**하는 역할을 한다
- Dispatching은 Continuation Interceptor 기반으로 작동하고 있다
    - ContinuationInterceptor 코루틴 Context는 코루틴이 중단되었을 때 interceptContinuation 메서드로 컨티뉴에이션 객체를 수정하고 포장한다

📎 https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.coroutines/-continuation-interceptor/ 

</br>

## 작업 종류에 따른 Dispatcher 성능 비교

- 각 열은 다음과 같은 순서
    - 1열 : 1초동안 중단
    - 2열 : 1초간 Block
    - 3열 : CPU 집약적 연산
    - 4열 : 메모리 집약적 연산

|  | 중단 | Blocking | CPU연산 | 메모리 연산 |
| --- | --- | --- | --- | --- |
| 싱글 스레드 | 1002 | 100,003 | 39,103 | 94,358 |
| Dispatchers.Default | 1002 | 13,003 | 8,473 | 21,461 |
| Dispatchers.IO | 1002 | 2,003 | 9,893 | 20,776 |
| 100개 스레드 | 1002 | 1,003 | 16,379 | 21,004 |
- 중단의 경우 스레드 수는 문제가 되지 않는다
- Blocking하는 경우 Thread 다다익선
- CPU 집약적 연산은 Dispatchers.Default 가 최적
- 메모리 집약적 연산은 Thread 다다익선

---
## 요약

- Coroutine Context인 Dispatcher는 코루틴이 실행 될 스레드나 스레드 풀을 결정한다
    - Dispatchers.Default : cpu 중심 연산에 활용
    - Dispatchers.Main : 메인스레드 접근 시 사용
    - Dispatchers.Main.immediate : 꼭 필요할 때만 재배정 하므로 cost를 조금이라도 낮출 때 활용
    - Dispatchers.IO : 입출력, 파일 다운로드 등 Blocking 연산을 할 필요가 있을 때 사용
    - 병렬 처리 1로 제한 : Race Condition을 막고자 할 때 활용
    - Dispatchers.Unconfined : 코루틴이 실행될 스레드를 신경쓰지 않아도 될 때 활용 (테스트 때만 쓰기)
