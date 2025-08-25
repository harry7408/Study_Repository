# 코루틴 스코프 함수

## 여러 데이터를 동시에 얻어오는 상황

1. 중단 함수에서 중단 함수를 호출하는 방법
    1. But 작업이 동시에 진행되지 않음
    2. 동시에 실행하려면 async 빌더로 감싸야 하나 GlobalScope를 사용해야 한다
    
    > Ex) 2개의 중단 함수 호출 (병렬 실행 X)
    > 
    
    ```kotlin
    suspend fun getUserData() : UserProfileData {
    	val user = getUserInfo()
    	val notifications = getNotifications()
    	
    	return UserProfileData(
    		user = user,
    		notifications = notifications,
    	)
    }
    ```
    
    > Ex) async Builder로 병렬 실행은 갖춘 상태
    > 
    
    ```kotlin
    suspend fun getUserData() : UserProfileData {
    	// Global Scope로 감싼 형태 (사용 지양)
    	val user = GlobalScope.async { getUserInfo() }
    	val notifications = GlobalScope.async { getNotifications() }
    	
    	return UserProfileData(
    		user = user.await(),
    		notifications = notifications.await(),
    	)
    }
    ```
    
    > 💡 GlobalScope
    > 
    
   <img width="863" height="207" alt="Image" src="https://github.com/user-attachments/assets/54b55c67-dfef-46b7-a849-110c0231782c" />
    
    ```kotlin
    object GlobalScope : CoroutineScope {
    	override val coroutineContext: CoroutineContext
    		get() = EmptyCoroutineContext
    }
    
    GlobalScope에서 async를 호출하면
    - 부모 코루틴과 아무런 관계가 없다
      -> Application 전역에서 살아남는 스코프 (메모리 누수 위험으로 사용하지 않는 것을 권장)
      -> 코루틴이 취소 될 수 없다
      -> 부모로 부터 Scope를 상속받지 않는다 (구조화된 동시성도 갖출 수 없다)
      -> 코루틴을 단위테스트 하는 도구가 동작하지 않아 함수를 테스트하기 어렵다
    ```
    

1. Scope를 인자로 넘기는 방법 or 확장 함수로 만드는 방법
    1. 취소가 가능하고 단위 테스트를 추가할 수는 있다
    2. 스코프가 함수에서 함수로 전달되기 때문에 예상치 못한 부작용이 발생할 수 있다 
        1. async에서 예외 발생 시 모든 Scope가 닫힌다
        2. 함수에서 cancel 메서드를 사용해 스코프를 취소 하는 등의 조작이 가능해진다
    
    > Ex) Scope를 인자로 받는 경우 + 확장함수 활용
    > 
    
    ```kotlin
    suspend fun getUserData(
    	scope : CoroutineScope,	
    ) : UserProfileData {
    	val user = scope.async { getUserInfo() }
    	val notifications = scope.async { getNotifications() }
    	
    	return UserProfileData(
    		user = user.await(),
    		notifications = notifications.await(),
    	)
    }
    
    ---------------------
    // 확장함수
    suspend fun CoroutineScope.getUserProfile() : UserProfileData {
    	val user = async { getUserData() }
    	val notifications = async { getNotifications() }
    	
    	return UserProfileData(
    		user = user.await(),
    		notifications = notifications.await(),
    	)
    }
    
    ```
    

## CoroutineScope

