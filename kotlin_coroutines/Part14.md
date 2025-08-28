# 공유 상태로 인한 문제

- 코루틴을 사용할 때 같은 시간에 두 개 이상의 스레드에서 함수가 호출된다면 Race Condition 문제가 발생할 수 있다
- 앞서 12장의 Executors 의 single Thread로 제한한다면 공유 상태 문제를 해결할 수 있다

> Ex) 사용자 정보 불러오기, 정보 추가하기
> 

```kotlin
class UserDownloader(
	private val api: NetworkService
) {
		private val users = mutableListOf<User>()
		
		fun download() : List<User> = users.toList()
		
		suspend fun fetchUser(id: Int) {
			val newUser = api.fetchUser(id)
			// user가 변경될 여지가 있는데 공유 상태에 대한 대비가 되어있지 않은 상태
			users.add(newUser)
		}
}
```

> Ex) Race Condition 발생 (연산의 일부가 반영되지 않는다)
> 

```kotlin
var counter = 0

suspend fun massiveRun(action: suspend () -> Unit) =
    withContext(Dispatchers.Default) {
        repeat(1000) {
	        // 1000개의 코루틴을 띄우고 action을 1000번 반복
            launch {
                repeat(1000) { action() }
            }
        }
    }

fun main() = runBlocking {
    massiveRun {
        counter++
    }
    println(counter)
}
-----------------
1000000보다 작은 값 출력
```
</br>

## 동기화 Blocking

- Java 진영에서는 `synchronized Block`이나 `동기화된 Collection`을 사용해서 공유 상태 문제를 해결할 수 있다

```kotlin
var counter = 0

fun main() = runBlocking {
	val lock = Any()
	// massiveRun 메서드는 위와 동일
	massiveRun {
		synchronized(lock) {
			counter++
		}
	}
	println(counter)
}
---------------
1000000
```

- synchronized 블럭 내에서 중단 함수를 사용할 수 없다 (suspend 함수가 아닌 inline Function이기 때문)
- synchronized 블럭에서 코루틴이 자기 차례를 기다릴 때 스레드를 Block 한다
- (Main)Thread를 막을 수 있다는 점과 Thread는 비용이 큰 자원이라는 점에서 잘 사용하지 않고 코루틴에 특화된 방법을 사용한다

</br>

## 원자성

- Java에서 지원하는 다양한 원자 값을 사용하여 빠른 연산과 스레드 안전을 보장할 수 있다
- 원자성 연산은 Lock 없이 저수준으로 구현되어있어 효율적이고 사용하기 편리하다
- 접두어로 `Atomic`이 붙어 Atomic[자료형] 형태로 java.util.concurrent 패키지에 내장 되어있다

> Ex) AtomicInteger를 활용한 공유 상태 문제 해결
> 

```kotlin
private var counter = AtomicInteger()

fun main() = runBlocking {
	massiveRun {
		counter.incrementAndGet()
	}
	// counter.get()을 호출해버리면 원자성이 깨진다
	println(counter)
}
---------------
1000000
```

- `incrementAndGet()`, `getAndSet()`, `compareAndSet()` 같은 **하나의 메서드 호출은 원자적**이지만 여러 개의 연산을 조합했을 때는 **전체가 원자적이지 않다**는 점을 유의해야한다
- 일반적인 Class에 원자성을 적용하고 싶으면 `AtomicReference` 형태로 래핑하면 된다

</br>

## 싱글 Thread로 제한된 디스패처 (가장 쉬운 방법)

- 병렬성을 하나의 스레드로 제한하면 공유 상태 문제를 해결할 수 있다

```kotlin
val dispatcher=Executors.newSingleThreadExecutor()
		.asCoroutineDispatcher()
		

val dispatcher=Dispatchers.Default
	.limitedParallelism(1)
```

> Ex) 싱글 스레드로 공유 상태 문제 해결
> 

```kotlin
val dispatcher = Dispatchers.IO.limitedParallelism(1)

var counter = 0

fun main() = runBlocking {
	massiveRun {
		withContext(dispatcher) {
			counter++
		}
	}
	println(counter)
}
--------------
1000000
```

### 디스패처 활용 방법

1. 코스 그레인드 스레드 한정 방식
    1. Dispatchers를 싱글 스레드로 제한한 withContext로 전체 함수를 Wrapping 하는 방법
    2. 사용하기 쉽고 공유 상태 문제를 쉽게 해결할 수 있지만 멀티 스레딩의 이점을 누리지 못한다
    
    ```kotlin
    private val dispatcher = Dispatchers.IO.limitedParallelism(1)
    
    suspend fun download() : List<User> = 
    	// 코스 그레인드 방법
    	withContext(dispatcher) {
    		users.toList()
    	}
    ```
    

