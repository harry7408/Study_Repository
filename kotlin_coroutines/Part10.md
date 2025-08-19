# 예외 처리
- 예외 처리는 코루틴 작동 원리 중 중요한 기능
- 잡히지 않은 예외가 발생하면 프로그램이 종료되는 것처럼 코루틴도 잡히지 않은 예외가 발생하면 종료
- Coroutine은 자신을 취소하고 부모에게 예외를 전파 및 취소
- 취소된 부모는 모든 자식들을 취소 시키고 예외를 부모에게 전파 (구조화된 동시성에 의함)
- Coroutine Builder는 부모도 종료시키고 취소된 부모는 자식들 모두를 취소시킨다

> Ex) 취소 상황
> 

```kotlin
fun main() : Unit = runBlocking {
	// 예외 발생하는 코루틴의 부모 (예외가 발생했음을 전파 받는다)
	launch {
		// 예외 발생하는 코루틴 
		launch {
			delay(1000)
			throw Error("Error Occurred") // 에외 발생
		}
		
		launch {
			delay(2000)
			println("Not Printed")
		}
		
		launch {
			delay(500)
			println("Printed")
		}
	}
	
	launch {
		delay(2000)
		println("Not Printed")
	}
}
--------------------
Printed
Exception in thread "main" java.lang.Error: Error Occurred ..
```

## 코루틴 종료 멈추기

- 코루틴이 종료 전 예외를 잡을 때 조금이라도 늦으면 손쓸 수 없게 된다
- Coroutine 간의 상호작용은 Job 컨텍스트에 의해 일어난다

```kotlin
fun main() : Unit = runBlocking {
	try {
		launch {
			delay(1000)
			throw Error("Error Occur")
		}
	} catch (e: Throwable) {
		println("Will Not be Printed")
	}
	
	launch {
		delay(2000)
		println("Will Not be Printed")
	}
}
-------------------
Exception in thread "main" ~~
```

## SupervisorJob

- SuperVisorJob을 사용하면 자식에서 발생한 모든 예외를 무시할 수 있다
- 위와 같은 이유로 일반적으로 다수의 코루틴을 시작하는 Scope로 사용된다

```kotlin
fun main(): Unit = runBlocking {
    // SupervisorJob을 사용
    val scope = CoroutineScope(SupervisorJob())
    scope.launch {
        delay(1000)
        throw Error("Some error")
    }
    scope.launch {
        delay(2000)
        println("printed")
    }
    delay(3000)
    println(scope.isActive)
}
-------------------
Exception in thread "main" ~~
printed
true
```

- SupervisorJob을 사용할 땐 위와 같이 Scope의 인자로 SupervisorJob을 사용하는 것이 좋고 각 Builder의 인자에 넣는 것은 좋지 않다
    - 단 하나의 자식만 가지기 때문에 예외처리에 도움되지 않는다 (예외 처리가 부모로 전파되지 않을 가능성이 있다)
    
    ```kotlin
    fun main(): Unit = runBlocking {
        launch(SupervisorJob()) { // runBlocking에 예외가 전파되지 않는다
            launch {
                delay(1000)
                throw Error("Some error")
            }
    
            launch {
                delay(2000)
                println("Will not be printed")
            }
        }
    
        delay(3000)
    }
    ----------------
    Exception in thread "main" ~~~
    ```
    
    > SupervisorJob을 사용하고자 할 때
    > 
    
    ```kotlin
    fun main() : Unit = runBlocking {
    	val job = SupervisorJob()
    	
    	launch(job) {
    		dekay(1000)
    		throw Error("Error1")
    	}
    	
    	launch(job) {
    		delay(2000)
    		println("Will be Printed")
    	}
    	
    	job.join()
    }
    ------------------
    (1초)
    Exception in thread ~
    (1초)
    Will be Printed
    ```
    

## supervisorScope

- Coroutine Builder를 supervisorScope로 래핑하여 예외 전파를 막을 수 있다
- 다른 코루틴에서 발생한 예외를 무시하고 부모와의 연결을 유지한 채 사용 가능하다는 점에서 유용
- supervisorScope는 중단 함수 본체를 래핑할 때 사용된다

 ⭐ 서로 무관한 다수의 작업을 supervisorScope 내에서 실행하여 예외가 발생하더라도 다른 영역은 잘 실행되도록 할 수 있다