- 코루틴 빌더가 만든 동**시성을 안전하게 묶고 관리하기 위해** 코루틴 안에서 호출되는 중단 함수
- 스코프를 시작하는 중단 함수이며 인자로 들어온 함수가 **생성한 값**을 반환
    
    ```java
    suspend fun <R> coroutineScope(
    	block: suspend CoroutineScope.()-> R
    ): R
    ```
    
    - coroutineScope 의 본체는 리시버 없이 바로 호출된다 (coroutineScope를 호출한 코루틴 내에서 바로 동작한다)
    - coroutineScope 내부의 동작이 모두 끝날 때까지 coroutineScope를 호출한 코루틴을 중단시킨다 (자식들 작업이 완료되어야 반환)
    
    > Ex)
    > 
    
    ```kotlin
    fun main() = runBlocking {
        val a = coroutineScope {
            delay(1000)
            10
        }
    
        println("a is calculated")
    
        val b = coroutineScope {
            delay(1000)
            20
        }
        println(a) // 위치를 "a is calculated" 위로 바꾸면 먼저 호출된다
        println(b)
    }
    ------------------
    // 위의 coroutineScope가 순차적으로 실행된다
    a is calculated
    10
    20
    ```
    
    - coroutineScope는 단순히 중단함수를 호출한 것과 같다 (스코프 생성 + 자식 완료까지 대기의 역할)
    - 새로운 Job을 만들지 않고 부모의 coroutineContext를 상속받는다 (Job 오버라이딩)
    - **고로 구조화된 동시성을 따른다 (모든 자식을 기다린다 + 부모가 취소되면 자식들 모두 취소)**
    
    > Ex) coroutineScope의 동작 및 coroutineContext 상속
    > 
    
    ```kotlin
    suspend fun longTask() = coroutineScope {
        launch {
            delay(1000)
            val name = coroutineContext[CoroutineName]?.name
            println("[$name] Finished task 1")
        }
    
        launch {
            delay(2000)
            val name = coroutineContext[CoroutineName]?.name
            println("[$name] Finished task 2")
        }
    }
    
    fun main() = runBlocking(CoroutineName("Parent")) {
        println("Before")
        // 모든 자식들이 끝날 때까지 대기
        longTask()
        println("After")
    }
    ---------------------
    Before
    [Parent] Finished task 1
    [Parent] Finished task 2
    After
    ```
    
    - longTask() 메서드에서 coroutineScope가 모든 자식이 끝날 때까지 종료되지 않았다
    - CoroutineName 컨텍스트가 상속되어 출력되는 것을 볼 수 있다
    
    > Ex) coroutineScope가 취소되는 상황
    > 
    
    ```kotlin
    suspend fun longTask() = coroutineScope {
        launch {
            delay(1000)
            val name = coroutineContext[CoroutineName]?.name
            println("[$name] Finished Task 1")
        }
    		
    		// 1.5초에 부모 코루틴에서 cancel이 호출되어 자식도 취소되고 호출되지 않았다
        launch {
            delay(2000)
            val name = coroutineContext[CoroutineName]?.name
            println("[$name] Finished Task 2")
        }
    }
    
    fun main() = runBlocking {
        val job = launch(CoroutineName("Parent")) {
            longTask()
        }
        delay(1500)
        job.cancel()
    }
    -------------------
    [Parent] Finished Task 1
    ```
    
    - job.cancel()이 호출되어 코루틴이 취소될 때 자식 코루틴 모두 취소되는 것을 볼 수 있다

⭐ Coroutine 빌더와 달리 coroutineScope나 Scope에 속한 자식에서 예외가 발생하는 경우 다른 모든 자식이 취소되고 예외가 다시 던져진다

> Ex) 예외
> 

```kotlin
suspend fun work(): Unit = coroutineScope {
    val a = launch {
        delay(200)
        println("A done")           
    }
    val b = launch {
        delay(100)
        throw RuntimeException("B failed") // 예외 발생 지점
    }
}

fun main() = runBlocking {
    try {
        work()                       // 예외가 다시 던져진다
    } catch (e: Throwable) {
        println("caught: ${e.message}") // 잡히는 부분
    }
}
-----------
caught: B failed
```

- 예외가 다시 던져질 수 있어 coroutineScope를 호출한 Call Site에서 try ~ catch 구문으로 잡을 수 있다

⭐ **coroutineScope 사용의 주 목적은 구조화된 동시성을 위해 코루틴이 실행될 수 있는 Scope를 만들고 자식 코루틴들을 유연하게 관리하기 위함**

- 병렬 작업을 수행할 경우 coroutineScope를 열어 안에 코루틴 빌더를 만들어서 관리
- Main 함수 본체를 Wrapping할 때 사용
- 새로운 Scope를 만들고 싶을 때 사용

