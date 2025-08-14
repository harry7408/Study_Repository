# 취소
- 취소는 Coroutine에서 **아주 중요한 기능** 중 하나
- 중단 함수를 사용하는 몇몇 Class와 라이브러리는 **반드시** 취소를 지원

Ref) Coroutine의 취소 기능을 주로 사용하기 위해 안드로이드에서도 Coroutine을 지원하게 되었다 (ex. CoroutineWorker)

### 취소 방법 (안 좋은 예)

- **단순히 Thread를 죽이면 연결을 닫고 자원을 해제하는 기회가 없어** **최악의 취소 방법**이라고 할 수 있다.
- 개발자들이 상태가 Active한지 자주 확인하는 방법은 불편하다

## 기본적인 취소

- Job 인터페이스는 취소하게 하는 **cancel 함수**를 가지고 있다
    
    <img width="918" height="370" alt="Image" src="https://github.com/user-attachments/assets/63ab48ab-9ab9-45e2-bc05-4484ff309864" />
    
- cancel 함수는 아래 3가지 효과를 낼 수 있다
    - 호출한 Coroutine은 **첫 번째 중단점에서** Job을 끝낸다
    - Job이 자식을 가지고 있다면 그들 또한 취소된다. 다만 부모는 영향을 받지 않는다
    - Job이 취소되면 **취소된 Job은 새로운 Coroutine의 부모로 사용될 수 없다**
    (Cancelling 상태가 되었다가 Cancelled 상태로 전환)

```kotlin
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

suspend fun main() : Unit = coroutineScope {
    val job= launch {
        repeat(1_000) { i->
		        // 코루틴의 중단점
            delay(200)
            println("Printing $i")
        }
    }
    // delay 에 넣는 시간에 따라 출력 결과가 달라 진다.
    delay(1100)
    job.cancel()
    job.join()
    println("Cancelled Successfully")
}
--------------
Printing 0
Printing 1
Printing 2
Printing 3
Printing 4
Cancelled Successfully
```

- cancel이 호출되어 첫 번째 중단점인 `delay`에서 Job을 끝낸다
- 이후 join 함수를 만나 코루틴이 종료될 때까지 기다린다
- 참고로 cancel 함수에 각기 다른 예외를 인자로 넣는 방법을 사용하면 취소된 원인을 명확하게 할 수 있다
    - Coroutine을 취소하기 위해서 사용되는 예외는 반드시 CancellationException의 서브 타입이어야 한다. **(Coroutine을 취소하기 위해서 사용되는 예외는 CancellationException이어야 하기 때문)**
    
    <img width="1005" height="199" alt="Image" src="https://github.com/user-attachments/assets/32997fa4-ca37-4ae3-a55d-ebdac38f2e1f" />
    
- cancel이 호출된 뒤 취소 과정이 완료되는 걸 기다리기 위해 join을 사용하는 것이 일반적
    - join을 사용하지 않으면 race condition가 될 수 있다.

> Ex) Race Condition 발생
> 

```kotlin
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

suspend fun main() : Unit =coroutineScope {
    val job=launch {
        repeat(1000) { i->
            delay(100)
            // delay 보다 오래 걸리는 작업 이라고 가정
            Thread.sleep(100)
            println("Printing $i")
        }
    }

    delay(1000)
    job.cancel()
//   join 을 넣을 때와 안 넣었을 때 마지막 부분의 출력 결과가 다르다
//    job.join()
    println("Cancelled Successfully")
}

// 출력 결과
Printing 0
Printing 1
Printing 2
Printing 3
Cancelled Successfully
Printing 4
```

### CancelAndJoin

- kotlinx.coroutines 라이브러리는 cancel과 join을 함께 호출할 수 있는 간단한 방법으로 **cancelAndJoin이라는 확장 함수를 제공**

```kotlin
public suspend fun Job.cancelAndJoin() {
    cancel()
    return join()
}
```

- Job( ) Factory 함수로 생성된 잡은 cancelAndJoin 호출 방법으로 취소될 수 있다
    - **이는 Job에 딸린 수 많은 Coroutine을 한번에 취소할 때 자주 사용**
        
        Ex) 화면에서 뒤로가기를 했을 때 설정 화면에서 시작된 모든 코루틴을 취소하는 경우  etc..
        

