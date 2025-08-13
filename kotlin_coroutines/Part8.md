# Job과 자식 코루틴 기다리기

## 부모-자식 Coroutine 관계의 특성

- 자식은 부모로부터 Context를 상속 받는다.
- 부모는 모든 자식이 작업을 마칠 때까지 기다린다.
- 부모 Coroutine이 취소되면 자식 Coroutine도 취소된다.
- 자식 Coroutine에서 에러가 발생하면 부모 Coroutine 또한 에러로 소멸한다.

**⭐ 자식이 부모로부터 Context를 물려받는 것은 Coroutine Builder의 가장 기본적인 특징이다 (1번)**

→ 2,3,4번 특성은 Job CoroutineContext와 관련이 있다

## Job

- Coroutine을 취소하고 상태를 파악하는 등 다양하게 사용될 수 있다
- **Coroutine은 각자의 Job을 가지고 있다.**

### Job이란?

- 수명을 가지고 있으며 취소 가능한 Interface이다 (추상 Class 처럼 다룰 수 있음)
- Job의 수명은 상태로 나타낸다 (하단의 그림 참조)

<img width="856" height="892" alt="Image" src="https://github.com/user-attachments/assets/1d360ad9-59c8-41b6-8d63-9a47e4f3d21a" />

### Active

- Active 상태에서는 Job이 실행되고 Coroutine은 Job을 수행한다.
    - Job이 Coroutine Builder에 의해 생성되었을 때 Coroutine의 본체가 실행되는 상태
    
    → 이 상태일 때 자식 Coroutine을 시작할 수 있다
    

**⭐ 일반적으로 대부분의 Coroutine은 생성되는 즉시 Active 상태가 된다 (지연 시작 Coroutine 제외)**

### New

- 지연 시작되는 Coroutine만 New 상태에서 시작된다
    - Active 상태가 되기 위해선 작업이 실행되어야 한다
    - `launch(start = CoroutineStart.Lazy)` 과 같이 생성하며 start() 메서드와 같이 시작하는 함수를 호출해줘야한다

### Completing

- 실행이 완료되면 Completing으로 바뀌고 자식들을 기다린다

### Completed

- 나의 실행이 종료되고 자식들의 실행도 모두 끝난 상태

### Cancelling

- Active 상태 또는 Completing 에서 Job이 실행 도중에 취소되거나 실패하면 나오는 상태
    - 이 상태에서 연결을 끊거나 자원을 반납하는 등의 후 처리 작업을 할 수 있다

### Cancelled

- Cancelling 상태에서 연결 끊기, 자원 반납 등의 처리 작업이 완료된 상태

> Ex)
> 

```kotlin
import kotlinx.coroutines.*

suspend fun main() = coroutineScope {
    val job= Job()
    println(job)

    job.complete()
    println(job)

    val activeJob=launch {
        delay(1000L)
    }

    println(activeJob)
    activeJob.join()
    println(activeJob)

    val lazyJob=launch(start=CoroutineStart.LAZY) {
        delay(1000L)
    }

    println(lazyJob)
		// 실행을 해줘야 Active 상태가 된다
    lazyJob.start()
    println(lazyJob)
    lazyJob.join()
    println(lazyJob)
}
------------------
JobImpl{Active}@484b61fc
JobImpl{Completed}@484b61fc
StandaloneCoroutine{Active}@47d384ee
(1초)
StandaloneCoroutine{Completed}@47d384ee
LazyStandaloneCoroutine{New}@2000d4d
LazyStandaloneCoroutine{Active}@2000d4d
(1초)
LazyStandaloneCoroutine{Completed}@2000d4d
```

- job을 직접 생성하면 JobImpl 로 나타난다
- launch는 내부적으로 StandAloneCoroutine을 반환한다.
- **join() 함수는 Coroutine Builder의 내부가 실행을 끝마칠 때까지 기다리는 함수이다**
    - 코루틴이 완료되는 걸 기다리기 위해 사용하는 함수
- lazyjob을 생성하는 부분에서 (CoroutineStart.LAZY)로 지연 Coroutine을 생성한다

### Coroutine Job의 상태 확인 방법

- isActive, isCompleted, isCancelled Property 활용