## 코루틴 Scope 함수 종류

1. withContext : 코루틴 컨텍스트를 바꿀 수 있는 coroutineScope
2. supervisorScope : Job대신 SupervisorJob을 사용
3. withTimeout : 시간 제한이 있는 coroutineScope

💡 Coroutine Builder와 Coroutine Scope의 차이점

- runBlocking 제외

| 코루틴 빌더 | 코루틴 스코프 |
| --- | --- |
| launch, async, produce | coroutineScope, supervisorScope, withContext, withTimeout |
| CoroutnieScope의 Extension Function | 중단 함수 |
| CoroutnieScope 리시버의 코루틴 Context를 사용 | 중단 함수의 Continuation 객체가 가진 코루틴 컨텍스트랄 사용 
(부모로부터 Context를 상속 받는다) |
| 예외는 Job을 통해 부모로 전파 | 일반 함수와 같은 방식으로 예외를 던진다 
(Call Site에서 예외가 던져짐) |
| 비동기인 Coroutine을 시작 
(동시성을 위해서는 코루틴 빌더를 만들어야 한다) | 코루틴 빌더가 호출된 곳에서 코루틴을 시작한다 
(코루틴 시작점인 Builder 내부에서 호출되는 중단함수일 뿐 동시성과는 무관하다) |

## withContext

<img width="892" height="44" alt="Image" src="https://github.com/user-attachments/assets/e04f784a-3dc3-4b61-be79-79de07655b74" />

- coroutineScope와 비슷하나 Scope의 Context를 변경할 수 있다
- `withContext(~~)`에 인자로 컨텍스트를 제공하면 부모 Scope의 컨텍스트를 대체한다

> Ex) 컨텍스트 변경 (CoroutineName)
> 

```kotlin
fun CoroutineScope.log(text: String) {
    val name = this.coroutineContext[CoroutineName]?.name
    println("[$name] $text")
}

fun main() = runBlocking(CoroutineName("Parent")) {
    log("Before")

    withContext(CoroutineName("Child 1")) {
        delay(1000)
        log("Test 1")
    }

    // 순차적으로 실행
    withContext(CoroutineName("Child 2")) {
        delay(1000)
        log("Test 2")
    }
    log("After")
}
-------------------
[Parent] Before
[Child 1] Test 1
[Child 2] Test 2
[Parent] After
```

- 기존 스코프와 컨텍스트가 다른 코루틴 스코프를 설정하고자 할 때 사용 → Android에서 특히 Main Thread와 I.O Thread를 분리하기 위해 Dispatcher와 자주 사용되는 중단함수이다
    
    > Ex) UI와 백그라운드 작업 분리
    > 
    
    ```kotlin
    launch(Dispatchers.Main) {
    	view.showProgress()
    	
    	withContext(Dispatchers.IO) {
    		tempRepository.saveData(data)
    	}
    	// 작업이 끝나면 프로그래스 안보이도록 숨기기
    	view.hideProgress()
    }
    ```
    

## supervisorScope

<img width="683" height="40" alt="Image" src="https://github.com/user-attachments/assets/e38cc493-304b-4e9c-bac8-9c05d11b7fbd" />

- coroutineScope와 유사하나 SupervisorJob으로 Job을 오버라이딩하여 자식 코루틴이 예외를 던지더라도 취소되지 않는다
    - `withContext(SupervisorJob())`으로 대체 가능할 것 같지만 그렇지 않다
- 다른 작업이 중단되어도 영향을 받지 않아야 할 때 주로 사용된다 → 서로 독립적인 작업을 할 때

> Ex) supervisorScope 활용
> 

```kotlin
fun main() = runBlocking {
    println("Before")

    supervisorScope {
        launch {
            delay(1000)
            throw Error()
        }

        launch {
            delay(2000)
            println("Done")
        }
    }
    println("After")
}
-----------------
Before
(1초)
Exception in thread 'main' ~
(1초)
Done
After
```

