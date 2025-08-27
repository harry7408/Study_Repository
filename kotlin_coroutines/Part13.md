# 코루틴 스코프 만들기
## CoroutineScope 팩토리 함수

- CoroutineScope는 coroutineContext를 유일한 프로퍼티로 가지고 있는 Interface이다
    
    <img width="1144" height="293" alt="Image" src="https://github.com/user-attachments/assets/aa6a118e-db91-4772-b430-fc7ea2d8cd72" />
    
- CoroutineScope 인터페이스를 구현한 Class를 만든다면 내부에서 코루틴 빌더를 직접 호출할 수 있게된다
    - 하지만 이 방법은 `cancel, ensureActive` 같은 메서드 호출 시 문제가 생길 여지가 있어 잘 사용되지 않는다
    
    ```kotlin
    class CustomScope : CoroutineScope {
    	override val coroutineContext: CoroutineContext = Job()
    	
    	fun onStart() {
    		launch { ... } 
    	}
    }
    ```
    

- 코루틴 스코프 인스턴스를 Property로 가지고 있다가 코루틴 빌더를 호출할 때 사용하는 방법이 더 좋다
    
    ```kotlin
    class Foo {
    	val scope: CoroutineScope = ...
    	
    	fun onStart() {
    		// 스코프를 활용해서 코루틴 빌더 호출
    		scope.launch {
    			...
    		}
    	}
    }
    ```
    

- 코루틴 스코프를 만드는 가장 쉬운 방법은 CoroutineScope 팩토리 함수를 사용하여 만드는 것이다
    - **Job을 달아줘서 구조화된 동시성의 초석이 된다**
    
    <img width="1101" height="263" alt="Image" src="https://github.com/user-attachments/assets/a89d1707-4abe-407e-af2b-8ccb8b2d7c42" />
    
    ```kotlin
    public fun CoroutineScope(context: CoroutineContext): CoroutineScope =
        ContextScope(if (context[Job] != null) context else context + Job())
    ```

</br>

## 안드로이드에서 스코프 만들기

- 일반적으로 안드로이드에서는 ViewModel에서 코루틴이 가장 먼저 시작된다
    - UseCase를 주입받아 데이터를 가져오는 작업이 수행되는데 REST 등의 네트워크 작업은 장시간 소요, 취소, 동기화처리 등을 위해 중단함수로 선언하는 경우가 잦은데 이를 호출하려면 코루틴이 필요하다
- Repository, UseCase 등에서는 중단 함수가 일반적으로 사용된다
    
    > Ex) BaseViewModel에서 코루틴을 Property로 선언
    > 
    
    ```kotlin
    abstract class BaseViewModel(
    	private val onError: (Throwable) -> Unit
    ) : ViewModel {
    		
    		private val exceptionHandler = 
    			CoroutineExceptionHandler { _, throwable ->
    				onError(throwable)
    			}
    		
    		private val context = 
    			Dispatchers.Main + SupervisorJob() + exceptionHandler
    		
    		protected val scope = CoroutineScope(context)
    		
    		override fun onCleared() {
    			context.cancel() // 메모리 누수 방지 (전체 코루틴 취소)
    			context.coroutineContext.cancelChildren() // 자식 코루틴만 취소
    		}
    }
    
    ----------------------
    
    class MainViewModel(
    	private val repository: UserRepository
    ) : BaseViewModel {
    	
    	fun onCreate() {
    		// scope 사용
    		scope.launch { 
    			val info = repository.getInfo() 
    			view.showInfo()
    		}
    		...
    	}
    }
    
    ```
    
    - Android에서는 Main Thread가 많은 함수를 호출해야하기 때문에 기본 Dispatcher를 Main으로 잡아두는 것이 좋다 (ANR 발생하지 않도록 유의)
    - Scope를 취소 가능하게 만들어야 하므로 (ex. 사용자가 화면을 나가는 경우) 구조화된 동시성을 위해 Job이 필요하다
    - 전체 코루틴을 취소하는 대신 자식 코루틴만 취소하게 구성하여 ViewModel이 살아있는 한 같은 Scope에서 다른 코루틴이 시작될 수 있게 유지하는 것이 좋다
    - 예외 처리를 위한 ExceptionHandler를 설정하는 것도 중요하다

</br>

## viewModelScope와 lifecycleScope

- 안드로이드에서는 개발자가 따로 Scope를 정의하는 대신 안드로이드에서 제공하는 `viewModelScope, lifecycleScope` 를 사용할 수 있다 (의존성 필요)
- 기본적으로 두 Scope는 Dispatchers.Main과 SupervisorJob을 사용하고 ViewModel이나 Lifecycle이 종료되었을 때 알아서 Job을 취소시켜준다
- ExpcetionHandler 와 같은 특정 Context가 필요 없다면 안드로이드에서 제공하는 두 Scope를 사용하는 것이 편리하고 좋다 (실제로 많이 사용한다)

</br>

## Backend 측에서의 코루틴

- Spring Boot에서는 Controller 함수가 suspend로 선언되는 것을 허용, Ktor에서는 모든 핸들러가 기본적으로 중단 함수이다
- Scope를 따로 만들어야 한다면 `Thread Pool을 가진 Custom Dispatcher, SupervisorJob, CoroutineExceptionHandler` 등이 필요하다

</br>

## 추가적인 호출을 위한 Scope 만들기

- 추가적인 연산을 위한 Scope를 만들 때는 함수나 생성자의 인자를 통해 주입된다 (추가적인 연산을 위해선CoroutineContext를 변경이 필요하기 때문)
    - SupervisorJob으로 Job을 만들기
    - 예외 처리를 위한 CoroutineExceptionHandler 만들기
    - 코루틴이 실행하는 스레드를 정하기 위한 Dispatcher 변경

```kotlin
val scope = CoroutineScope(SupervisorJob())

------------------------

private val exceptionHandler = 
	CoroutineExceptionHandler { _, throwable ->
		FirebaseCrashlytics.getInstance()
				.recordException(throwable)
	}

val exceptionScope = CoroutineScope(SupervisorJob() + exceptionHandler)

--------------------------

val backgroundScope = CoroutineScope(
	SupervisorJob() + Dispatchers.IO
)
```

---

## 요약

⭐ 코루틴 스코프의 궁극적인 목적은 구조화된 동시성을 보장하고, 스코프 단위로 코루틴의 실행과 취소를 안전하게 관리하기 위함이다
