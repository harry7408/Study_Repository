# Hot Data Source & Cold Data Source

## Hot vs Cold

- Hot Data Source
    - 데이터를 소비하는 것과 무관하게 원소를 생성
    - 한식 뷔페에서 음식을 차려 놓으면 손님들이 소비하는 상황
    - List, Set 과 같은 컬렉션
    - Channel
    - **StateFlow, ShareFlow**
    
    → Hot Data Stream의 빌더와 연산은 **즉각 실행된다**
    

- Cold Data Source
    - 요청이 있을 때만 작업을 수행하며 아무것도 저장하지 않는다
    - 식당에서 손님의 주문이 들어가면 조리가 시작되는 상황
    - Sequence, Stream
    - **Flow**, RxJava(Observable, Single)
    
    → Cold Data Stream에서는 **원소가 필요할 때까지 실행되지 않는다**
    

> Ex) Hot & Cold 예시
> 

```kotlin
fun main() {
    val l = buildList {
        repeat(3) {
            println("L: Adding User$it")
            add("User$it")
        }
    }

    val l2 = l.map {
        println("L: Processing $it")
        "Processed $it"
    }
		
		// Teminal 연산이 없어 동작하지 않는다
    val s = sequence {
        repeat(3) {
            println("S: Adding User$it")
            yield("User$it")
        }
    }

    val s2 = s.map {
        println("S: Processing $it")
        "Processed $it"
    }
}
---------------
L: Adding User0
L: Adding User1
L: Adding User2
L: Processing User0
L: Processing User1
L: Processing User2
```

- Hot Data Stream은 즉시 실행되며 중간 연산 결과가 매번 생성되어 메모리 사용량이 증가한다
    
    → 항상 데이터가 사용 가능한 상태이며 여러 번 사용되었을 때 매번 결과를 다시 계산할 필요가 없다
    
- Cold Data Stream은 중간 변환 규칙만 기억하고 Terminal 연산이 호출될 때 중간 결과를 저장하지 않고 하나씩 흘려보내 최종 결과를 낸다
    
    → 무한할 수 있으며 최소한의 연산만 수행 및 메모리 사용량 절약
    

> Ex) list와 sequence 연산 비교
> 

```kotlin
fun m(i: Int): Int {
    print("m$i ")
    return i * i
}

fun f(i: Int): Boolean {
    print("f$i ")
    return i >= 10
}

fun main() {
    listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .map { m(it) }
        .find { f(it) }
        .let { print(it) }

    println()

    sequenceOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .map { m(it) }
        .find { f(it) }
        .let { print(it) }
}
-----------------------
m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 f1 f4 f9 f16 16
m1 f1 m2 f4 m3 f9 m4 f16 16
```

- list의 연산 과정
    - 첫 번째 map 함수에서 m1~m10까지 출력 후 1~10까지의 제곱 수가 List로 저장된다
    - 2번째 find 함수에서 f1~f4까지 출력되며 16을 만날 때 true가 되므로 탐색이 종료되고 16이 저장된다
    - 마지막 let에서 null이 아니므로 16이 출력된다
    
- sequence 연산 과정
    - Terminal 함수인 find에서 연산이 시작되고 1부터 값이 순차적으로 들어오며 map, find Body의 동작이 시작된다
    - find의 첫 true 반환 시점인 4일때 까지만 m1 ~ m4, f1 ~ f16이 출력되며 16이 저장된다
    - let 함수에서 null이 아니므로 16이 출력된다
      
```
💡 Terminal Operation

- 변환/수집
    - toList(), toSet(), toMap()
    - joinToString()
    - associate { }, associateBy { }
- 소비
    - forEach { }, forEachIndexed { }
    - onEach { }+toList()
- 단일 값 반환
    - first(), firstOrNull()
    - last(), lastOrNull()
    - single(), singleOrNull()
    - elementAt(idx)
    - find { }, any { }, all { }, none { }
- 집계 연산
    - count()
    - sum(), sumOf { }
    - average()
    - min(), max(), minBy { }, maxBy { }
- 리듀스/폴드
    - reduce { }
    - fold(initial) { }

💡 Intermediate Operation

- map, filter, flatMap, drop …
  
```