| 상태 | isActive | isCompleted | isCancelled |
| --- | --- | --- | --- |
| NEW (지연 시작될 때) | false | false | false |
| Active (시작 상태 기본 값) | true | false | false |
| Completing (일시적인 상태) | true | false | false |
| Cancelling (일시적인 상태) | false | false | true |
| Cancelled (최종 상태) | false | true | true |
| Completed (최종 상태) | false | true | false |

## Coroutine Builder는 부모의 잡을 기초로 자신들의 Job을 생성한다

**⭐ 모든 Coroutine Builder는 자신만의 Job을 생성한다**

- 대부분의 Coroutine Builder는 Job을 반환 (withContext, runBlocking 제외)
    - launch : Job 반환
    - async :  Deferred 반환 (Job 인터페이스를 상속)
- Job은 Coroutine Context 이므로 coroutineContext[Job]을 이용하여 접근하거나 확장 Property 인 job을 이용하는 방법이 있다.

```kotlin
import kotlinx.coroutines.Job
import kotlinx.coroutines.runBlocking
import kotlin.coroutines.CoroutineContext

// 확장 Property
val CoroutineContext.job : Job
		// 확장 Property를 사용해서 Coroutine Context의 get 메서드로 Context 가져오거나
		// null인 경우 에러 발생시키거나
    get() = get(Job) ?: error("Current Context not")

fun main() : Unit = runBlocking {
    println(coroutineContext.job.isActive)
}
------------
true
```

⭐ Job은 Coroutine이 상속하지 않는 유일한 Coroutine Context이며 아주 중요한 법칙이다

- 모든 Coroutine은 자신만의 Job을 생성하며 **인자 또는 부모 Coroutine으로부터 온 Job은 새로운 Job의 부모로 사용된다.**

```kotlin
import kotlinx.coroutines.CoroutineName
import kotlinx.coroutines.Job
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking

fun main() : Unit = runBlocking {
    val name = CoroutineName("Some name")
    val job= Job() // Job Factory로 Job 생성
		
		// Builder 안에 인자로 넘어오기 때문에 새로운 Job의 부모가 된다
    launch(name+job) {
        val childName=coroutineContext[CoroutineName]
        println(childName==name) // Context의 이름은 유지가 된다

        val childJob=coroutineContext[Job] // launch Builder에서 생성한 Job
        println(childJob==job)
        println(childJob==job.children.first())
    }
}
---------------
true
false
true
```

- 자식 Coroutine에 인자로 name+job가 들어간다
- childName과 name이 같은 것으로 보아 Coroutine Name이 상속되었다
- 자식 Coroutine에서 자신의 독자적인 Job을 생성하였고 인자로 전달된 job은 childJob의 부모로 사용되는 것을 맨 마지막 줄에서 볼 수 있다.
- 자식 코루틴에서 부모 코루틴을 참조했다

```kotlin
import kotlinx.coroutines.Job
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking

fun main() : Unit = runBlocking {
    val job: Job =launch {
        delay(1000L)
    }

    val parentJob: Job=coroutineContext.job
    println(job==parentJob)

    val parentChildren : Sequence<Job> = parentJob.children
    println(parentChildren.first() == job)
}
-----------
false
true
```

- 부모 job이 자식 job을 참조한 것을 볼 수 있다. (parentJob.children)
    - Job을 참조할 수 있는 부모-자식 관계가 있기 때문에 Coroutine Scope 내에서 취소와 예외 처리 구현이 가능하다.
    

> 새로운 Job Context가 부모의 Job을 대체한 경우 (구조화 동시성 깨짐)
> 

```kotlin
fun main() : Unit = runBlocking {
	launch(Job()) {
		delay(1000L)
		println("Will not be printed")
	}
}
-----------------
아무것도 출력되지 않음
```

- 자식 Coroutine은 인자로 들어온 Job을 부모로 사용한다.
- 새로운 Job Context가 부모의 Job을 대체하면 구조화된 동시성의 작동 방식이 유효하지 않다
    - 부모-자식 사이에 아무런 관계가 없기 때문에 **부모가 자식 Coroutine을 기다리지 않는다**

**⭐ Coroutine이 자신만의 독자적인 Job을 가지고 있으면 부모와 아무런 관계가 없다고 할 수 있다**

⇒ 구조화된 동시성을 잃게 되어 Coroutine을 다루기 어려워 진다.