- 만약 supervisorScope 내에서 async Builder를 사용한다면 예외가 부모로 전파되는 것을 막기 위해 추가적인 예외 처리가 필요하다
    
    → await를 호출하는 부에서 다시 예외를 던지기 때문에 try ~ catch 블럭으로 await 호출을 Wrapping 해줘야 전파되지 않는다
    

## withTimeout

<img width="802" height="47" alt="Image" src="https://github.com/user-attachments/assets/edfdbeba-dc66-453c-9629-58031513a7a3" />

- withTimeout의 인자로 들어온 람다식을 실행할 때 제한시간을 둘 수 있는 중단함수
- 코루틴이 실행될 수 있는 Scope를 만들고 값을 반환한다 (Expression으로 마지막 표현식이 반환)
- 실행 시간이 너무 오래걸리면 본문은 취소되고 `TimeoutCancellationException`을 던진다
    - CancellationException의 Sub 타입이므로 코루틴 빌더 내부에서 TimeoutCancellationException 발생 시 해당 코루틴만 취소되고 다른 자식 코루틴에게 영향을 주지 않는다

> Ex) TimeoutCancellationException 발생 (1.5초 제한이 걸렸는데 본문이 2초가 걸리는 상황)
> 

```kotlin
suspend fun test() : Int = withTimeout(1500) {
    delay(1000)
    println("Still Working")
    delay(1000)
    println("Done")
    42
}

suspend fun main() = coroutineScope {
    try {
        test()
    } catch (e: TimeoutCancellationException) {
        println("Cancelled")
    }
    delay(1000)
}
```

- API 호출 시 withTimeout을 걸어 제한을 두면 테스트할 때 매우 유용하게 사용할 수 있다
    - runTest 내부에서 사용 시 가상의 시간으로 작동하게 된다
    
    ```kotlin
    class Test {
     @Test // 예외 발생 안하는 테스트
     fun testTime() = runTest {
    	 withTimeout(1000) {
    		 ...
    		 delay(900)
    	 }
     }
     
     @Test(expected = TimeoutCancellationException::class)
     fun testTime2() = runTest {
    	 withTimeout(1000) {
    		 ...
    	   delay(1100) // Timeout 예외 발생
    	 }
     }
     
    }
    ```
    
- withTimeout의 완화된 형태인 withTimeoutOrNull은 예외를 던지지 않고 타임아웃이 발생하면 람다식이 취소되고 null을 반환한다
    - wrapping 함수에서 걸리는 시간이 너무 길 때 null 값을 통해 무언가 잘못 되었음을 알리는 용도로 활용된다 (Network로부터 응답을 받지 못했음을 알리는 등)
    

## 코루틴 스코프 함수 연결하기

- 서로 다른 코루틴 스코프 함수의 두 가지 기능이 모두 필요하다면 한 코루틴 스코프 함수에서 다른 기능을 가지는 코루틴 스코프 함수를 호출하면 된다
- 각 coroutineScope 함수들은 특정한 성질을 가지기 때문에 상황에 따라 적절하게 사용하는 것이 중요하다

> Ex) Timeout과 컨텍스트 변경이 필요한 상황 (1초 계산기)
> 

```kotlin
suspend fun calculateAnswerOrNull() : User? = 
	// Context 변경 (cpu 집약적 연산 요구)
	withContext(Dispatchers.Default) {
		// 1초의 시간 제한
		withTimeoutOrNull(1000) {
				calculateAnswer()
		}
	}
```

## 추가적인 연산

- 핵심 동작에 영향을 주지 않는 추가적인 연산이 있을 경우 또 다른 Scope를 만들어서 작업을 시작하는 것이 좋다
- 이를 생성자를 통해 주입한다면 단위테스트 추가 및 Scope를 편리하게 사용할 수 있다

---

## 요약

- 코루틴 스코프 함수는 모든 중단 함수에서 사용될 수 있으므로 유용하다
- 람다식 전체를 Wrapping 할 때 주로 사용된다
- 코루틴 스코프 함수는 코루틴 생태계의 중요한 요소 중 하나이므로 각 중단함수들의 특징 및 기능을 잘 파악하고 있는 것이 좋다
