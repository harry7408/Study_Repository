# 코루틴 컨텍스트

- 코루틴 빌더의 첫 번째 파라미터가 CoroutineContext이다
- CoroutineScope 인터페이스 내부에도 CoroutineContext가 감싸져 있다
- Continuation 객체도 CoroutineContext를 포함하고 있다

**→ 중요한 요소들이 CoroutineContext를 사용하고 있다**

## CoroutineContext

- 원소나 원소들의 집합을 나타내는 **인터페이스 (일종의 컬렉션)**
    
    **(Job, CoroutineName, CoroutineDispatcher** 같은 Element 객체들이 인덱싱 된 집합)
    
- 각 Element 또한 CoroutineContext이다 (컨텍스트의 지정과 변경이 편리하다)

```kotlin
fun main() {
    val name: CoroutineName = CoroutineName("A name")
    val element: CoroutineContext.Element=name
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
- CoroutineName은 Companion Object이기 때문에 ctx[CoroutineName]=ctx[CoroutineName.key]가 된다

```kotlin
data class CoroutineName(
		val name: String
) : AbstractCoroutineContextElement(CoroutineName) {

		override fun toString() : String = "CoroutineName($name)"

		companion object Key: CoroutineContext.Key<CoroutineName>
} 

// Job
interface Job: CoroutineContext.Element {
	companion object Key : CoroutineContext.Key<Job>
	...
}
```

- kotlinx.coroutines 라이브러리에서 Companion 객체를 key로 사용하여 같은 이름을 가진 원소를 찾는 것은 흔한 일이다.

## 컨텍스트 더하기

- 두 개의 CoroutineContext를 합쳐 하나의 CoroutineContext로 만들 수 있다
    
    → 다른 키를 가진 두 원소를 더하면 만들어진 Context는 두 가지 키를 모두 가진다
    

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
// 출력 결과
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
// 출력 결과
Name1
Name2
Name2
```

## 원소 제거

- **minusKey** 함수에 key를 넣어 원소를 Context에서 제거할 수 있다

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
```

## 컨텍스트 폴딩

- 컨텍스트의 각 원소를 조작해야하는 경우 **fold** 메서드 사용
- fold 메서드 (2가지 인자)
    - 누산기의 첫 번째 값
    - 누산기의 현재 상태와 실행되고 있는 원소로 누산기의 다음 상태를 계산할 연산 (메서드)

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
```

## 코루틴 Context와 Builder

- **CoroutineContext는 코루틴의 데이터를 저장하고 전달하는 방법**
- 부모-자식 관계의 영향 중 하나로 부모는 기본적으로 Context를 자식에게 전달한

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
// 출력 결과
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

### 코루틴 Context 계산 공식

defaultContext + parentContext + childContext

- 새로운 원소가 같은 키를 가진 이전 원소를 대체하기 때문에

(부모로부터 상속 받은 Context 중 같은 key를 가진 원소를 대체한다)

- 디폴트 Context는 어디 서도 키가 지정되지 않았을 때만 사용된다
    - ContinuationInterceptor가 설정되지 않았을 땐 Dispatcher.Default가 기본으로 설정
    - Debug 모드에선 CoroutineId도 디폴트로 설정된다

## 중단 함수에서 Context에 접근하기

- CoroutineContext는 중단 함수 사이에 전달되는 Continuation 객체가 참조하고 있

```kotlin
suspend fun printName() {
    println(coroutineContext[CoroutineName]?.name)
}

suspend fun main() = withContext(CoroutineName("Outer")) {
    printName()

    launch(CoroutineName("Inner")) {
        printName()
    }
    delay(1000)
    printName()
}
// 출력 결과
Outer
Inner
Outer
```

- withContext로 Coroutine이 실행되는 영역을 바꿀 수 있다.

(EX. Main 영역 → I/O 변경 등)

## Context를 개별적으로 생성하기 (Custom 하기)

- CoroutineContext.Element 인터페이스를 구현하는 클래스를 만들어라
    - CoroutineContext.Key<*> 타입의 key Property 필요 (Context 식별 용)
    
- 일반적인 Custom 방식

```kotlin
class MyCustomContext : CoroutineContext.Element {
    
    override val key: CoroutineContext.Key<*> = Key

    companion object Key : CoroutineContext.Key<MyCustomContext>
}
```

**Ex**

```kotlin
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import kotlin.coroutines.CoroutineContext
import kotlin.coroutines.coroutineContext

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
// 출력 결과
Outer : 0
Outer : 1
Outer : 2
Inner : 0
Inner : 1
Inner : 2
Outer : 3

Inner 3개와 Outer 3은 뭐가 먼저 호출될지 실행할 때마다 다르
```

---

## 요약

- CoroutineContext는 Map이나 Set과 같은 컬렉션이랑 개념적으로 비슷
- Element 인터페이스의 인덱싱 된 집합이며 Element 또한 CoroutineContext 이다
    - Element : Job, CoroutineName, CoroutineDispatcher 등
- CoroutineContext를 식별할 때 사용되는 **유일한 Key**를 갖고 있다
- CoroutineContext는 코루틴에 관련된 정보를 객체로 그룹화 하고 전달하는 보편적 방법
- 코루틴에 저장되며 코루틴의 상태, 어떤 스레드 선택할 지 등 코루틴의 작동 방식을 정할 수 있다
