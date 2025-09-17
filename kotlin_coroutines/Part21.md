# Flow 만들기

- Flow도 코루틴과 유사하게 어디선가 시작되어야 한다
- Flow를 시작하는 다양한 방법이 있다

</br>

## 원시 값을 가지는 플로우

- 플로우가 어떤 값을 가져야 하는지 정의하는 `flowOf` 함수 활용
- 값이 없는 Flow가 필요할 때는 `emptyFlow()` 활용
    
    > Ex) flowOf
    > 
    
    ```kotlin
    suspend fun main() {
    	flowOf(1,2,3,4,5)
    		.collect {
    			print(it)
    		}
    }
    -----------
    12345
    ```
    
    > Ex) Empty Flow
    > 
    
    ```kotlin
    suspend fun main() {
        emptyFlow<Int>()
            .collect { print(it) } 
    }
    ----------
    아무것도 출력되지 않는다
    ```
    
</br>   

## 컨버터

- Iterable, Iterator, Sequence등을 Flow로 바꾸는 방법
- `asFlow()` 함수를 사용하여 즉시 사용 가능한 원소들의 Flow를 만들 수 있다
    
    > Ex) List를 flow로 변환
    > 
    
    ```kotlin
    suspend fun main() {
        listOf(1, 2, 3, 4, 5)
            .asFlow()
            .collect { print(it) } 
    }
    -----------
    12345
    ```
    
    - set, sequence 등도 asFlow()를 사용하여 Flow로 컨버팅 할 수 있다

</br>

## 함수를 Flow로 바꾸기

- 중단 함수를 Flow로 변환할 수 있다 (중단 함수의 결과가 Flow의 유일한 값이 됨)
- suspend() → T 함수형의 확장 함수인 asFlow()를 사용하면 변환 가능하다

  <img width="712" height="253" alt="Image" src="https://github.com/user-attachments/assets/b8372226-fcd3-423a-92e8-a4b305c5c7ed" />

> Ex) suspend Function to Flow
> 

```kotlin
suspend fun main() {
    val function = suspend {
        delay(1000)
        "UserName"
    }

    function.asFlow()
        .collect { println(it) }
}
-------------
(1초)
UserName
```

- 일반 함수를 Flow로 변경하고 싶을 때는 함수 참조값 필요

> Ex) normal Function to Flow
> 

```kotlin
fun getUserName(): String {
    return "UserName"
}

suspend fun main() {
    ::getUserName
        .asFlow()
        .collect { println(it) }
}
-----------
UserName
```

</br>

## Flow와 Reactive Stream

- RxJava와 같은 리액티브 스트림을 활용하고 있을 때 Flow로 쉽게 전환할 수 있다 (kotlinx-coroutines-reactive 라이브러리 활용)
- kotlinx-coroutines-(reactor, rx3) 등의 라이브러리를 사용하면 역으로도 변환 가능하다
    
    > Ex) Flow와 Reactive Stream 간 변환
    > 
    
    ```kotlin
    // Reactive Stream을 Flow로 변환
    suspend fun main() = coroutineScope {
    	Flux.range(1, 5).asFlow()
    		.collect { print(it) }
    		
    	Flowable.range(1, 5).asFlow()
    		.collect { print(it) }
    	
    	Observable.range(1, 5).asFlow()
    		.collect { print(it) }
    }
    
    // 역으로 변환
    suspend fun main() : Unit = coroutineScope {
    	val flow = flowOf(1, 2, 3, 4, 5)
    	
    	flow.asFlux()
    		.doOnNext { print(it) }
    		.subscribe()
    		
    	flow.asFlowable()
    		.subscribe { print(it) }
    		
    	flow.asObservable()
    		.subscribe { print(it) }
    }
    ```
    
</br>    

## ⭐ Flow Builder

- 플로우를 만들 때 가장 많이 사용되는 방법
- sequence builder나 produce 빌더와 비슷하게 동작
- flow Builder 함수 호출 후 emit 함수를 사용해 값을 방출
    
    > Ex) Flow Builder
    > 
    
    ```kotlin
    fun makeFlow(): Flow<Int> = flow {
        repeat(3) { num ->
            delay(1000)
            emit(num)
        }
    }
    
    suspend fun main() {
        makeFlow()
            .collect { println(it) }
    }
    --------------
    (1초)
    0
    (1초)
    1
    (1초)
    2
    ```
    
    - 네트워크 API에서 **페이지 별로 요청되어야 하는** 목록 Stream 등을 만드는 목적으로 사용된다

</br>

## Flow Builder 이해하기

