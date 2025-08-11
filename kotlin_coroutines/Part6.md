# 코루틴 빌더

- 중단 함수는 Continuation 객체를 다른 중단 함수로 전달해야 한다

**⭐ 중단 함수가 일반 함수를 호출하는 것은 가능하지만 일반 함수가 중단 함수를 호출하는 것은 불가능 (즉, 모든 중단 함수는 또 다른 중단 함수에 의해 호출 되어야 한다)**

- 중단 함수를 연속적으로 호출하면 **시작되는 지점이 반드시 존재**

→ **CoroutineBuilder**가 그 역할을 한다 (일반 함수와 중단 가능한 세계를 연결하는 역할)

## 코루틴 빌더의 종류

- launch
    - 📎 https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
- runBlocking
    - 📎  https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
- async
    - 📎 https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html

⇒ 3가지는 서로 다른 쓰임새가 있다

- runBlocking 함수는 쓸 때 고려해봐야 한다 (특히 Android에서)
- launch와 async 메서드는 CoroutineScope Interface의 확장 함수이다!!

## launch 빌더

### 메서드 시그니처

<img width="501" height="138" alt="Image" src="https://github.com/user-attachments/assets/f8efcf15-a681-41ef-a6ef-b0d3c808e8e1" />

- launch의 동작 방식은 thread { } 블록을 호출하여 새로운 Thread를  시작하는 것과 유사
    - launch { } 에 의해 Coroutine을 시작하면 **각자 별개로 실행**
- launch 함수는 CoroutineScope 인터페이스의 확장 함수
    - CoroutineScope : **부모 코루틴과 자식 코루틴 사이의 관계를 정립하기 위한 목적으로 사용**
    
    → (구조화된 동시성의 핵심)
    

**⭐ 실무에서는 GlobalScope의 사용을 지양해야 한다**

**EX)**

```kotlin
fun main() {
    GlobalScope.launch {
        delay(1000L)
        println("1")
    }

    GlobalScope.launch {
        delay(1000L)
        println("2")
    }

    GlobalScope.launch {
        delay(1000L)
        println("3")
    }

    println("Hello")
    Thread.sleep(2000L)
}
//
* 위의 코드에서 delay 시간 보다 sleep 시간이 짧은 부분은 실행되지 않는다
 -> 프로그램이 먼저 죽어버린다

* 어떤 코루틴이 먼저 종료될지는 실행 시 마다 다르다
```

- main 함수의 끝에 Thread.sleep을 호출하지 않으면 Hello 만 호출되고 코루틴이 일을 할 기회가 주어지지 않는다 **(delay가 Thread를 블록시키지 않고 코루틴을 중단시킨다)**
    - delay에 의해 코루틴만 일시중단된 채 main Thread는 즉시 반환되는데 JVM 프로세스가 종료되면 Background 에서 동작하는 코루틴도 같이 멈춰버리기 때문에 Coroutine이 일 할 기회를 얻지 못한다
- 위 방식은 Daemon Thread의 동작 방식과 유사하지만 코루틴이 훨씬 가볍다
    - Block 된 스레드를 유지하는 것은 비용이 들지만 중단된 코루틴 유지는 공짜나 다름없다
    - 둘 다 별개의 작업을 시작하며 **작업을 하는 동안 프로그램이 끝나는 것을 방지해야 한다**는 공통점이 있다.  (위의 예제에서는 Thread.sleep(2000L)이 그 역할을 한다)

```kotlin
fun main() {
    thread(isDaemon = true) {
        Thread.sleep(1000L)
        println("1")
    }

    thread(isDaemon = true) {
        Thread.sleep(1000L)
        println("2")
    }

    thread(isDaemon = true) {
        Thread.sleep(1000L)
        println("3")
    }
    println("CountDown")
    Thread.sleep(1500L)
}
// 각 Thread에 이름을 찍어보면 모두 다르게 나온다
-------------
CountDown
1
2
3
// 1,2,3 순서는 실행할 때 마다 다르게 나옴
```

Daemon Thread : 일반 스레드의 작업을 돕는 보조적인 역할을 한다 (백그라운드 용 스레드)

**→ 일반 Thread가 종료되면 (isDaemon=true)인 데몬 스레드도 강제 종료 된다**

## runBlocking 빌더

### 메서드 시그니처

<img width="520" height="112" alt="Image" src="https://github.com/user-attachments/assets/29140a5b-727a-4cde-9189-bae097b533ca" />

- 코루틴이 스레드를 블록 해야 하는 경우 사용 (Main 함수의 경우 프로그램을 너무 빨리 끝내지 않기 위해 스레드를 Blocking 해야 한다)
    - Android 같은 경우는 UI와 관련된 Main Thread 1개만 가지고 있어 runBlocking 사용 시 유의해야 한다 (안쓰는 것이 좋다)
- **코루틴이 중단되었을 경우 runBlocking 빌더는 시작한 스레드를 중단시킨다**