```kotlin
import kotlinx.coroutines.Job
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

suspend fun main() : Unit = coroutineScope {
    val job= Job()
    launch(job) {
        repeat(1_000) { i->
            delay(200)
            println("Printing $i")
        }
    }

    delay(1100)
    job.cancelAndJoin()
    println("Cancelled Successfully")
}

// 출력 결과
Printing 0
Printing 1
Printing 2
Printing 3
Printing 4
Cancelled Successfully
```

> Ex) Android에서 코루틴을 취소하는 방법
> 
- 공통의 Scope에 `SupervisorJob()` 팩토리 함수로 Job 생성 + 화면에 띄우기 위해 Main Thread에서 작업을 위한 Dispatcher.Main 설정

```kotlin
class ProfileViewModel : ViewModel() {
		private val scope = 
				CoroutineScope(Dispatchers.Main + SupervisorJob())

		fun onCreate() {
				scope.launch { loadUserData() }

	override fun onCleared() {
			scope.coroutineContext.cancelChildren()
	}
  ...
}
```

## 취소는 어떻게 작동하는가?

- Job이 취소되면 `Cancelling` 상태로 바뀌고 상태가 바뀐 후 첫 번째 중단점에서 `CancellationException` 예외를 던진다

```kotlin
suspen fun main() : Unit = coroutineScope {
	val job = Job()
	launch(job) {
		try {
			repeat(1000) { i ->
				// 1.1초 후 Exception을 던지게 된다
				delay(200)
				println("$i")
			}
		} catch(e: CancellationException) {
			println(e)
			throw e
		}
	}
	
	delay(1100)
	job.cancelAndJoin()
	println("Cancel Finish")	
}
-------------------
0
1
2
3
4
JobCancellationException ~~~
Cancel Finish
```

⭐ 취소된 Coroutine이 단지 멈추는 것이 아닌 예외를 사용해서 취소되는 것

- try ~ catch ~ finally 의 예외 처리처럼 finally Block 안에서 리소스 해제 작업 등을 할 수 있다

## 취소 중 코루틴을 1번 더 호출

- 코루틴이 종료되기 전 `CancellationException` 을 잡고 더 많은 연산을 수행할 수 있고, 코루틴은 모든 자원을 정리할 필요가 있는 한 계속해서 실행될 수 있다
- But. 정리 과정에서 중단하는 것은 허용하지 않는다
    - Job이 이미 `Cancelling` 상태가 되어 새로운 코루틴 시작, 중단은 절대 불가능하다
    - 새로운 코루틴 시작은 무시, 중단을 시도하면 `CancellationException` 발생

> Ex) 취소 중 새로운 코루틴 발생 및 중단 시도
> 

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        try {
            println("Coroutine started")
            // 중단 지점
            delay(200)
            println("Coroutine finished")
        } finally {
            println("Finally")
            // finally Block에 속하지만 새로운 코루틴은 무시
            launch {
                println("Children executed")
            }
            // 예외 발생 (에러로 표면화 되지는 않음)
            delay(1000L)
            println("Cleanup done")
        }
    }
    delay(100)
    job.cancelAndJoin()
    println("Done")
}
----------------
Coroutine started
Finally
Done
```

- 만약 이미 취소가 된 상태에서 중단 함수를 호출하고자 할 땐 `withContext(NonCancellable)` 로 포장하여 취소될 수 없는 Job 객체를 사용하면 Block 내부에서 Active 상태를 유지하여 중단함수를 호출할 수 있다

> Ex) NonCancellable
> 

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        try {
            println("Coroutine started")
            delay(200)
            println("Coroutine finished")
        } finally {
            println("Finally")
            // Active 상태가 유지 된다
            withContext(NonCancellable) {
                launch {
                    println("Children executed")
                }
                delay(1000L)
                println("Cleanup done")
            }
        }
    }
    delay(100)
    job.cancelAndJoin()
    println("Done")
}
---------------------
Coroutine Started
Finally
Children executed
Cleanup done
Done
```

## invokeOnCompletion 메서드

