# Flow 생명주기 함수

- Flow는 요청이 한쪽 방향으로 흐르고 요청에 의해 생성된 값이 다른 방향으로 흐르는 파이프로 볼 수 있다
- Flow가 완료되거나 예외가 발생했을 때 정보가 전달되어 중간 단계가 종료
- 모든 정보가 Flow로 전달되기 때문에 예외 및 특정 이벤트를 감지할 수 있다
- Flow를 수행하는 과정에서 다양한 메서드들이 존재

</br>

## onEach

- Flow의 값을 하나씩 받을 때 사용하는 메서드
    
    <img width="1103" height="206" alt="Image" src="https://github.com/user-attachments/assets/12c5bfab-238a-4f4c-ad3d-2dd305e90679" />
    
    ```kotlin
    suspend fun main() {
    	flowOf(1,2,3,4)
    		.onEach { print(it) }
    		.collect()
    }
    -------------
    1234
    ```
    
    - onEach의 인자 람다식은 중단 함수이다
    - 들어오는 원소는 순서대로 처리된다

</br>

## onStart

- Flow가 시작되는 경우에 호출되는 Listener를 설정하는 메서드
    
    <img width="1242" height="409" alt="Image" src="https://github.com/user-attachments/assets/a11ca037-8e71-4522-9d4d-7dfe07902714" />
    
    ```kotlin
    suspend fun main() {
    	flowOf(1,2)
    		.onEach { delay(1000) }
    		.onStart { println("Before") }
    		.collect { println(it) }
    }
    ---------------
    Before
    (1초)
    1
    (1초)
    2
    ```
    
    ⭐ 첫 번째 원소를 요청했을 때 호출되는 메서드이다
    
    - collect 하기 직전에 실행할 Action을 붙이기 때문에 SharedFlow와 같이 사용할 때 SharedFlow가 보낸 값들은 수집된다는 보장이 없다 (Hot Flow)
    
    > Ex) onStart에서의 원소 방출
    > 
    
    ```kotlin
    suspend fun main() {
    	flowOf(1,2)
    		.onEach { delay(1000) }
    		.onStart { emit(0) } // 값 emit
    		.collect { println(it) }
    }
    ----------
    0
    (1초)
    1
    (1초)
    2
    ```

</br>

## onCompletion

- Flow 빌더가 끝났을 때 즉 Flow가 완료되었을 때 호출되는 Listener를 설정하는 메서드
- 잡히지 않은 예외가 발생하거나 코루틴이 취소될 때도 onCompletion 메서드의 인자가 실행된다
    
    <img width="881" height="327" alt="Image" src="https://github.com/user-attachments/assets/cb531aed-204f-4f59-8e9e-8e7731b9e69a" />
    
    ```kotlin
    suspend fun main() = coroutineScope {
    	flowOf(1,2)
    		.onEach { delay(1000) }
    		.onCompletion { println("Completed") }
    		.collect { println(it) }
    }
    -----------
    (1초)
    1
    (1초)
    2
    Completed
    ```
    
    > ⭐ 안드로이드에서 Flow 메서드 활용
    > 
    
    ```kotlin
    fun doSomething() {
    	scope.launch {
    		FooFlow()
    			.onStart { showProgressBar() }
    			.onCompletion { hideProgressBar() }
    			.collect { view.show(it) }	
    	}
    }
    ```
    
    - 네트워크 응답을 기다리는 동안 ProgressBar를 보여줄 때 활용하면 좋다

</br>

## onEmpty

- 원소를 내보내기 전에 Flow가 완료되면 실행되는 메서드
- 기본 값을 내보내기 위한 목적으로도 활용될 수 있다
    
    <img width="890" height="217" alt="Image" src="https://github.com/user-attachments/assets/84738e8d-07d8-4680-9ccb-e3641dbc963e" />
    
    ```kotlin
    suspend fun main() = coroutineScope {
    	flow<List<Int>> { delay(1000) }
    		.onEmpty { emit(emptyList()) }
    		.collect { println(it) }
    }
    ---------------
    (1초)
    []
    ```

</br>

## catch