## 자식들 기다리기

- Job의 중요한 이점 중 하나는 Coroutine이 완료될 때까지 기다리는데 활용할 수 있다는 것
    - **join() 메서드 활용 (지정한 Job이 Completed나 Cancelled 같은 완료 상태에 도달할 때 까지 기다리는 중단 함수)**

```kotlin
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking

fun main() : Unit = runBlocking {
    val job1=launch {
        delay(1000L)
        println("Test1")
    }

    val job2=launch {
        delay(2000L)
        println("Test2")
    }

    job1.join()
    job2.join()
    println("All Clear")
}
-------------
(1초)
Test1
(1초)
Test2
All Clear
```

- join( )을 통해 launch Builder 내부가 모두 끝날 때 까지 대기

- Job 인터페이스는 모든 자식을 참조할 수 있는 children Property를 노출 시킨다.
    - 모든 자식이 마지막 상태가 될 때까지 기다리는데 활용 가능

```kotlin
import kotlinx.coroutines.Job
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking

fun main() : Unit = runBlocking {
    launch {
        delay(1000)
        println("Test1")
    }

    launch {
        delay(2000L)
        println("Test2")
    }
		
		// 모든 참조할 수 있는 Coroutine Context 중 Job을 가져왔다
		// Sequence<Job> 형태
    val children = coroutineContext[Job]?.children

    val childrenNum=children?.count()
    println("Number of Children is $childrenNum")
    children?.forEach { it.join() }
    println("All Clear")
}
-------------------
Number of Children is 2
(1초)
Test1
(1초)
Test2
All Clear
```

- children Property로 현재 포함된 children를 모두 참조할 수 있다
- forEach { it.join() } 을 통해 모든 자식이 종료 상태가 되는 것을 기다릴 수 있다.

## Job Factory 함수

- Job은 Job( ) 함수 (Factory 함수)를 사용하면 Coroutine 없이 Job을 만들 수 있다
    - 생성자처럼 보이지만 함수
    - **Job은 인터페이스이기 때문에 생성자를 갖지 못한다 (CompletableJob 반환)**
        - completableJob : Job의 확장 인터페이스
        
        ```kotlin
        public fun Job(parent: Job?=null): CompletableJob
        ```
        
- Job( ) 함수로 생성한 **Job은 어떤 Coroutine과도 연관되지 않으며** Context로 활용될 수 있다.
    - 1개 이상의 자식 Coroutine을 가진 부모 Job으로 사용할 수 있다.
    - Job() 으로 생성된 Job을 컨텍스트에 넣어 **스코프의 “부모 Job”** 으로 쓰면, 그 스코프에서 `launch/async`로 만든 **여러 자식 코루틴**을 한꺼번에 관리(취소/대기)할 수 있다
    - 즉 **빌더를 대체하는 게 아니라**, *빌더가 붙을 스코프의 부모를 직접 만들어 주는* 용도입니다.

**주의**❗ Job( ) Factory 함수로 Job을 생성하고 다른 Coroutine의 부모로 지정한 뒤 join( )을 호출하면 자식 Coroutine이 작업을 끝마쳐도 Job이 Active한 상태에 있기 때문에 프로그램이 종료되지 않는다

**⇒ Factory 함수로 만들어진 Job은 다른 Coroutine에 의해 여전히 사용될 여지가 있기 때문**

```kotlin
import kotlinx.coroutines.Job
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

suspend fun main(): Unit = coroutineScope {
    // Job() Factory 함수 활용
    val job= Job()
    launch(job) {
         delay(1000L)
        println("TEXT 1")
    }
    launch(job) {
        delay(2000)
        println("TEXT 2")
    }
    // Job() 으로 생성되어 Active 한 상태 유지
    job.join()
    println("Will not be Printed")
}
---------------
TEXT 1
TEXT 2
(멈추지 않고 영원히 실행)
```

- 위를 해결하려면 Job의 모든 자식 Coroutine 에서 join( )을 호출하는 것이 바람직한 방법이다

```kotlin
import kotlinx.coroutines.Job
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

suspend fun main(): Unit = coroutineScope {
    // Job() Factory 함수 활용
    val job= Job()
    launch(job) {
         delay(1000L)
        println("TEXT 1")
    }
    launch(job) {
        delay(2000)
        println("TEXT 2")
    }
    job.children.forEach { it.join() }
    println("Finish")
}
----------------
Text 1
Text 2
Finish
```