- 자원을 해제하는데 자주 사용되는 방법으로 `Completed`, `Cancelled` 처럼 Job이 마지막 상태에 도달했을 때 호출 될 Handler를 지정하는 방법

<img width="1226" height="184" alt="Image" src="https://github.com/user-attachments/assets/d61c25e1-a3e0-4b44-a42c-f4098443e1d1" />

> Ex) invokeOnCompletion
> 

```kotlin
suspend fun main() : Unit = coroutineScope {
	val job = launch {
		delay(1000)
	}
	job.invokeOnCompletion { exeception : Thorwable? ->
		println("Done")
	}
	
	delay(500)
	job.cancelAndJoin()
}
--------------
Done
```

- exception 예외 종류
    - Job이 예외 없이 끝나면 null
    - 코루틴이 취소 되었다면 `CancellationException`
    - 코루틴을 종료시킨 예외

## 중단될 수 없는 걸 중단하기

⭐ 코루틴 취소는 중단점이 없으면 할 수 없다 

- 대부분의 (중단점 = 취소 가능 지점)

1. `yield()` 메서드 주기적으로 호출
    - yield는 코루틴을 중단하고 즉시 재실행한다
    - yield라는 중단점이 생겨 취소를 포함해 중단 중에 필요한 모든 작업을 할 수 있는 기회가 생긴다
    
    ```kotlin
    suspend fun main(): Unit = coroutineScope {
        val job = Job()
        launch(job) {
            repeat(1_000) { i ->
    		        // 중단점이 아님
                Thread.sleep(200)
                // 중단 지점
                yield()
                println("$i")
            }
        }
        delay(1100)
        job.cancelAndJoin()
        println("Cancelled")
        delay(1000)
    }
    -----------------
    0
    1
    2
    3
    4
    Cancelled
    ```
    
    - yield는 중단 가능하지 않으면서 CPU, 시간 집약적인 연산들이 중단함수 내에 있을 때 사용하는 것이 좋다

1. Job의 상태를 추적하는 방법
    - 코루틴 Builder 내부에서 this는 Builder의 Scope를 참조하고 있고 코루틴 스코프는 coroutineContext 프로퍼티를 가지고 Context를 참조할 수 있다
    - `isActive()` 메서드를 사용해 현재 상태를 확인하여 Active하지 않은 경우 연산을 중단하도록 조건문을 걸면 중단이 가능하다
    
    ```kotlin
    suspend fun main(): Unit = coroutineScope {
        val job = Job()
        launch(job) {
            do {
    	        // 중단 불가능
                Thread.sleep(200)
                println("Printing")
            } while (isActive) // 조건에 따라 중단 가능
        }
        delay(1100)
        job.cancelAndJoin()
        println("Cancelled")
    }
    -----------------
    Printing
    Printing
    Printing
    Printing
    Printing
    Printing
    Cancelled
    ```
    

1. `ensureActive()` 메서드 활용
    - Job이 Active 상태가 아니면 `CancellationException` 을 던진다
    - 비중단 함수로 취소 체크만 한다
    
    ```kotlin
    suspend fun main(): Unit = coroutineScope {
        val job = Job()
        launch(job) {
            repeat(1000) { num ->
                Thread.sleep(200)
                ensureActive()
                println("$num")
            }
        }
        delay(1100)
        job.cancelAndJoin()
        println("Cancelled")
    }
    ----------------
    0
    1
    2
    3
    4
    Cancelled
    ```
    

## suspendCancellableCoroutine

- Continuation 객체를 `CancellableContinuation<T>` 로 Wrapping

```kotlin
suspend fun someTask() = suspendCancellableCoroutine { cont ->
	cont.invokeOnCancellation {
		// 코루틴이 취소되었을 때 행동을 정의
		// 주로 라이브러리 실행을 취소하거나 자원을 해제
	}
}
```

- 잡의 상태를 확인할 수 있고 Continuation을 취소할 때 취소가 되는 원인을 추가적으로 제공 가능

---

## 요약

- 취소는 코루틴의 강력한 기능

⭐ 취소를 적절하게 사용하면 자원 낭비와 메모리 누수를 줄일 수 있다

