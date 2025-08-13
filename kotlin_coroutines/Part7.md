# 코루틴 컨텍스트

- 코루틴 빌더의 첫 번째 파라미터가 CoroutineContext이다
- CoroutineScope 인터페이스 내부에도 CoroutineContext가 감싸져 있다
    
    ```kotlin
    interface CoroutineScope {
    	// 2개의 Property 존재
    	abstract val coroutineContext: CoroutineContext
    	val CoroutineScope.isActive: Boolean
    }
    ```
    
- Continuation 객체도 CoroutineContext를 포함하고 있다
    
    ```kotlin
    interface Continuation<in T> {
    	// 1개의 Property 존재
    	abstract val context: CoroutineContext
    }
    ```
    

**→ 중요한 요소들이 CoroutineContext를 사용하고 있다**

 ⭐ 코루틴 실행 환경을 구성하는 Key-Element 들의 집합

## CoroutineContext

- 원소나 원소들의 집합을 나타내는 **인터페이스 (일종의 컬렉션)**
    
    **(Job, CoroutineName, CoroutineDispatcher,  CoroutineExceptionHandler** 같은 **Element 객체들이 인덱싱 된 집합**)
    
- 각 Element 또한 CoroutineContext이다 (컨텍스트의 지정과 변경이 편리하다) → Element가 CoroutineContext를 상속
- Coroutine Context의 모든 Element가 CoroutineContext로 되어있어 컨텍스트의 지정과 변경이 편리해진다

```kotlin
fun main() {
    val name: CoroutineName = CoroutineName("A name")
    val element: CoroutineContext.Element=name
    // Element 또한 CoroutienContext를 상속하고 있기 때문
    val context: CoroutineContext=element

    val job: Job = Job()
    val jobElement: CoroutineContext.Element=job
    val jobContext: CoroutineContext=jobElement
}
/* CoroutineName, Job이 CoroutineContext의 Element이고 Element 자체로도
   CoroutineContext이다 */
```

- Context에서 모든 원소는 식별할 수 있는 유일한 Key를 가지고 있다. (주소로 비교)

## CoroutineContext에서 원소 찾기

- CoroutineContext는 컬렉션과 비슷하기 때문에 get을 이용하여 유일한 key를 가진 원소를 찾을 수 있다. ( [ ] 도 사용 가능→ 원소가 있으면 반환 없으면 null 반환)
- CoroutineName은 Companion Object로 Key를 구현하고 있기 때문에 ctx[CoroutineName]=ctx[CoroutineName.key]가 된다

```kotlin
data class CoroutineName(
		val name: String
) : AbstractCoroutineContextElement(CoroutineName) {

		override fun toString() : String = "CoroutineName($name)"
		
		// Companion Object가 CoroutineContext의 Key를 구현하고 있어
		// Class 이름을 Key 값 처럼 사용 가능
		companion object Key: CoroutineContext.Key<CoroutineName>
} 

// Job
interface Job: CoroutineContext.Element {
	companion object Key : CoroutineContext.Key<Job>
	...
}
```

- kotlinx.coroutines 라이브러리에서 Companion 객체를 key로 사용하여 같은 이름을 가진 원소를 찾는 것은 흔한 일이다 (**Class의 이름이 Companion Object에 대한 참조로 사용되는 Kotlin의 특징 활용)**

## 컨텍스트 더하기

- 두 개의 CoroutineContext를 합쳐 하나의 CoroutineContext로 만들 수 있다
    
    → **다른 키**를 가진 두 원소를 더하면 만들어진 Context는 두 가지 키를 모두 가진다
    
- CoroutineContext Interface의 `plus` 연산자 오버로딩 메서드로 인해 `+` 로 Context를 합칠 수 있음

```kotlin
fun main() {
    val ctx1: CoroutineContext = CoroutineName("Name1")
    println(ctx1[CoroutineName]?.name)
    println(ctx1[Job]?.isActive)

    val ctx2: CoroutineContext = Job()
    println(ctx2[CoroutineName]?.name)
    println(ctx2[Job]?.isActive)

    val ctx3=ctx1+ctx2
    println(ctx3[CoroutineName]?.name)
    println(ctx3[Job]?.isActive)
}
-----------
Name1
null
null
true
Name1
true
```

- Coroutine Context에 같은 Key를 가진 또 다른 원소가 더해지면 새로운 원소가 기존의 것을 대체한다

```kotlin
fun main() {
    val ctx1: CoroutineContext = CoroutineName("Name1")
    println(ctx1[CoroutineName]?.name)

    val ctx2: CoroutineContext = CoroutineName("Name2")
    println(ctx2[CoroutineName]?.name)

    val ctx3=ctx1+ctx2
    println(ctx3[CoroutineName]?.name)
}
----------
Name1
Name2
Name2
```

