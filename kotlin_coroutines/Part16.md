# Channel

- 코루틴 간의 통신을 위한 기본적인 방법
- 동작 방법은 도서관처럼 책을 찾고 있다면 누가 책장에 가지고 와야한다
- 또한 책을 빌리고 반납하는 사람의 제한이 없듯이 채널도 **송신자와 수신자의 수에 제한이 없다**
- 채널을 통해 전송되는 모든 값은 **단 1번만 받을 수 있다**

<img width="1104" height="122" alt="Image" src="https://github.com/user-attachments/assets/7211f414-9290-4688-9b63-a198efc50cc8" />
2개의 인터페이스를 구현한 하나의 인터페이스

- SendChannel : 원소를 보내거나 채널을 닫는 용도로 사용
    
    ```kotlin
    interface SendChannel<in E> {
    	
    	suspend fun send(element: E) // 채널에 element 전송
    	abstract fun trySend(element: E) : ChannelResult<Unit>
    	...
    	
    	fun close() : Boolean
    	
    }
    ```
    
    - 원소를 보낼 때 중단함수를 호출한다
    - 채널에 용량이 다 찬 경우 중단 (도서관 책장에 꽂을 곳이 없으면 사서 분들이 보관해뒀다가 책장이 비면 꽂아 넣는 상황)
    
- ReceiveChannel : 원소를 받거나 꺼낼 때 사용
    
    ```kotlin
    interface ReceiveChannel<out E> {
    	
    	suspend fun receive() : E // 채널에서 element 수신 후 채널에서 제거
    	abstract fun tryReceive() : ChannelResult<E>
    	...
    	
    	fun cancel(cause: CancellationException? = null) 	
    }
    ```
    
    - 원소를 받을 때 중단함수를 호출한다
    - receive를 호출했는데 채널에 원소가 없다면 원소가 들어올 때 까지 중단 (도서관에 책이 없으면 반납할 때까지 기다려야 하는 상황)
    
    💡 중단 함수가 아닌 송/수신을 해야할 때는 trySend/tryReceive를 사용할 수 있으며 이는 연산이 성공했는지 실패했는지에 대한 정보(ChannelResult)를 가지고 있다. (버퍼 용량이 제한적인 채널에서만 두 함수를 사용 가능하다)
    

- 채널의 송신자와 수신자의 수에 제한이 없지만 일반적으로 채널의 양 쪽 끝에 송신자/수신자 역할을 하는 코루틴이 각각 1개씩 있는 경우가 일반적이다

> Ex) 1개의 송신자/수신자가 존재하는 상황
> 

```kotlin
	suspend fun main(): Unit = coroutineScope {
    val channel = Channel<Int>()
	   // 송신자 역할 코루틴
    launch {
        repeat(5) { index ->
            delay(1000)
            println("Producing next one")
            channel.send(index * 2) // 데이터 송신
        }
    }
		
		// 수신자 역할 코루틴
    launch {
        repeat(5) {
            val received = channel.receive() // 데이터 수신 
            println(received)
        }
    }
}
--------------
Producing next one
0
Producing next one
2
Producing next one
4
Producing next one
6
Producing next one
8
```

- 위의 방식은 수신자 측에서 송신자가 얼마나 많은 원소를 보내는지 모른다
- 하여 송신자가 원소를 보내는 만큼 수신자가 기다리는 방식이 선호된다
- `consumeEach` 또는 for 문으로 파악 가능

> Ex) consumeEach 활용
> 

```kotlin
suspend fun main(): Unit = coroutineScope {
    val channel = Channel<Int>()
    launch {
        repeat(5) { index ->
            println("Producing next one")
            delay(1000)
            channel.send(index * 2)
        }
        channel.close()
    }

    launch {
		    // for 문으로 대기
        for (element in channel) {
            println(element)
        }
        // consumeEach로 대기 
        /* channel.consumeEach { element ->
             println(element)
         } */
    }
}
-------------------
// Producing next one이 연달아 찍힌 것은 다음 반복 시작이
// 0을 출력하는 것보다 먼저 스케쥴링된 현상
Producing next one
Producing next one
0
Producing next one
2
Producing next one
4
Producing next one
6
8
```

- 이 방식은 close를 닫아줘야 하는데 이를 까먹을 수 있다는 단점이 있다
- 예외가 발생하여 코루틴이 중단된다면 receive하려는 코루틴은 영원히 대기하게 되는 문제가 발생할 수 있다
- produce 메서드를 활용한다면 block 내에서 ProducerScope를 리시버로 받게 되는데 이는 완료되거나 취소될 때 반드시 close를 호출하고 있어 반드시 채널을 닫아준다

<img width="660" height="340" alt="Image" src="https://github.com/user-attachments/assets/80944ca2-e00e-48f9-b5bc-bac2518e8ad3" />

> Ex) produce Builder 활용
> 

```kotlin
suspend fun main(): Unit = coroutineScope {
    val channel = produce { // produce 빌더 활용
        repeat(5) { index ->
            println("Producing next one")
            delay(1000)
            send(index * 2)
        }
    }

    for (element in channel) {
        println(element)
    }
}
---------------
Producing next one
0
Producing next one
2
Producing next one
4
Producing next one
6
Producing next one
8
```

</br>

## 채널 타입

- 4가지 채널 타입이 존재
    - Unlimited : Channel.UNLIMITED로 설정된 제한이 없는 용량 버퍼를 가진 채널로 send가 중단되지 않는다
    - Buffered : Channel.BUFFERED 또는 특정 용량으로 설정된 채널
    - 랑데뷰 : Channel.RENDEZVOUS 또는 용량이 0으로 설정된 채널로 송신자와 수신자가 만날 때만 원소를 교환한다 (직거래 느낌)
    - Conflated : Channel.CONFLATED 또는 버퍼 크기가 1로 설정된 채널로 새로운 원소가 들어오면 이전 원소를 대체한다
