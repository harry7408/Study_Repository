# Flow의 실제 구현
- Flow는 어떤 연산을 실행할 지 정의한 것이다
- 중단 가능한 람다식에 몇 가지 요소를 추가


## Flow 이해하기

- 절차에 따라 flow 빌더가 어떻게 동작하는지 이해

1. 람다식
    
    ```kotlin
    fun main() {
        val f: () -> Unit = {
            print("A")
            print("B")
            print("C")
        }
        // 1번 정의 후 여러 번 호출 가능하다
        f() 
        f() 
    }
    -----------
    ABCABC
    ```
    </br>
    
2. 지연이 있는 람다식
    
    ```kotlin
    suspend fun main() {
        val f: suspend () -> Unit = {
            println("A")
            delay(1000)
            println("B")
            delay(1000)
            println("C")
        }
        // 마찬가지로 1번 정의 후 여러 번 호출 가능하다
        f()
        f()
    }
    ----------
    A
    (1초)
    B
    (1초)
    C
    A
    (1초)
    B
    (1초)
    C
    ```
    </br>
    

3. 함수를 나타내는 파라미터를 갖는 람다식
    
    ```kotlin
    suspend fun main() {
        val f: suspend ((String) -> Unit) -> Unit = { emit ->
            emit("A")
            emit("B")
            emit("C")
        }
        // it은 함수의 매개변수가 된다 
        //(String 타입의 매개변수 1개만 존재하므로 it으로 명명된다)
        f { print(it) } 
        f { print(it) } 
    }
    ------------
    ABCABC
    ```
    
    - 여기서 emit은 (String) → Unit 타입이며 람다식의 파라미터 이름이다
    - 호출의 흐름
        1. `f { print(it) }` 실행
        2. `emit = { s: String → print(s) }`
        3. f의 호출 body에서 `emit(”A”)` 실행
        4. `{ s → print(s) }(”A”)` 실행
        5. `it = “A” 이며 → print(”A”)`가 실행된다
        </br>

4. 매개변수 대신 추상 메서드를 갖는 함수형 인터페이스 정의
    
    ```kotlin
    // 함수형 인터페이스 정의
    fun interface FlowCollector {
        suspend fun emit(value: String)
    }
    
    suspend fun main() {
        val f: suspend (FlowCollector) -> Unit = {
    		    // it -> FlowCollector
            it.emit("A")
            it.emit("B")
            it.emit("C")
        }
        f { print(it) } 
        f { print(it) } 
    }
    ------------
    ABCABC
    ```
    </br>
    

5. 리시버가 있는 람다식 정의
    
    ```kotlin
    fun interface FlowCollector {
        suspend fun emit(value: String)
    }
    
    suspend fun main() {
    		// Extension Function 형태
        val f: suspend FlowCollector.() -> Unit = {
    		    // this 가 생략되어있다
            emit("A")
            emit("B")
            emit("C")
        }
        f { print(it) } 
        f { print(it) } 
    }
    -----------
    ABCABC
    ```
    </br>
    

6. 람다식 대신 인터페이스를 구현한 객체 제작
    
    ```kotlin
    fun interface FlowCollector {
        suspend fun emit(value: String)
    }
    
    // Flow 인터페이스
    interface Flow {
        suspend fun collect(collector: FlowCollector)
    }
    
    suspend fun main() {
        val builder: suspend FlowCollector.() -> Unit = {
            emit("A")
            emit("B")
            emit("C")
        }
        // 인터페이스의 구현체
        val flow: Flow = object : Flow {
            override suspend fun collect(
                collector: FlowCollector
            ) {
                collector.builder()
            }
        }
        flow.collect { print(it) } 
        flow.collect { print(it) } 
    }
    -------------
    ABCABC
    ```
    </br>
    

7. Flow 생성하는 빌더 정의 및 제너릭 타입 적용
    
    ```kotlin
    fun interface FlowCollector<T> {
        suspend fun emit(value: T)
    }
    
    interface Flow<T> {
        suspend fun collect(collector: FlowCollector<T>)
    }
    
    // Flow 빌더 정의
    fun <T> flow(
        builder: suspend FlowCollector<T>.() -> Unit
    ) = object : Flow<T> {
        override suspend fun collect(
            collector: FlowCollector<T>
        ) {
    			  // collector를 리시버로 주입하여 빌더 블록을 실행하겠다
            collector.builder()
        }
    }
    
    suspend fun main() {
        val f: Flow<String> = flow {
            emit("A") // collector.emit("A")
            emit("B")
            emit("C")
        }
        f.collect { print(it) } 
        f.collect { print(it) } 
    }
    --------------
    ABCABC
    ```
    