## CompletableJob 인터페이스

- 두  가지 Method를 추가하여 Job Interface 확장

### complete() : Boolean

- job을 완료하는데 사용
    - 모든 자식 Coroutine은 **작업이 완료될 때까지** 실행된 상태를 **유지**
    - complete를 호출한 job에서 새로운 Coroutine이 시작될 수 없다.
    - job이 완료되면 true 그렇지 않을 경우(이미 완료된 경우) false

```kotlin
import kotlinx.coroutines.Job
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking

fun main() = runBlocking {
    val job= Job()

    launch(job) {
       repeat(5) {num->
           delay(200L)
           println("Repeat $num")
       }
    }

    launch(job) {
        delay(500L)
				// complete 호출
        job.complete()
    }

    job.join()
		
		// 이미 Complete 된 부모 job에 새로운 Coroutine이 붙으나 동작하지 않음
    launch(job) {
        println("Will not be Printed")
    }

    println("Done")
}
--------------
Repeat 0
Repeat 1
Repeat 2
Repeat 3
Repeat 4
Done
```

- complete가 호출되어 join( )을 만나 모든 작업이 완료될 때 까지 기다려 Repeat 3 이후도 호출된다
- job에서 complete( )를 호출했기 때문에 마지막 launch Builder로 만든 새로운 Coroutine이 시작될 수 없다.

### completeExceptionally(exception: Throwable) : Boolean

- 인자로 받은 예외로 잡을 완료 시킨다.
- 모든 자식 Coroutine은 주어진 예외를 Wrapping 한 CancellationException으로 **즉시 취소된다**

```kotlin
import kotlinx.coroutines.Job
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking

fun main() = runBlocking {
    val job= Job()

    launch(job) {
        repeat(5) { num->
            delay(200L)
            println("Repeat $num")
        }
    }

    launch {
        delay(500L)
        // 코루틴이 즉시 취소 된다
        job.completeExceptionally(Error("Some error occured"))
    }

    job.join()

    launch(job) {
        println("Will not be printed")
    }

    println("Done")
}
-----------------
Repeat 0
Repeat 1
Done
```

- 0.5초 후 completeExceptionally 를 만나 모든 자식 Coroutine이 죽는다.
    - Repeat 0,1 까지 호출되고 이후 작업은 X

## 이외의 활용

- complete 함수는 job의 마지막 Coroutine을 시작한 후 자주 사용된다
    - complete( ) 호출 후 join( ) 함수를 활용해 job이 완료되는 것을 기다리면 된다

```kotlin
import kotlinx.coroutines.Job
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

suspend fun main() : Unit = coroutineScope {
    val job= Job()
    launch(job) {
        delay(1000L)
        println("Text 1")
    }

    launch(job) {
        delay(1500L)
        println("Text 2")
    }
    println(job.complete().toString())
    job.complete()
    job.join()
    println(job.complete().toString())
}
----------------
true
Text 1
Text 2
false
```

- complete 가 호출되기 전에 왜 job.complete( )가 true가 되는지

<img width="822" height="220" alt="Image" src="https://github.com/user-attachments/assets/7e1e5d82-1590-4331-a8f8-a60e75fe9d16" />

- Job 함수의 인자로 부모 Job의 참조 값을 전달할 수 있다. 이때 부모 Job이 취소되면 해당 Job 또한 취소된다.

```kotlin
import kotlinx.coroutines.Job
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

suspend fun main() : Unit = coroutineScope {
    val parentJob= Job()
    val job=Job(parentJob)

    launch(job) {
        delay(1000L)
        println("Text 1")
    }

    launch(job) {
        delay(2000L)
        println("Text 2")
    }
    // 2000 으로 걸면 코루틴의 text2가 출력될 수도 있고 안될 수 있다.
    delay(1100)
    parentJob.cancel()
    job.children.forEach { it.join() }
}

// 출력 결과
Text 1
```

- parentJob.cancel( )에 의해 부모 Job이 취소되어 main의 delay보다 Coroutine의 delay가 길면 해당 Coroutine은 출력 되지 않는다