- 채널 설정 방법
    - `val channel = Channel<T>(capacity = Channel.UNLIMITED)`
    - `val channel = produce(apacity = Channel.UNLIMITED) { … }`

</br>

## 버퍼 오버플로인 상황

- 채널을 커스터마이징 하기 위해 버퍼가 꽉 찼을 때의 동작을 정의할 수 있다
- 3가지 옵션이 존재
    - SUSPEND : 버퍼가 가득 찼을 때  send 중단
    - DROP_OLDEST : 버퍼가 가득 찼을 때 가장 오래된 원소가 제거
    - DROP_LATEST : 버퍼가 가득 찼을 때 가장 최근의 원소가 제거
- 버퍼 크기가 1로 설정되고 새로운 원소가 들어오면 이전 원소를 대체하는 Conflated는 DROP_OLDEST가 적용된 것이다
- **Overflow 옵션은 produce 빌더에서는 적용할 수 없다 (실험 API로 제공)**

</br>

## 전달되지 않는 원소 핸들러

- 채널 함수의 마지막 인자인 onUndeliveredElement는 원소가 어떠한 이유로 처리되지 않을 때 호출된다
- 보통 채널이 닫히거나 취소 되었음을 의미하나 에러를 던질 때 발생할 수 도 있다
- 보통 채널을 닫아 리소스 낭비를 줄이고자 할 때 사용된다

> Ex) onUndeliveredElement 활용
> 

```kotlin
val channel = Channel<T>(
	capacity = Channel.RENDEZOUS,
	onBufferOverflow = BufferOverflow.SUSPEND
) { resource ->
	resource.close() // 채널 해제
}
```

</br>

## Fan-out

- 하나의 producer 코루틴으로부터 여러 consumer 코루틴이 데이터를 받는 경우
- 값이 들어올 때마다 여러 소비자 중 하나에게 값을 전달한다 (Load Balancing 과 유사)
- 원소를 잘 처리하려면 for 문을 반드시 활용해야 한다 (consumeEach는 여러 개 코루틴이 사용하기엔 불안전)

> Ex) Fan-out 상황
> 

```kotlin
fun CoroutineScope.produceNumbers() = produce {
    repeat(10) {
        delay(100)
        send(it)
    }
}

fun CoroutineScope.launchProcessor(
    id: Int,
    channel: ReceiveChannel<Int>
) = launch {
		// for문 활용
    for (msg in channel) {
        println("#$id received $msg")
    }
}

suspend fun main(): Unit = coroutineScope {
		// 1개의 producer 코루틴
    val channel = produceNumbers()
    repeat(3) { id ->
	    // 3개의 consumer 코루틴
        delay(10)
        launchProcessor(id, channel)
    }
}
----------------
#0 received 0
#1 received 1
#2 received 2
#0 received 3
#1 received 4
#2 received 5
#0 received 6
#1 received 7
#2 received 8
#0 received 9
```

</br>

## Fan-In

- 여러 개의 producer 코루틴이 보내는 데이터를 하나의 consumer 코루틴이 받는 경우
- 여러 채널을 하나로 합쳐야하는 경우 `fanIn` 함수를 사용할 수 있다

> Ex) Fan-in 상황
> 

```kotlin
suspend fun sendString(
    channel: SendChannel<String>,
    text: String,
    time: Long
) {
    while (true) {
        delay(time)
        channel.send(text)
    }
}

fun main() = runBlocking {
    val channel = Channel<String>()
	  // 2개의 producer 코루틴이 같은 채널을 공유하는 상황
    launch { sendString(channel, "foo", 200L) }
    launch { sendString(channel, "BAR!", 500L) }
    repeat(50) {
        println(channel.receive())
    }
    coroutineContext.cancelChildren()
}
--------------------
foo
foo
BAR!
foo
foo
BAR!
foo
foo
foo
BAR!
foo
foo
BAR!
...
```

</br>

## 파이프라인

- 한 채널로부터 받은 원소를 다른 채널로 전송하는 경우

> Ex) 파이프라인
> 

```kotlin
fun CoroutineScope.numbers(): ReceiveChannel<Int> =
    produce {
        repeat(3) { num ->
            send(num + 1)
        }
    }

fun CoroutineScope.square(numbers: ReceiveChannel<Int>) =
    produce {
		    // numbers에서 값을 send 된 값을 받아와 제곱 전달
        for (num in numbers) {
            send(num * num)
        }
    }

suspend fun main() = coroutineScope {
    val numbers = numbers()
	  // 다른 채널로 값 전송
    val squared = square(numbers)
    for (num in squared) {
        println(num)
    }
}
------
1
4
9

```

</br>

## 채널 실제 활용 예시

- 전형적으로 데이터가 한 쪽에서 생성되고 다른 쪽에서 처리하는 경우 채널을 사용한다
- 사용자 클릭에 반응, 서버로 부터 새로운 알림이 오는 경우, 시간에 따라 검색 결과를 업데이트 하는 상황 etc..

---

## 요약

- 채널은 코루틴(생산/소비)간 통신할 때 사용하는 도구이다
- 송신자와 수신자의 수에 제한이 없으며 채널을 통해 보내진 데이터는 단 1번 받는 것이 보장된다 (값을 사용하면 제거)
- Channel 함수 또는 produce 빌더를 통해 채널을 생성할 수 있다
- 채널의 용량, 버퍼 오버플로 발생 시 동작, 전달되지 않았을 때의 동작을 정의할 수 있다
- **채널은 충돌이 발생하지 않으며 공평함을 보장한다 (Race Condition 발생 X)**