```kotlin
fun main(): Unit = runBlocking {
		// supervisorJob으로 감싸진 Scope (중단 함수)
    supervisorScope {
        launch {
            delay(1000)
            throw Error("Some error")
        }
        launch {
            delay(2000)
            println("Will be printed")
        }
        launch {
            delay(2000)
            println("Will be printed")
        }
    }
    println("Done")
}
-------------------
Exception in thread "main" ~~~
Will be printed
Will be printed
Done
```

> supervisorScope 사용 예시
> 

```kotlin
suspend fun notifyInfo(actions: List<AnimalAction>) = 
	supervisorScope {
		actions.forEach { action -> 
			launch {
					notifyInfo(action)
		}
	}
}
```

## coroutineScope 사용

- coroutineScope는 Coroutine Builder와 달리 부모에 영향을 미치는 대신 try~catch 문으로 잡을 수 있는 예외를 던져준다
- 부모 Job을 취소시키는 launch Builder와 달리 coroutineScope는 자식에서 예외 발생시 coroutineScope가 rethrow해서  try~catch로  예외처리를 할 수 있다

```kotlin
runBlocking {
    try {
        coroutineScope {
            launch {
                throw RuntimeException("boom")
            }
        }
    } catch (e: Exception) {
        println("caught $e") // 여기서 잡힘!
    }
}
```

## await

- async Builder의 경우 부모 코루틴을 종료하고 그와 관련된 자식 코루틴을 종료하지만 launch와 달리 Job을 즉시 취소하지 않는다
- await가 호출될 때 예외를 던지게 된다
- SuperVisorJob 또는 SupervisorScope와 async 빌더를 사용할 때 이를 알아두면 좋다

```kotlin
suspend fun main() = supervisorScope {
    val str1 = async<String> {
        delay(1000)
        throw MyException()
    }

    val str2 = async {
        delay(2000)
        "Text2"
    }

    try {
        println(str1.await()) // 예외가 이때 던져진다
    } catch (e: MyException) {
        println(e.cause)
    }

    println(str2.await()) // 같은 Scope에 있더라도 supervisorScope를 사용해서 전파되지 않음
}
------------------
null
Text2
```

## CancellationException

- 만약 예외가 CancellationException의 SubClass라면 부모로 예외가 전파되지 않는다 (현재 Coroutine만 취소시킨다)

<img width="1053" height="162" alt="Image" src="https://github.com/user-attachments/assets/2cdc1073-d6a7-48b1-b039-cbcc42c8a410" />

```kotlin
object MyException : CancellationException()

suspend fun main() : Unit = coroutineScope {
	launch { // 부모 코루틴 (예외로 인한 영향을 받는다)
		launch {
			delay(2000)
			println("Will not be printed")
		}
		throw MyException // 예외 발생 지점
	}
	
	launch { // 동일 Scope 내의 또다른 코루틴 (영향을 받지 않는다)
		delay(2000)
		println("Will be printed")
	}
}
------------------
(2초)
Will be printed
```

## CoroutineException Handler

- CoroutineExceptionHandler Context를 사용하면 예외처리 시 기본으로 다룰 행동을 정의할 때 유용하다
- 이는 예외 전파를 막을 순 없으나 예외 처리 시 할 작업에 대한 내용을 정의할 때 사용 (예외 처리 Callback)
- scope 정의 시 CoroutineContext에 정의한 Handler를 포함해주면 된다

```kotlin
fun main(): Unit = runBlocking {
	// handler 정의
    val handler =
        CoroutineExceptionHandler { ctx, exception ->
            println("Caught $exception")
        }
    val scope = CoroutineScope(SupervisorJob() + handler) // 예외 발생 시 handler 내용이 동작
    scope.launch {
        delay(1000)
        throw Error("Some error")
    }

    scope.launch {
        delay(2000)
        println("Will be printed")
    }

    delay(3000)
}

```

⭐ launch 과 같이 취소가 바로 되는 상황에선 Handler가 동작하나 async builder는 handler가 동작하지 않고 await()가 호출되는 시점에 예외처리를 직접 해줘야 한다