- 앞선 장에서 살펴봤듯이 collect 메서드 내부에서 block 함수를 호출하는 Flow Interface를 구현한다
    
    > Ex) flow 내부 동작
    > 
    
    ```kotlin
    fun <T> flow(
    	block : suspend FlowCollector<T>.() -> Unit
    ) : Flow<T> = object: Flow<T>() {}
    		override suspend fun collect(collector: FlowCollector<T>) {
    			// 인자로 들어오는 람다가 호출
    			collector.block()
    		}
    }
    
    interface Flow<out T> {
    	suspend fun collect(collector: FlowCollector<T>)
    }
    
    fun interface FlowCollector<in T> {
    	suspend fun emit(value: T)
    }
    ```

</br>

## 채널 Flow

- Flow는 Cold Data Stream이므로 필요할 때만 값을 생산
- ChannelFlow는 여러 코루틴에서 동시에 값을 보낼 때
- 콜드 Flow인데 데이터 수집 중에는 Hot 처럼 병렬 생산 및 버퍼링이 필요할 때 사용한다 (네트워크 호출이 더 빈번해지는 단점이 존재한다)
    
    > Ex) flow와 Channel Flow 결과 비교
    > 
    
    ```kotlin
    data class User(val name: String)
    
    interface UserApi {
        suspend fun takePage(pageNumber: Int): List<User>
    }
    
    class FakeUserApi : UserApi {
        private val users = List(20) { User("User$it") }
        private val pageSize: Int = 3
    
        override suspend fun takePage(
            pageNumber: Int
        ): List<User> {
            delay(1000) // 중단 지점
            return users
                .drop(pageSize * pageNumber)
                .take(pageSize)
        }
    }
    
    fun allUsersFlow(api: UserApi): Flow<User> = flow {
        var page = 0
        do {
            println("Fetching page $page")
            val users = api.takePage(page++)
            // 체크를 모두할 때까지 다음일 불가
            emitAll(users.asFlow())
        } while (users.isNotEmpty())
    }
    
    suspend fun main() {
        val api = FakeUserApi()
        val users = allUsersFlow(api)
        val user = users
            .first {
                println("Checking $it")
                delay(1000) // suspending
                it.name == "User3"
            }
        println(user)
    }
    --------------
    Fetching page 0
    Checking User(name=User0)
    Checking User(name=User1)
    Checking User(name=User2)
    Fetching page 1
    Checking User(name=User3)
    User(name=User3)
    ```
    
    - Channel Flow 적용
    
    ```kotlin
    data class User(val name: String)
    
    interface UserApi {
        suspend fun takePage(pageNumber: Int): List<User>?
    }
    
    class FakeUserApi : UserApi {
        private val users = List(20) { User("User$it") }
        private val pageSize: Int = 3
    
        override suspend fun takePage(
            pageNumber: Int
        ): List<User>? {
            delay(1000)
            return users
                .drop(pageSize * pageNumber)
                .take(pageSize)
        }
    }
    
    fun allUsersFlow(api: UserApi): Flow<User> = channelFlow {
        var page = 0
        do {
            println("Fetching page $page")
            val users = api.takePage(page++) // suspending
            // 채널 버퍼를 통해 값 전달
            users?.forEach { send(it) }
        } while (!users.isNullOrEmpty())
    }
    
    suspend fun main() {
        val api = FakeUserApi()
        val users = allUsersFlow(api)
        val user = users
            .first {
                println("Checking $it")
                delay(1000)
                it.name == "User3"
            }
        println(user)
    }
    -----------------
    Fetching page 0
    Checking User(name=User0)
    Fetching page 1
    Checking User(name=User1)
    Fetching page 2
    Checking User(name=User2)
    Fetching page 3
    Checking User(name=User3)
    Fetching page 4
    User(name=User3)
    ```
    
    - 여러 값을 독립적으로 계산 해야할 때 channelFlow를 주로 사용한다

</br>

## Callback Flow

- 사용자의 클릭이나 활동 변화를 감지하는 이벤트 Flow가 필요할 때 주로 사용된다 (감지 프로세스는 이벤트 처리 프로세스와 독립적이어야 함)
- Callback을 래핑하는 함수들 존재
    - `awaitClose { }` : 채널이 닫힐 때까지 중단되는 함수, 채널이 닫힌 후 인자로 들어온 함수가 실행된다 **(일반적으로 자원 해제)**
    - `trySendBlocking` : send와 비슷하지만 중단하는 대신 **Blocking하여** 일반 함수에서도 사용 가능하다
    - `trySend` : 즉시 시도 후 ChannelResult를 반환하며 가장 안전한 방식이다
    - `close` : 채널을 닫는다 (완료)
    - `cancel` : 채널을 종료하고 Flow에 예외를 던진다 → Block 자체를 즉시 종료하고 awaitClose 호출 후 정리로 넘어간다

---

## 요약

- Flow를 생성하는 다양한 방법 소개
    - `flowOf,` `emptyFlow`
    - flow Builder 함수
    - `channelFlow`, `callbackFlow`
- 각각의 용도가 다르기 때문에 여러 종류를 파악해야할 필요가 있다