1. 파인 그레인드 스레드 한정 방식
    1. 상태를 변경하는 구문들만 Wrapping 한다
    2. 코스 그레인드 방식에 비해 번거롭지만 Critical Section이 아닌 부분이 Blocking 되거나 CPU 집약적인 경우 더 나은 퍼포먼스를 보인다
    
    ```kotlin
    private val dispatcher = Dispatchers.IO.limitedParallelism(1)
    
    suspend fun fetchUser(id: Int) {
    	// no-critical-section
    	val newUser = api.fetchUser(id)
    	// 공유 상태 문제가 발생할 수 있는 임계 구역에만 Wrapping
    	withContext(dispatcher) {
    		users += newUser
    	}
    }
    ```
</br>   

## Mutex

- 1칸만 있고 열쇠가 있는 화장실처럼 lock이 호출되면 점유하고 있는 코루틴이 unlock를 호출할 때까지 나머지 코루틴이 중단을하고 Queue에 들어가 대기한다
- synchronized와 달리 mutex는 스레드를 blocking 하는 것이 아닌 코루틴을 중단시킨다

```kotlin
suspend fun main() = coroutineScope {
    repeat(5) {
        launch {
            delayAndPrint()
        }
    }
}

val mutex = Mutex()

suspend fun delayAndPrint() {
		// 잠금
    mutex.lock()
    delay(1000)
    println("Done")
    // 작업이 끝나면 반드시 호출해줘야 한다
    mutex.unlock()
}
------------------
(1초)
Done
위 과정이 5번 반복된다
```

- 하지만 lock과 unlock을 직접 사용하면 두 함수 사이에서 예외가 발생할 경우 열쇠를 돌려받을 수 없어 DeadLock 현상이 발생하게 된다 (아무도 일을 못하는 상태)
- lock으로 시작해서 finally Block에서 unlock를 호출해서 무조건 해제를 보장하는 witLock 메서드를 사용하면 예외가 발생하더라도 DeadLock 상태에 빠지지 않을 수 있다

<img width="691" height="304" alt="Image" src="https://github.com/user-attachments/assets/e06d03bc-9c4c-4c88-9097-6ffc2a9a2447" />

```kotlin
massiveRun {
	mutex.withLock {
		counter++
	}
}
```

- mutex도 적절히 사용하는 것이 중요하다
    - 코루틴이 Lock을 2번 통과할 수 없다 (DeadLock 발생)
        
        ```kotlin
        suspend fun main() {
            val mutex = Mutex()
            println("Started")
            mutex.withLock {
                mutex.withLock {
                    println("Will never be printed")
                }
            }
        }
        ------------
        Started
        ... 프로그램 죽지 않음
        ```
        
    - 코루틴이 중단되었을 때 Mutex를 풀 수 없다
        - 전체 함수를 mutex로 Wrapping 한다면 Lock을 풀 수 없기 때문에 지양해야한다
        
        ```kotlin
        class MessagesRepository {
            private val messages = mutableListOf<String>()
            private val mutex = Mutex()
        
            suspend fun add(message: String) = mutex.withLock {
                delay(1000) // 장시간 소요 작업이라 가정
                messages.add(message)
            }
        }
        
        suspend fun main() {
            val repo = MessagesRepository()
            val timeMillis = measureTimeMillis {
                coroutineScope {
        		        // 5개의 코루틴 생성
                    repeat(5) {
                        launch {
                            repo.add("Message$it")
                        }
                    }
                }
            }
            println(timeMillis) 
        }
        -------------------
        5000 이상의 값이 출력된다
        ```
        
        - singleThread로 제한하고 mutex로 wrapping한 것을 withContext(dispatcher)로 호출하면 약 1초가 걸리게 된다 (Thread륾 막지 않는다는 것을 알 수 있다)

</br>

## 세마포어

- Mutex는 하나의 열쇠를 가지고 있어 하나의 접근만을 허용한다
- 세마포어는 여러개의 접근을 허용한다 (by. 세마포어 count) ⇒ 여러 개가 들어와 동시성 문제는 여전히 발생 가능
- 세마포어는 공유 상태로 인해 생기는 문제를 해결할 수는 없지만 동시 요청 처리 수를 제한할 수 있다

```kotlin
suspend fun main() = coroutineScope {
    val semaphore = Semaphore(2)

    repeat(5) {
        launch {
            semaphore.withPermit {
                delay(1000)
                print(it)
            }
        }
    }
}
-------------------
// 호출할 때마다 다른 결과가 나온다 (묶음 수만 유지)
10
(1초)
23
(1초)
4
```

---

## 요약

- 공유 상태를 변경할 때 발생할 수 있는 충돌을 피하기 위해 코루틴을 다루는 다양한 방법 파악
- 가장 많이 사용되는 기법은 싱글 스레드로 제한한 Dispatcher를 만들어서 다루는 것이 좋다
- 동기화가 필요한 부분만 래핑하는 파인 그레인드 방식, 전체 함수를 래핑하는 코스 그레인드 방식
- Mutex
- 결국은 OS 스케쥴링을 다룰 때 배운 내용이 코틀린 문법에서도 적용된 것을 볼 수 있다
