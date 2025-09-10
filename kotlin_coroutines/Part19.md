# Flow란?

- 비동기적으로 계산해야 할 값의 Stream
- 떠다니는 원소들을 모으는 역할을 하며 Flow의 끝에 도달할 때까지 각 값을 처리 (forEach와 유사)
    
    <img width="1267" height="286" alt="Image" src="https://github.com/user-attachments/assets/8bf9cc93-6d96-4610-adf6-3bd5e3ad2195" />
    
    - collect 중단 함수만 멤버로 가지며 다른 함수들은 Extension Function으로 정의 되어있다

</br>

## 플로우와 값들을 나타내는 다른 방법들의 비교

- List나 Set과 같은 Collection은 계산이 완료된 컬렉션으로 원소들이 채워질 때까지 모든 값이 생성되길 기다려야 한다
    
    > Ex) List 연산
    > 
    
    ```kotlin
    fun getList(): List<String> = List(3) {
       Thread.sleep(1000)
       "User$it"
    }
    
    fun main() {
       val list = getList()
       println("Function started")
       list.forEach { println(it) }
    }
    ----------------
    (3초) // getList로 원소들이 모두 채워지는데 3초 소요
    Function Started
    User0
    User1
    User2
    ```
    
    > Ex) Sequence 연산
    > 
    
    ```kotlin
    fun getSequence(): Sequence<String> = sequence {
       repeat(3) {
           Thread.sleep(1000)
           yield("User$it")
       }
    }
    
    fun main() {
       val list = getSequence()
       println("Function started")
       list.forEach { println(it) }
    }
    ---------------
    // Terminal 연산인 forEach에서 필요할 때 하나씩 계산
    Function started 
    (1초)
    User0
    (1초)
    User1
    (1초)
    User2
    ```
    
    - sequence는 중단 함수를 사용할 수 없는 특수한 코루틴이기 때문에 delay() 함수를 호출할 수 없고, Thread.sleep을 사용하게 된다면 스레드를 Blocking 해버린다
    - 또한 sequence 빌더 안에서는 yield, yieldAll 외에 다른 중단 함수를 호출할 수 없다
    
    > Ex) Sequence의 Thread Block
    > 
    
    ```kotlin
    fun getSequence(): Sequence<String> = sequence {
        repeat(3) {
            Thread.sleep(1000)
            yield("User$it")
        }
    }
    
    suspend fun main() {
        withContext(newSingleThreadContext("main")) {
            launch {
                repeat(3) {
                    delay(100)
                    println("Processing on coroutine")
                }
            }
    
            val list = getSequence() // 여기서 Thread Block
            list.forEach { println(it) }
        }
    }
    ------------
    (1초)
    User 0
    (1초)
    User 1
    (1초)
    User 2
    Processing on coroutine
    (0.1초)
    Processing on coroutine
    (0.1초)
    Processing on coroutine
    ```
    
    - Flow를 사용하면 코루틴이 연산을 수행하는데 필요한 기능을 전부 사용할 수 있다
    - Flow builder와 연산은 중단 함수이며 구조화된 동시성과 예외처리를 지원한다
    
    > Ex) 위의 코드에서 Sequence 대신 Flow 활용
    > 
    
    ```kotlin
    fun getFlow(): Flow<String> = flow {
        repeat(3) {
            delay(1000) // Thread를 막지 않는다
            emit("User$it")
        }
    }
    
    suspend fun main() {
        withContext(newSingleThreadContext("main")) {
            launch {
                repeat(3) {
                    delay(100)
                    println("Processing on coroutine")
                }
            }
    
            val list = getFlow() 
            list.collect { println(it) }
        }
    }
    -------------
    // Thread를 막지 않아 자식 코루틴의 실행 결과가 먼저 출력되었다
    Processing on coroutine
    Processing on coroutine
    Processing on coroutine
    User0
    User1
    User2
    ```

</br>

## Flow의 특징

- Flow의 최종 연산은 Thread를 막는 대신 코루틴을 중단시킨다
- 코루틴 컨텍스트 활용과 예외 처리 등의 코루틴 기능을 제공한다
- 취소 가능하며 구조화된 동시성을 갖추고 있다

<img width="905" height="225" alt="Image" src="https://github.com/user-attachments/assets/a596bf23-e446-4704-a238-0f7da6a9d729" />

> Ex) Flow  활용 예시
> 

```kotlin
fun usersFlow(): Flow<String> = flow {
    repeat(3) {
        delay(1000)
        val ctx = currentCoroutineContext()
        val name = ctx[CoroutineName]?.name
        emit("User$it in $name")
    }
}

suspend fun main() {
    val users = usersFlow()

    withContext(CoroutineName("Name")) {
        val job = launch {
            users.collect { println(it) }
        }

        launch {
            delay(2100)
            println("I got enough")
	           // 형제 코루틴 취소
            job.cancel()
        }
    }
}
------------------
// 3번 반복되는 중 2.1초 후 잡이 취소되어 flow 처리도 적절하게 취소
(1초)
User0 in Name
(1초)
User1 in Name
(0.1초)
I got enough
```

</br>

## Flow 명명법

- 플로우도 어디선가 시작되어야 한다
    - 플로우 Builder
    - 다른 객체에서의 변환
    - Helper 함수
- Flow의 마지막 연산은 Terminal 연산으로 불리며 중단 가능하거나 Scope를 필요로 하는 유일한 연산이다 (주로 collect 함수에 의해 최종 연산이 수행)
- 시작과 최종 연산 사이에 Flow를 변경하는 중간 연산을 가질 수 있다
    
    > Ex) Flow의 연산
    > 
    
    ```kotlin
    suspend fun main() {
    	flow { emit("Message 1") } // Builder로 시작
    		.onEach { println(it) }
    		.onStart { println("Something") }
    		.onCompletion { println("End") }
    		.collect { println("Collect $it") } // Terminal 연산
    	}
    }
    ```
</br>   

## 실제 활용 사례

- 대부분의 데이터는 필요할 때마다 요청을 하기 때문이 Cold Stream인 Flow가 자주 쓰인다
- 감지해야 할 이벤트가 있을 때 감지하는 모듈의 유무에 따라 이벤트 처리를 해야하는데 이때 Flow를 사용하는 것이 좋다
- 웹 등에서 서버가 보낸 이벤트롤 통해 전달된 메세지를 받는 경우
- 텍스트 입력 또는 클릭과 같은 사용자 액션이 감지된 경우 (채팅, 검색)
- 센서 또는 위치나 지도와 같은 기기의 정보 변경을 받는 경우
- 데이터베이스의 변경을 감지하는 경우 등

---

## 요약

- Flow는 Sequence와 달리 코루틴을 지원하며 비동기적으로 계산된 값을 나타내여 유용하다
