# Select

- 실험적 API로 실제로 사용되는 사례는 드문 함수
- 가장 먼저 완료되는 코루틴의 결과를 기다리는 함수
- 여러 채널 중 버퍼에 남은 공간이 있는 채널을 확인하여 데이터 전송
- 이용 가능한 원소가 있는 채널로부터 데이터를 받을 수 있는지 여부 확인 가능
- 코루틴 사이에 경합을 일으키거나 여러 데이터 소스로 부터 나오는 결과 값을 합칠 수 있다

</br>

## 지연되는 값 선택하기

- 가장 빠른 응답을 얻고자할 때 요청을 여러 개의 비동기 프로세스로 시작한 후 select 함수를 표현식으로 사용하고 그 리시버로 받는 람다 내부에서 값을 기다리면 된다
    > Ex) select Method
    > 
    ```kotlin
    public suspend inline fun <R> select(crossinline builder: SelectBuilder<R>.() -> Unit): R {
        contract {
            callsInPlace(builder, InvocationKind.EXACTLY_ONCE)
        }
        return suspendCoroutineUninterceptedOrReturn { uCont ->
            val scope = SelectBuilderImpl(uCont)
            try {
                builder(scope)
            } catch (e: Throwable) {
                scope.handleBuilderException(e)
            }
            scope.getResult()
        }
    }
    ```
    
    > Ex) select 활용 예시
    > 
    
    ```kotlin
    suspend fun requestData1(): String {
        delay(100_000)
        return "Data1"
    }
    
    suspend fun requestData2(): String {
        delay(1000)
        return "Data2"
    }
    
    suspend fun askMultipleForData(): String = coroutineScope {
        select<String> {
            async { requestData1() }.onAwait { it }
            async { requestData2() }.onAwait { it }
        }.also { coroutineContext.cancelChildren() }
    }
    
    suspend fun main(): Unit = coroutineScope {
        println(askMultipleForData())
    }
    ------------------
    // 가장 빠르게 응답한 결과가 출력된 것을 볼 수 있다
    Data2
    ```
    
    - select 표현식이 하나의 코루틴 작업이 완료됨과 동시에 끝나게 되어 결괏값을 반환
    - 위의 코드에서 cancelChildren 을 호출하지 않으면 100초 뒤 결과가 반환된다 (명시적인 취소가 필요)
    - 취소 하는 부분이 복잡하여 헬퍼 함수를 별도로 정의하거나 raceOf 함수를 지원하는 라이브러리를 활용할 수 있다

</br>

## 채널에서 값 선택하기

- select 함수는 채널에서도 사용 가능하다
    - onReceive : 채널이 값을 가지고 있을 때 선택된다 (값을 받은 뒤 람다의 인자로 활용하며 onReceive가 선택되었을 때 select는 람다식의 결과를 반환한다)
    - onReceiveCatching : 채널이 값을 가지고 있거나 닫혔을 때 선택된다 (값을 나타내거나 채널이 닫혔다는 걸 알려주는 ChannelResult를 받으며 람다식의 인자로 활용 + 람다식의 결과값를 반환)
    - onSend : 채널의 버퍼에 공간이 있을 때 선택된다 (채널에 값을 보낸뒤 채널의 참조값으로 람다식을 수행 + Unit을 반환)
    
    > Ex) 값 선택 상황
    > 
    
    ```kotlin
    suspend fun CoroutineScope.produceString(s: String, time: Long) =
        produce {
            while (true) {
                delay(time)
                send(s)
            }
        }
    
    fun main() = runBlocking {
        val fooChannel = produceString("foo", 210L)
        val barChannel = produceString("BAR", 500L)
        
        repeat(7) {
            select {
                fooChannel.onReceive {
                    println("From fooChannel: $it")
                }
                barChannel.onReceive {
                    println("From barChannel: $it")
                }
            }
        }
        
        coroutineContext.cancelChildren()
    }
    ----------------------
    From fooChannel: foo
    From fooChannel: foo
    From barChannel: BAR
    From fooChannel: foo
    From fooChannel: foo
    From barChannel: BAR
    From fooChannel: foo
    ```
    

---

## 요약

- **select는 가장 먼저 완료되는 코루틴의 결괏값을 기다릴 때나** 여러 개의 채널 중 전송 또는 수신 가능한 채널을 선택할 때 유용하다
- 실험적인 API로 언제 사용되고 존재 유무만 파악하고 있기