## 비어있는 Coroutine Context

- CoroutineContext는 Collection의 일종이므로 빈 Context도 만들 수 있다

```kotlin
fun main() {
	val empty : CoroutineContext = EmptyCoroutineContext
	println(empty[CoroutineName])
	println(empty[Job])
}
----------
null
null
```

## 원소 제거

- **minusKey** 함수에 key를 넣어 원소를 Context에서 제거할 수 있다
    - plus와 달리 minus 연산자 오버로딩을 하지 않고 `minusKey` 추상 메서드 제공

```kotlin
fun main() {
    val ctx= CoroutineName("Name1") + Job()
    println(ctx[CoroutineName]?.name)
    println(ctx[Job]?.isActive)

    val ctx2= ctx.minusKey(CoroutineName)
    println(ctx2[CoroutineName]?.name)
    println(ctx2[Job]?.isActive)

    val ctx3= (ctx+CoroutineName("Name2")).minusKey(CoroutineName)
    println(ctx3[CoroutineName]?.name)
    println(ctx3[Job]?.isActive)
}
---------------
Name1
true

null
true

null
true
```

## 컨텍스트 폴딩

- 컨텍스트의 각 원소를 조작해야하는 경우 **fold** 메서드 사용
- fold 메서드 (2가지 인자)
    - 누산기의 첫 번째 값
    - 누산기의 현재 상태와 실행되고 있는 원소로 누산기의 다음 상태를 계산할 연산 (람다)
        - acc : 지금까지 접어온 결과 값(누적 값)
        - element: 각 element

```kotlin
fun main() {
    val ctx= CoroutineName("Name1") + Job()
    ctx.fold("") {acc,element->"$acc$element"}
        .also(::println)

    val empty= emptyList<CoroutineContext>()
    ctx.fold(empty) { acc,element -> acc+element}
        .joinToString()
        .also(::println)
}
------------------
CoroutineNmae(Name1) JobImpl{Active}@~~~

CoroutineName(Name1), JobImpl{Active}@~~~
```

## 코루틴 Context와 Builder

- **CoroutineContext는 코루틴의 데이터를 저장하고 전달하는 방법**
- 부모-자식 관계의 영향 중 하나로 부모는 기본적으로 Context를 자식에게 전달한다

```kotlin
import kotlinx.coroutines.*

fun CoroutineScope.log(msg: String) {
    val name=coroutineContext[CoroutineName]?.name
    println("[$name] $msg")
}

fun main()= runBlocking(CoroutineName("main")) {
    log("Started")
		// 부모의 Context를 상속받는다
    val v1= async {
        delay(500)
        log("Running async")
        1
    }
		// 부모의 Context를 상속받는다
    launch {
        delay(2000)
        log("Running launch")
    }
    log("The answer is ${v1.await()}")
}
----------------
[main] Started
(0.5초 후)
[main] Running async
[main] The answer is 1
(2초 후)
[main] Running launch
```

- 각 빌더 마다 특정 Context를 가질 수 있다 (부모로 부터 상속 받은 Context를 대체)

```kotlin
package ch07

import kotlinx.coroutines.*

// log는 위와 동일

fun main()= runBlocking(CoroutineName("main")) {
    log("Started")
		// 빌더에서 정의된 특정 컨텍스트
    val v1= async(CoroutineName("async")) {
        delay(500)
        log("Running async")
        1
    }

    launch(CoroutineName("launch")) {
        delay(2000)
        log("Running launch")
    }
    log("The answer is ${v1.await()}")
}
// 출력 결과
[main] Started
(0.5초)
[async] Running async
[main] The answer is 1
(2초)
[launch] Running launch
```

 ⭐ Job Element만이 부모-자식 관계를 결정하기 때문에 자식 Coroutine Builder에서 `Job(), SupervisorJob()` Element를 추가하면 부모 Job이 대체되어 부모-자식 관계가 끊기게 된다

### 코루틴 Context 계산 공식

defaultContext + parentContext + childContext

- 새로운 원소가 같은 키를 가진 이전 원소를 대체하기 때문에

(부모로부터 상속 받은 Context 중 같은 key를 가진 원소를 대체한다)

- 디폴트 Context는 어디 서도 키가 지정되지 않았을 때만 사용된다
    - ContinuationInterceptor가 설정되지 않았을 땐 **Dispatcher.Default**가 기본으로 설정
    - Job Element는 부모-자식 Coroutine 간에 소통하기 위해 사용되는 특별한 Context
    - Debug 모드에선 CoroutineId도 디폴트로 설정된다

## 중단 함수에서 Context에 접근하기

- CoroutineContext는 중단 함수 사이에 전달되는 Continuation 객체가 참조하고 있다