</br>

## Hot Channel, Cold Flow

- Channel은 Hot Data Stream이며 일반적으로 produce 함수로 생성
    
    ```kotlin
    val channel = produce {
    	while(true) {
    		val x = compute()
    		send(x)
    	}
    }
    ```
    
    - 값을 곧바로 계산하며 별도의 코루틴에서 수행해야한다 (CoroutineScope의 확장 함수로 정의 되어있는 빌더가 필요)
    
    > Ex) Channel 예시
    > 
    
    ```kotlin
    private fun CoroutineScope.makeChannel() = produce {
        println("Channel started")
        for (i in 1..3) {
            delay(1000)
            send(i)
        }
    }
    
    suspend fun main() = coroutineScope {
        val channel = makeChannel()
    
        delay(1000)
        // 첫 번째 소비자
        println("Calling channel...")
        for (value in channel) {
            println(value)
        }
        // 두 번째 소비자
        println("Consuming again...")
        for (value in channel) {
            println(value)
        }
    }
    Channel started
    Calling channel...
    1
    2
    3
    Consuming again...
    // 값을 모두 소비하여 더이상 출력되지 않음
    ```
    
    - Channel은 Hot Stream이기 때문에 값을 생성한 뒤에 값을 가지게된다
    - 각 원소는 단 1번만 받을 수 있어 첫 번째 소비자가 값을 사용하면 두 번째 소비자는 채널이 비어있고 닫힌 것을 확인할 수 있다

- Flow는 Clod Data Stream이며 produce와 유사하기 flow Builder를 사용하여 생성
  
    
    ```kotlin
    val flow = flow {
    	while(true) {
    		val x = compute()
    		emit(x)
    	}
    }
    ```
    
    - flow는 Cold Data Stream이며 내부에 최종 연산이 호출될 때 원소가 어떻게 생성되어야 하는지 정의한 것에 불가하다
    - flow 빌더는 CoroutineScope가 필요하지 않고 빌더를 호출한 최종 연산의 Scope 내에서 실행된다
    
    > Ex) Flow 예시
    >
    
    
    ```kotlin
    private fun makeFlow() = flow {
        println("Flow started")
        for (i in 1..3) {
            delay(1000)
            // 구독할 때 이 값을 써
            emit(i)
        }
    }
    
    suspend fun main() = coroutineScope {
        val flow = makeFlow()
    
        delay(1000)
        println("Calling flow...")
        // 구독
        flow.collect { value -> println(value) }
        println("Consuming again...")
        flow.collect { value -> println(value) }
    }
    --------------
    Calling flow...
    Flow started
    1
    2
    3
    Consuming again...
    Flow started
    1
    2
    3
    ```
    

---

## 요약

- 대부분의 데이터는 Hot Data Source거나 Cold Data Source이다
- Hot Data Source는 가능한 빨리 원소를 만들고 저장하며 원소가 소비되는 것과 무관하게 생성한다
- Cold Data Source는 Terminal 연산에서 값이 필요할 때 처리가 시작되고 원소를 저장하지 않고 연산을 최소한으로 수행한다
- Hot Data Source는 한식 뷔페, Cold Data Source는 일반 식당을 떠올리면 된다
- Hot Data Stream 중 Channel은 send 즉시 데이터가 파이프에 들어가고 이는 생방송 라디오럼 DJ가 말하는 즉시 송출이 되는 것과 유사하다
    
    → 저장되지 않고 소비하는 즉시 증발
    
- Cold Data Stream 중 Flow는 내부에 어떤 동작을 하는지 정의하고 요리 레시피처럼 emit에서 이렇게 요리를 만들라는 가이드만 제시해 주고 실제 동작은 collect 가 호출될 때 시작된다
    
    → 구독할 때마다 다시 실행되어 여러 소비자가 있어도 동일한 값을 받을 수 있다
