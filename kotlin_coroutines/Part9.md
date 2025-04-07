# 취소

- 취소는 Coroutine에서 **아주 중요한 기능** 중 하나
- 중단 함수를 사용하는 몇몇 Class와 라이브러리는 **반드시** 취소를 지원

Ref) Coroutine의 취소 기능을 주로 사용하기 위해 안드로이드에서도 Coroutine을 지원하게 되었다

**⭐ Coroutine의 취소 방식은 최고라고 할 수 있다**

### 취소 방법 (안 좋은 예)

- **단순히 Thread를 죽이면 연결을 닫고 자원을 해제하는 기회가 없어** **최악의 취소 방법**이라고 할 수 있다.
- 개발자들이 상태가 Active한지 자주 확인하는 방법은 불편하다

## 기본적인 취소

- Job 인터페이스는 취소하게 하는 **cancel 함수**를 가지고 있다
- cancel 함수는 아래 3가지 효과를 낼 수 있다
    - 호출한 Coroutine은 첫 번째 중단 점에서 Job을 끝낸다
    - Job이 자식을 가지고 있다면 그들 또한 취소된다. 다만 부모는 영향을 받지 않는다
    - Job이 취소되면 취소된 Job은 새로운 Coroutine의 부모로 사용될 수 없다
    (Cancelling 상태가 되었다가 Cancelled 상태로 전환)

```kotlin
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

suspend fun main() : Unit = coroutineScope {
    val job= launch {
        repeat(1_000) { i->
            delay(200)
            println("Printing $i")
        }
    }
    // delay 에 넣는 시간에 따라 출력 결과가 달라 진다.
    delay(1100)
    job.cancel()
    job.join()
    println("Cancelled Successfully")
}

// 출력 결과
Printing 0
Printing 1
Printing 2
Printing 3
Printing 4
Cancelled Successfully
```

- cancel이 호출되어 첫 번째 중단 점인 delay에서 Job을 끝낸다
- 이후 join 함수를 만나 cancel이 종료될 때 까지 기다린다
- 참고로 cancel 함수에 각기 다른 예외를 인자로 넣는 방법을 사용하면 취소된 원인을 명확하게 할 수 있다
    - Coroutine을 취소하기 위해서 사용되는 예외는 반드시 CancellationException의 서브 타입이어야 한다. **(Coroutine을 취소하기 위해서 사용되는 예외는 CancellationException이어야 하기 때문)**
    
- cancel이 호출된 뒤 취소 과정이 완료되는 걸 기다리기 위해 join을 사용하는 것이 일반적
    - join을 사용하지 않으면 race condition가 될 수 있다.

```kotlin
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

suspend fun main() : Unit =coroutineScope {
    val job=launch {
        repeat(1000) { i->
            delay(100)
            // delay 보다 오래 걸리는 작업 이라고 가정
            Thread.sleep(100)
            println("Printing $i")
        }
    }

    delay(1000)
    job.cancel()
//   join 을 넣을 때와 안 넣었을 때 마지막 부분의 출력 결과가 다르다
//    job.join()
    println("Cancelled Successfully")
}

// 출력 결과
Printing 0
Printing 1
Printing 2
Printing 3
Cancelled Successfully
Printing 4
```

- Thread.sleep이 delay 보다는 비교적 오래 걸리는 작업 같다

### CancelAndJoin

- kotlinx.coroutines 라이브러리는 cancel과 join을 함께 호출할 수 있는 간단한 방법으로 **cancelAndJoin이라는 확장 함수를 제공**

```kotlin
public suspend fun Job.cancelAndJoin() {
    cancel()
    return join()
}
```

- Job( ) Factory 함수로 생성된 잡은 cancelAndJoin 호출 방법으로 취소될 수 있다
    - **이는 Job에 딸린 수 많은 Coroutine을 한번에 취소할 때 자주 사용**

```kotlin
import kotlinx.coroutines.Job
import kotlinx.coroutines.coroutineScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

suspend fun main() : Unit = coroutineScope {
    val job= Job()
    launch(job) {
        repeat(1_000) { i->
            delay(200)
            println("Printing $i")
        }
    }

    delay(1100)
    job.cancelAndJoin()
    println("Cancelled Successfully")
}

// 출력 결과
Printing 0
Printing 1
Printing 2
Printing 3
Printing 4
Cancelled Successfully
```

- 한꺼번에 취소하는 기능은 매우 유용하다
- Platform 종류를 불문하고 동시에 수행되는 작업 그룹 전체를 취소해야 할 때가 있다
    - 안드로이드에서 사용자가 View 창을 나갔을 때 View에서 시작된 모든 Coroutine을 취소하는 경우
    

```kotlin
class ProfileViewModel : ViewModel() {
		private val scope = 
				CoroutineScope(Dispatchers.Main + SupervisorJob())

		fun onCreate() {
				scope.launch { loadUserData() }

	override fun onCleared() {
			scope.coroutineContext.cancelChildren()
	}
  ...
}
```