```kotlin
suspend fun printName() {
    println(coroutineContext[CoroutineName]?.name)
}

suspend fun main() = withContext(CoroutineName("Outer")) { // 부모 측 Context
    printName()

    launch(CoroutineName("Inner")) { // 자식 측 Context : 같은 Key를 갖는 CoroutineName 덮어씀
        printName()
    }
    delay(1000)
    printName()
}
--------------
Outer
Inner
Outer
```

- withContext로 Coroutine이 실행되는 영역을 바꿀 수 있다.
    - (EX. Main 영역 → I/O 변경 등)

## Context를 개별적으로 생성하기 (Custom 하기)

- CoroutineContext.Element 인터페이스를 구현하는 클래스를 만들어라
    - **CoroutineContext.Key<*> 타입의 key Property 필요 (Context 식별 용)**
    
- 일반적인 Custom 방식

```kotlin
class MyCustomContext : CoroutineContext.Element {
    
    override val key: CoroutineContext.Key<*> = Key
		
		// Companion Object를 선언해야 .Key를 MyCustomContext로 참조 가능
		// <> 제너릭 타입에 Class 이름 넣기
    companion object Key : CoroutineContext.Key<MyCustomContext>
}
```

> Ex)
> 

```kotlin
class CounterContext(
    private val name: String
) : CoroutineContext.Element {
    override val key: CoroutineContext.Key<*> = Key
    private var nextNumber=0

    fun printNext() {
        println("$name : $nextNumber")
        nextNumber++
    }

    companion object Key:CoroutineContext.Key<CounterContext>
}

suspend fun printNext() {
    coroutineContext[CounterContext]?.printNext()
}

suspend fun main() : Unit =
    withContext(CounterContext("Outer")) {
        printNext()
        launch {
            printNext()
            launch {
                printNext()
            }
            launch(CounterContext("Inner")) {
                printNext()
                printNext()
                launch {
                    printNext()
                }
            }
        }
        printNext()
    }
-----------------
Outer : 0
Outer : 1
Outer : 2
Inner : 0
Inner : 1
Inner : 2
Outer : 3

같은 Key 값을 CounterContext를 자식에서 생성하여 대체되어 3,4,5가 아닌 0,1,2가 찍힌다
```

---

## 요약

- CoroutineContext는 Map이나 Set과 같은 컬렉션이랑 개념적으로 비슷
- **Element 인터페이스의 인덱싱 된 집합이며 Element 또한 CoroutineContext 이다**
    - Element : Job, CoroutineName, CoroutineDispatcher 등
- CoroutineContext를 식별할 때 사용되는 **유일한 Key**를 갖고 있다
- CoroutineContext는 코루틴에 관련된 정보를 객체로 그룹화 하고 전달하는 보편적 방법
    - **코루틴 실행 환경을 구성하는 Key-Element 들의 집합이다**
- CoroutineContext는 코루틴에 저장되며 코루틴의 상태, 어떤 스레드 선택할 지 등 코루틴의 작동 방식을 정할 수 있다
    
    ⭐ 즉 실행 환경을 구성/제어하는 핵심 도구이다
    
    > 활용 예시
    > 
    1. Job Element를 활용하여 ViewmodelScope, LifecycleScope에서 자식들 작업 간 격리할 때
        
        ```kotlin
        class MyViewModel : ViewModel() {
            private val handler = CoroutineExceptionHandler { _, e -> log(e) }
            
            fun loadBoth() = viewModelScope.launch(handler) {
                supervisorScope {
                    launch { repo.loadA() } // A 실패해도
                    launch { repo.loadB() } // B는 계속
                }
            }
        }
        ```
        
    
    1. 이전 작업은 취소하고 새 작업만을 살리고자 할 때
        
        ```kotlin
        var searchJob: Job? = null
        fun onQuery(q: String) {
            searchJob?.cancel() // 취소
            searchJob = viewModelScope.launch { search(q) }
        }
        ```
        
    
    1. CoroutineName을 사용하여 로깅 및 디버깅에 활용
        
        ```kotlin
        viewModelScope.launch(CoroutineName("LoadProfile")) {
            println("${coroutineContext[CoroutineName]} started")
        }
        ```
        
    
    1. CoroutineExceptionHandler로 예외처리 및 로깅
        
        ```kotlin
        val handler = CoroutineExceptionHandler { _, e -> reportCrash(e) }
        viewModelScope.launch(handler) {
        	// launch 빌더에 handler를 넣으면 빌더 내에서 던져진 예외는 handler로 넘어간다
        }
        ```
        

---

## 래퍼런스

- 공식문서
    - https://kotlinlang.org/api/core/kotlin-stdlib/kotlin.coroutines/-coroutine-context/