- Flow를 만들거나 처리하는 도중 발생하는 예외를 잡고 관리하는 메서드
- 예외를 인자로 받고 정리를 위한 연산을 수행할 수 있다
    
    <img width="897" height="309" alt="Image" src="https://github.com/user-attachments/assets/95d4374b-86b0-452b-bca6-01fef65d26e0" />
    
    ```kotlin
    class MyError : Throwable("My error")
    
    val flow = flow {
        emit(1)
        emit(2)
        throw MyError() // 예외 발생 지점
    }
    
    suspend fun main(): Unit {
        flow.onEach { println("Got $it") } 
            .catch { println("Caught $it") }
            .collect { println("Collected $it") }
    }
    -----------
    Got 1
    Collected 1
    Got 2
    Collected 2
    Caught ~.MyError: My error
    ```
    
    - 💡 onEach는 예외에 반응하지 않고, onCompletion 핸들러만 예외가 발생했을 때 호출된다
    - catch 메서드는 예외를 잡아 전파 되는 것을 막으며 블럭 내에서 새로운 값을 보낼 수 있어 남은 Flow를 지속할 수 있다
    
    > Ex) catch 블럭 내에서의 emit
    > 
    
    ```kotlin
    class MyError : Throwable("My error")
    
    val flow = flow {
        emit("Message1")
        throw MyError()
    }
    
    suspend fun main(): Unit {
        flow.catch { emit("Error") }
            .collect { println("Collected $it") }
    }
    ----------
    Collected Message1
    Collected Error
    ```
    
     ⭐ catch는 자신 이후에 발생하는 예외는 반응하지 않고 위쪽에서 난 예외만 잡는다 
    
    ```kotlin
    suspend fun main() : Unit {
    	flowOf("Message1")
    		.catch { emit("Foo") }
    		.onEach { throw Error(it) } // catch의 DownStream에서 발생
    		.collect {println("Collected $it") }
    }
    ---------------
    Exception in thread main java.lang.Error: Message1
    ```
    
    - 안드로이드에서는 Flow에서 일어나는 예외 또는 Screen에서 보여지는 기본 값을 내보낼 때 유용하다
    
    ```kotlin
    fun doSomething() {
    	scope.launch {
    		FooFlow()
    			.catch { view.handleError(it) } // 예외 보여주기
    			.onStart { showProgressBar() }
    			.onCompletion { hideProgressBar() }
    			.collect { view.showNews(it) }
    	}
    }
    ---------------------------------------
    
    fun doSomething() {
    	scope.launch {
    		FooFlow()
    			.catch { 
    				view.handleError(it)
    				emit(emptyList()) // 빈 화면에 보여줄 데이터 방출
    			 } 
    			.onStart { showProgressBar() }
    			.onCompletion { hideProgressBar() }
    			.collect { view.showNews(it) }
    	}
    }
    ```

</br>

## 잡히지 않는 Exception

- Flow에서 잡히지 않는 예외는 Flow를 즉시 취소하고 Collect는 다시 예외를 던지게 된다
- Flow 바깥에서 try~catch 블럭으로 예외를 잡을 수 있다
    
    ```kotlin
    val flow = flow {
    	emit("Message 1")
    	throw MyError() // 예외 발생
    }
    
    suspend fun main(): Unit {
    	try {
    		flow.collect { println("Collected $it") } // 예외가 다시 던져진다
    	} catch(e: MyError) {
    		println("Caught")
    	}
    }
    -----------------
    Collected Message1
    Caught
    ```
    
    - 앞선 catch에서 살펴 봤듯이 catch Block의 하단에서 발생하는 에러는 잡지 못하여 Block 밖으로 예외가 던져진다
    
    > Ex) 예외를 최종 연산에서 발생하는 경우
    > 
    
    ```kotlin
    val flow = flow {
    	emit("Message1")
    	emit("Message2")
    }
    
    suspend fun main(): Unit {
    	flow.onStart { println("Before") }
    			.catch { println("Caught $it") }
    			.collect { throw MyError() } // 예외를 잡지 못하게 된다
    }
    -----------
    Before
    Exception in thread main MyError: My error
    
    // 해결 방법
    suspend fun main(): Unit {
    	flow.onStart { println("Before") }
    			.onEach { throw MyError() }
    			.catch { println("Caught $it") }
    			.collect()
    }
    -------------
    Before
    Caught MyError: My error
    ```

</br>

## flowOn

- 해당 위치의 UpStream에 있는 연산들을 어떤 Context에서 동작하게 할 지 변경할 때 사용하는 메서드
    
    <img width="897" height="251" alt="Image" src="https://github.com/user-attachments/assets/8d431965-e1ee-4ad7-abbe-b99f3e633e7c" />
    
    ```kotlin
    suspend fun present(place: String, message: String) {
        val ctx = coroutineContext
        val name = ctx[CoroutineName]?.name
        println("[$name] $message on $place")
    }
    
    fun messagesFlow(): Flow<String> = flow {
        present("flow builder", "Message")
        emit("Message")
    }
    
    suspend fun main() {
        val users = messagesFlow()
        withContext(CoroutineName("Name1")) {
            users
                .flowOn(CoroutineName("Name3"))
                .onEach { present("onEach", it) }
                .flowOn(CoroutineName("Name2"))
                .collect { present("collect", it) }
        }
    }
    -------------
    [Name3] Message on flow builder
    [Name2] Message on onEach
    [Name1] Message on collect
    ```
    
    ⭐ flowOn은 윗부분에 있는 함수에서만 작동한다

</br>

## launchIn

- collect는 Flow가 완료될 때까지 코루틴을 중단하는 함수인데 launch Builder로 collect를 Wrapping하면 다른 코루틴에서 처리가 가능하다
- launchIn은 인자로 Scope를 받아 collect를 새로운 코루틴에서 시작할 수 있다
- 내부적으로 `scope.launch { collect() }`를 호출
    
    <img width="919" height="277" alt="Image" src="https://github.com/user-attachments/assets/e515873e-ffd4-43ed-ae52-7d0e5f80becf" />
    
    ```kotlin
    suspend fun main(): Unit = coroutineScope {
        flowOf("User1", "User2")
            .onStart { println("Users:") }
            .onEach { println(it) }
            .launchIn(this) // 현재 스코프에서 실행해라
    }
    ------------
    Users:
    User1
    User2
    ```
    

---

## 요약

- Flow가 시작될 때, 닫힐 때, 각 원소를 탐색할 때 Flow에 작업을 추가할 수 있다
- 예외를 잡거나 새로운 코루틴에서 Flow를 시작할 수도 있다

 **⭐ Android에서 Flow를 쓸 때 Flow 생명주기 함수들이 자주 사용된다**
