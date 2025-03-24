## 중단은 어떻게 작동할까?

### 중단 함수는 코틀린 코루틴의 핵심

⭐ **중단이 가능하다는 것이 코루틴의 다른 모든 개념의 기초가 되는 필수적 요소이다**

- 코루틴 중단이란? 
→ 실행을 중간에 멈추는 것
- 코루틴은 중단 되었을 때 **Continuation 객체**를 반환 (이 객체를 이용하면 멈췄던 곳에서 다시 코루틴을 실행할 수 있다)
→ 게임의 체크포인트 느낌

### 코루틴이 강력한 이유

⭐ Thread는 저장이 불가능하고 멈추는 것만 가능하다

- 코루틴은 중단했을 때 어떤 자원도 사용하지 않는다
- 코루틴은 다른 스레드에서 실행할 수 있고 Continuation 객체는 직렬화, 역직렬화 가능하며 다시 실행될 수 있다

## 재개

### 작업을 재개하기 위해서는 코루틴이 필요하다

- 코루틴은 runBlocking이나 launch와 같은 **코루틴 빌더**를 통해 생성 가능

⭐ 중단 함수 : 코루틴을 중단할 수 있는 함수 (suspend 키워드가 붙는 이유)

**→ 중단 함수가 반드시 다른 중단 함수(코루틴)에 의해 호출되어야 한다**

EX.1

```kotlin
suspend fun main() {
    println("Before")

    suspendCoroutine<Unit> {}

    println("After")
}
//출력 결과
Before
```

- 위 코드에서 suspendCoroutine에 의해 중단 되고 After는 호출되지 않는다
    - suspendCoroutine : Continuation 객체를 반환하고 현재 코루틴을 멈춘다

```kotlin

suspend fun main() {
    println("Before")

    suspendCoroutine<Unit> { continuation ->
				println("Before too")
    }

    println("After")
}
// 출력 결과
Before
Before too
```

- suspendCoroutine의 람다 표현식 내부는 코루틴이 중단되기 전에 실행된다.
- suspendCoroutine 메서드는 let, apply 처럼 곧바로 호출된다 (람다 내에서 continuation 객체 사용 가능)
    - suspendCoroutine이 호출된 뒤에는 이미 중단되어 continuation 객체를 사용할 수 없기 때문에 람다 표현식이 함수의 인자로 들어가 중단되기 전 실행된다. (람다 내부에서만 호출 가능)
    - 이때 람다 함수는 continuation 객체를 저장한 뒤 **코루틴을 다시 실행할 시점을 결정하기 위해 사용**
    → 이 객체를 이용해 코루틴을 중단한 후 곧바로 실행할 수 있다.

```kotlin
suspend fun main() {
    println("Before")

    suspendCoroutine<Unit> { continuation ->
        continuation.resume(Unit) // 멈췄던 것을 재개해라
    }

    println("After")
}
//출력 결과
Before
After
```

- 재개하는 방식은 2가지가 존재했으나 resumeWith 함수 하나만 남았다 (version 1.3 이후)
    
    ```kotlin
    inline fun <T> Continuation<T>.resume(value: T) : Unit =
    	resumeWith(Result.success(value))
    
    inline fun <T> Continuation<T>.resumeWithException(
    		exception: Throwable
    ) : Unit = resumeWith(Result.failure(exception))
    
    // 두 함수는 resumeWith를 사용하는 표준 라이브러리의 확장 함수가 되었다
    ```
    

EX.2

```kotlin
suspend fun main() {
    println("Before")

    suspendCoroutine<Unit> { continuation ->
        thread {
            println("Suspended")
            Thread.sleep(1000L)
            continuation.resume(Unit)
            println("Resumed")
        }
    }

    println("After")
}
// 출력결과
Before
Suspended
(1초 정적)
After
Resumed
```

- 코루틴이 중단되는 람다 내부에서 Thread를 생성하여 1초 간 멈춘 후 다시 스레드가 재개하는 코드이다
    - 위 코드에서 1초 뒤 사라지는 스레드는 불필요해 보인다 (생성 비용이 커 비효율적)

### ScheduledExecutorService (알람 시계)

```kotlin
private val executor=
    Executors.newSingleThreadScheduledExecutor {
        Thread(it,"scheduler").apply { isDaemon=true }
    }

suspend fun main() {
    println("Before")

    suspendCoroutine<Unit> { continuation ->
        executor.schedule({
            continuation.resume(Unit)
        },1000,TimeUnit.MILLISECONDS)
    }
    println("After")
}
// 출력 결과
Before
(1초 delay)
After
```

- ScheduledExecutorService를 사용한 방식

```kotlin
private val executor=
    Executors.newSingleThreadScheduledExecutor {
        Thread(it,"scheduler").apply { isDaemon=true } // Thread가 생성 된다.
    }

suspend fun delay(timeMillis: Long) : Unit=
    suspendCoroutine { continuation ->
        executor.schedule({
            continuation.resume(Unit)
        }, timeMillis,TimeUnit.MILLISECONDS)
    }

suspend fun main() {
    println("Before")
    delay(1000L)
    println("After")
}
```

→ 위의 방식을 delay라는 메서드를 따로 빼서 작성한 방식