⭐ collect를 호출하면 flow 빌더를 호출할 때 넣은 람다식이 실행된다 → 빌더 Body에서 emit을 호출하면 collect가 호출되었을 때 명시된 람다식이 호출된다

</br>

## Flow 처리 방식

- flow를 생성하고, 처리하고, 감지하기 위해 정의한 함수들에서 Flow의 강점을 확인할 수 있다
- Flow의 각 원소를 변환하는 map함수
    - 새로운 Flow를 만들기 때문에 flow builder 및 빌더 내부에서 collect가 호출된다
    
    ```kotlin
    inline fun <T, R> Flow<T>.map(
        crossinline transform: suspend (value: T) -> R
    ): Flow<R> = flow {
    	collect {
    		emit(transform(it))
    	}
    }
    ```
 
</br>   

## 동기로 작동하는 flow

- Flow 또한 중단 함수처럼 동기로 작동하기 때문에 flow가 완료될 때까지 collect 호출이 중단된다
- flow에서 각각의 처리 단계는 동기로 실행된다
    
    ```kotlin
    suspend fun main() {
        flowOf("A", "B", "C")
            .onEach { delay(1000) }
            .collect { println(it) }
    }
    -----------
    A
    (1초)
    B
    (1초)
    C
    ```

</br>

## FLOW와 공유 상태

- 플로우 처리로 복잡한 알고리즘을 구현할 때는 언제 변수에 대한 접근을 동기화해야 하는지 알아야 한다
- 외부의 변수는 flow가 모으는 모든 코루틴이 공유하게 되므로 외부 변수 추출해서 사용할 때는 동기화가 반드시 필요하다
    
    > Ex) 공유상태 문제 X
    > 
    
    ```kotlin
    fun Flow<*>.counter() = flow<Int> {
    		// 지역 변수
        var counter = 0
        collect {
            counter++
            // to make it busy for a while
            List(100) { Random.nextLong() }.shuffled().sorted()
            emit(counter)
        }
    }
    
    suspend fun main(): Unit = coroutineScope {
        val f1 = List(1000) { "$it" }.asFlow()
        val f2 = List(1000) { "$it" }.asFlow()
            .counter()
        
        launch { println(f1.counter().last()) } 
        launch { println(f1.counter().last()) } 
        launch { println(f2.last()) } 
        launch { println(f2.last()) } 
    }
    ------------
    1000
    1000
    1000
    1000
    ```
    
    - 1~1000까지 숫자를 방출하게 되며 last를 통해 마지막 값인 1000이 출력된다
    - counter 변수가 지역 변수이므로 counter()를 호출할 때마다 새로 만들어져 collect()가 시작될 때마다 0에서 부터 출발
    
    > Ex) 공유상태 문제 O
    > 
    
    ```kotlin
    fun Flow<*>.counter(): Flow<Int> {
        var counter = 0
        return this.map {
            counter++
            // to make it busy for a while
            List(100) { Random.nextLong() }.shuffled().sorted()
            counter
        }
    }
    
    suspend fun main(): Unit = coroutineScope {
        val f1 = List(1_000) { "$it" }.asFlow()
    	  // 하나의 flow 인스턴스를 만들었다
        val f2 = List(1_000) { "$it" }.asFlow()
            .counter()
        
        launch { println(f1.counter().last()) } 
        launch { println(f1.counter().last()) } 
        launch { println(f2.last()) } 
        launch { println(f2.last()) } 
    }
    ------------
    // f2 측은 2천보다 작은 값이 출력
    // 코루틴 완료 순서에 따라 출력이 달라진다
    1827
    1000
    1000
    1998
    ```
    
    - map 밖에서 counter 변수가 선언되어있다
    - 2개의 collect가 동시에 counter 변수를 증가시켜 2000이 나와야하는데 Race Condition 문제로 2000보다 작은 값이 출력
    - counter 변수를 코드 최상단에 두면 Race Condition이 발생하여 모두 4000보다 작은 값이 출력된다

---

## 요약

- flow는 리시버가 있는 Suspend 람다식에 비해 복잡하고 어렵게 여겨진다
- flow의 처리 함수들은 flow를 새로운 연산으로 Decorate한다

⭐ 즉, Collect로 주문을 Trigger하면 Builder 본문에서 레시피에 따라 생산 및 조리를 한 후 emit이 호출될 때 완성품을 전달하는 것이 Flow의 동작 흐름이다