```kotlin
fun main() {
   runBlocking { 
      delay(1000L)
      println("1")
   }
   runBlocking {
      delay(1000L)
      println("2")
   }
   runBlocking {
      delay(1000L)
      println("3")
   }
   println("Hello")
}
// 출력 결과
(1초)
1
(1초)
2
(1초)
3
Hello // runBlocking에서 Coroutine이 delay에 의해 중단되어 Main Thread가 중단되어 마지막에 호출
//
runBlocking에 의해 main 함수에 Thread.sleep이 필요 없이 
코루틴이 중단될 때 Main Thread도 같이 멈춘다
(**시작한 스레드가 main**이기 때문에 launch와 다르게 순차적으로 출력)
-> launch는 별개로 작업되기 때문

여기서 runBlocking {
		delay(1000L)
}은 Thread.sleep(1000L)과 같은 동작을 하게 된다
```

- runBlocking이 사용되는 특수한 경우 2가지
    - 프로그램이 끝나는 걸 방지하기 위해 Thread 를 막을 필요가 있는 메인 함수
    - 프로그램이 끝나는 걸 방지하기 위해 Thread 를 막을 필요가 있는 유닛 테스트

```kotlin
// main 함수 Block
fun main() = runBlocking {

}

// 단위 테스트 Block
class MyTest {

	@Test
	fun `a test`() = runBlocking {

	}
}
```

runBlocking은 거의 사용되지 않는다 → 단위 테스트 부분에서도 runTest가 주로 사용

- runTest : 코루틴을 가상으로 실행시킨다
- runBlockingTest는 Deprecated

**⭐ main 함수 중단은 suspend 키워드를 붙여 중단 함수로 만들어 멈추는 방법을 주로 사용**

```kotlin
suspend fun main() {
    GlobalScope.launch {
        delay(1000L)
        println("1")
    }

    GlobalScope.launch {
        delay(1000L)
        println("2")
    }

    GlobalScope.launch {
        delay(1000L)
        println("3")
    }
    println("Hello")
    delay(2000L)
}

// 출력 결과
Hello
(1초)
1,2,3
```

⭐ 구조화된 동시성과 아닌 것의 차이

```kotlin
fun main() = runBlocking {
    launch { // same as this.launch
        delay(1000L)
        println("World!")
    }
    launch { // same as this.launch
        delay(1000L)
        println("World!")
    }
    launch { // same as this.launch
        delay(1000L)
        println("World!")
    }
    println("Hello,")
}

--------------------
fun main() = runBlocking {
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    // 2초동안 멈춰주면 나머지 코루틴이 실행된다
    //delay(2000L)
    
}
```

- 전자의 경우 launch Builder가 runBlocking의 자식 코루틴이되어 자식 코루틴이 모두 끝날 때 까지 기다려주는 반면 후자는 별도의 Scope로 launch Builder를 만들기 때문에 부모 자식 관계가 형성되지 않아 main 함수에 delay를 걸지 않으면 나머지 코루틴들이 실행되지 않는다

## async 빌더

### 메서드 시그니처

<img width="501" height="134" alt="Image" src="https://github.com/user-attachments/assets/0b35a113-7405-4bd0-8acc-501a24a25473" />

- launch 빌더 와 유사하지만 **값을 생성하도록 설계** 되어있다 (람다 표현식에 의해 반환 되어야함 → async Builder의 인자가 람다식)
    - 마지막 표현식이 결과 타입 <T>가 된다
- Deferred<T> 타입의 객체**(Job 객체를 상속)**를 리턴하며 Deferred에는 **작업이 끝나면 값을 반환하는 중단 메서드인 await()가 있다**

```kotlin
fun main() = runBlocking {
   val resultDeferred : Deferred<Int> = GlobalScope.async {
       delay(1000L)
       42 // 람다 표현식 반환
   }

    val result: Int = resultDeferred.await() // 값을 반환하는 중단 메서드
    println(result)
}
// 출력결과 : 42
```

- launch 빌더와 비슷하게 aysnc 빌더는 호출되자마자 코루틴을 **즉시 시작**

**→ 몇 개의 작업을 한 번에 시작하고 모든 결과를 한꺼번에 기다릴 때 유용 (병렬로 처리할 때)**

- Deferred 객체는 값이 생성되면 내부에 저장하기 때문에 **await에서 값이 반환 되는 즉시 사용 가능** (값이 생성되기 전 await를 호출하면 값이 나올 때까지 기다린다)

**EX)**

```kotlin
scope.launch {
	val news= async {
			newsRepo.getNews()
					.sortedByDescending { it.date }
	}
	val newsSummary = newsRepo.getNewsSummary()
	view.showNews(
			newsSummary,
			news.await()
	)
}
```

⭐ async 빌더는 주로 두 가지 다른 곳에서 데이터를 얻어와 합치는 경우처럼 두 작업을 병렬로 실행할 때 주로 사용된다

## 구조화된 동시성

**코루틴은 어떤 스레드도 블록하지 않기 때문에 프로그램이 끝나는 것을 막을 방법이 없다**

부모 자식 간의 관계

- 자식은 부모로부터 Context를 상속받는다(자식이 재정의 가능)
- **부모는 모든 자식이 작업을 마칠 때까지 기다린다**
- **부모 코루틴이 취소되면 자식 코루틴도 취소된다**
- 자식 코루틴에서 에러가 발생하면 부모 코루틴 또한 에러로 소멸한다