**⭐ 스레드를 사용하지만 delay 함수를 사용하는 모든 코루틴의 전용 스레드가 된다**

## 값으로 재개하기

- suspendCoroutine의 형태

```kotlin
suspendCoroutine <T> { continuation: Continuation<T> ->  }
```

- Continuation 객체로 반환 될 타입을 지정할 수 있다 <T> : 제네릭 타입
- resume을 통해 반환 되는 값은 **반드시** 지정된 타입과 같은 타입이어야 한다

```kotlin
suspend fun main() {
    val i: Int= suspendCoroutine<Int> { cont->
        cont.resume(42) // Int 타입 반환
    }
    println(i)

    val str: String= suspendCoroutine<String> { cont->
        cont.resume("AAA") // String 타입 반환
    }
    println(str)
}
```

### suspendCoroutine 실제 사용 예시

```kotlin
suspend fun main() {
    println("Before")
    val user= suspendCoroutine<User> { continuation -> 
        requestUser { user -> 
            cont.resume(user)
        }
    }

    println(user)
    println("After")
}

// 변경 후 (중단함수를 호출 하는 방식)
suspend fun requestUser() : User {
    return suspendCoroutine<User> { continuation -> 
        requestUser { user-> 
            continuation.resume(user)
        }
    }
}

suspend fun main() {
    println("Before")
    val user= requestUser()
    println(user)
    println("After")
}
```

- 하단의 방식은 suspendCoroutine을 main에서 직접 호출하지 않고 중단 함수를 만들어 호출함

### 예외로 재개하기

- API가 데이터를 넘겨주는 대신 문제가 발생하는 경우
- 서비스가 종료되거나 에러로 응답이 오는 경우

⭐ 데이터를 반환할 수 없으므로 **코루틴이 중단된 곳에서 예외를 발생시켜야 한다**

- **resumeWithException**이 호출되면 중단된 지점에서 인자로 넣은 예외를 던진다

EX.3

```kotlin
 class MyException : Throwable("Just an exception")

suspend fun main() {
    try {
        suspendCoroutine<Unit> { cont->
            cont.resumeWithException(MyException()) // 예외 발생 시킴

        }
    }
    catch (e:MyException) {
        println("Caught")
    }
}
// 출력 결과
Caught
```

### 네트워크 관련 예

```kotlin
// 사용자 목록 불러오기
suspend fun requestUser(): User {
    return suspendCancellableCoroutine<User> { cont ->
        requestUser { resp ->
            if (resp.isSuccessful) {
                cont.resume(resp.data) // 사용자 정보 반환
            } else {
                val e=ApiException(
                    resp.code,
                    resp.message
                )
                cont.resumeWithException(e) // 에러 시 중단 지점에서 Exception 발생 
            }
        }
    }
}

// 뉴스 목록 불러오기
suspend fun requestNews() : News {
    return suspendCancellableCoroutine<News> { cont->
        requestNews(
            onSuccess= { news-> cont.resume(news)}, // 성공 시 얻은 데이터 반환
            onError= {e -> cont.resumeWithException(e)} // 중단 지점에 Exception 발생
        )
    }
}
```

### 함수가 아닌 코루틴을 중단시킨다

- 함수가 아닌 코루틴을 중단시킨다는 점을 명심
- 중단 함수는 코루틴이 아니고 코루틴을 중단할 수 있는 함수 (suspend 키워드가 붙은 함수)

```kotlin
var continuation: Continuation<Unit>?=null

suspend fun suspendAndSetContinuation() {
    suspendCoroutine<Unit> { cont ->
        continuation=cont
    }
}

suspend fun main() {
    println("Before")
    suspendAndSetContinuation()
    continuation?.resume(Unit) // 호출되지 않는다

    println("After")
}
// 출력 결과
Before

// 1초 뒤 다른 코루틴이 재개하는 상황
var continuation: Continuation<Unit>?=null

suspend fun suspendAndSetContinuation() {
    suspendCoroutine<Unit> { cont ->
        continuation=cont
    }
}

suspend fun main() = coroutineScope { // 코루틴 빌더
    println("Before")

    launch {
        delay(1000L)
        continuation?.resume(Unit) // 다른 스레드나 코루틴 내부에서 재개 해야한다
    }
    
    suspendAndSetContinuation()
    println("After")
}
// 출력 결과
Before
(1초 정적)
After
```

- 위 2번째 코드에서 suspendAndSetContinuation() 메서드 호출 부를 주석처리하면 delay가 걸리지 않고 Before After가 연달아 출력된다
- suspendAndSetContinuation()와 launch { } 부분의 순서를 바꾸면 위의 상황처럼 Before가 출력되고 코루틴이 중단된 상태로 존재
- 2번째 코드에서 coroutineScope를 전역으로 잡지 않아도 된다
- 2번째 코드 실행 원리를 알봐야겠다

---

## 📎 읽어보기

**suspendCoroutine**

- [suspendCoroutine - Kotlin Programming Language (kotlinlang.org)](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/suspend-coroutine.html)

**suspendCancellableCoroutine**

- [suspendCancellableCoroutine (kotlinlang.org)](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/suspend-cancellable-coroutine.html)