> EX)
> 

```kotlin
fun main = runBlocking {
	this.launch { // runBlocking이 제공하는 Coroutine.()->T 리시버를 사용
		delay(1000L)
		print("World")
	}
	
	launch { // this.launch와 동일
		delay(1500L)
		print("Wow")
	}
	
	print("Hello")
}

// 출력 결과
Hello
(1초)
World
(0.5초)
Wow
```

## 현업에서의 코루틴 사용

- 중단 함수는 다른 중단 함수들로부터 호출 되어야 하며, 모든 중단 함수는 Coroutine Builder로 시작 되어야 한다
- RunBlocking을 제외한 모든 Coroutine Builder는 **CoroutineScope에서 시작 되어야 한다**(Builder가 Coroutine Scope의 확장 함수이기 때문)
- Kotlin을 주로 활용하는 Android 같은 경우 ViewmodelScope, LifecycleScope 등의 프레임워크에서 제공하는 Scope를 활용하고 백엔드에서는 Ktor를 사용
- 첫 번째 빌더가 Scope에서 시작되면 다른 Builder가 첫 번째 빌더의 스코프 내에서 실행될 수 있다
    
    
    > Ex) Android에서의 Coroutine 활용 예시
    > 
    
    ```kotlin
    // Data Layer에 있을만한 Code
    class NetworkUserRepository(
    	private val api: UserApi,
    ) : UserRepository {
    		// 중단 함수
    		suspend fun getUser() : User = api.getUser.toDomainUser()
    }
    
    class NetworkNewsRepository(
    	private val api: NewsApi,
    	private val settings: SettingsRepository,
    ) : NewsRepository {
    	
    		suspend fun getNews(): List<News> = api.getNews()
    			.map {it.toDomainNews() }
    			
    		suspend fun getNewsSummary(): List<News> {
    			val type = settings.getNewsSummaryType()
    			return api.getNewsSummary(type)
    		}	
    }
    
    // mvp 방식 잘 안쓴다..
    class MainPresenter(
    	private val view: MainView,
    	private val userRepo: UserRepository,
    	private val newsRepo: NewsRepository,
    ) : BasePresenter {
    		
    		fun onCreate() {
    			scope.launch {
    				val user = userRepo.getUser()
    				view.shoUserData(user)
    			}
    			
    			// 첫번째 빌더
    			scope.launch {
    				// 빌더 안의 빌더
    				val news = async {
    					newsRepo.getNews()
    							.sortByDescending { it.date }
    				}
    				
    				// async는 값을 반환해야 하기 때문에 변수에 할당 가능
    				val newsSummary = async {
    						newsRepo.getNewsSummary()
    				}
    				view.showNews(newsSummary.await(), news.await())
    			}		
    		}
    }
    ```
    
    > Ex) Backend에서의 예시
    > 
    
    ```kotlin
    @Controller
    class UserController(
    	private val tokenService: TokenService,
    	private val userService: UserService,
    ) {
    		
    		// 중단 함수
    		@GetMapping("/me")
    		suspend fun findUser(
    			@PathVariable userId: String,
    			@RequestHeader("Authorization") authorization: String
    		) : UserJson {
    				val userId = tokenService.readUserId(authorization)
    				val user = userService.findUserById(userId)
    				return user.toJson()
    		}
    }
    ```
    

## coroutineScope

### 메서드 시그니처

<img width="682" height="41" alt="Image" src="https://github.com/user-attachments/assets/a7894606-dc07-4b2c-a1c3-8369c9999977" />

- 람다 표현식이 필요로 하는 스코프를 만들어주는 **중단함수** (람다식이 반환하는 것이면 무엇이든 반환)
    
    → 마지막 줄이 Expression
    
- 중단 함수 내에서 스코프가 필요할 때 일반적으로 사용하는 함수 (중단 함수 밖에서 스코프를 만들고자 할 때)

 ⭐ 중단 함수에서 **안전한 자식 코루틴 범위를 잠깐 만들어 사용할 때 활용**

> Ex) coroutineScope 메서드 활용
> 
- 사용자가 볼 수 있는 기사만 필터링 해서 가져오기

```kotlin
suspend fun getArticlesForUser(
	userToken: String?
) : List<ArticleJson> = coroutineScope { // 스코프 제작
		val articles = async { articleRepository.getArticles() }
		val user = userService.getUser(userToken)
		articles.await()
			.filter { canSeeOnList(user,it) }
			.map { toArticleJson(it) }
}
```

---

## 요약

- 코루틴은 빌더(Scope) 또는 **runBlocking** 에서 시작된다

→ 이후 다른 코루틴 빌더나 중단 함수를 호출할 수 있다

**→ coroutineScope로 함수를 래핑한 후 스코프 내에서 빌더 사용 (동시성을 처리하고자 할 때)**

- 중단 함수에서 빌더를 실행할 수는 없으므로 coroutineScope와 같은 코루틴 스코프 함수를 사용한다
