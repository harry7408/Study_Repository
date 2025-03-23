## 시퀀스 빌더

- 파이썬, 자바스크립트에서는 제한된 형태의 코루틴을 사용
    - 비동기 함수 (async/await)
    - 제네레이터 함수 (값을 순차적으로 반환)

**⭐ 코틀린에서는 제네리이터 함수 대신 시퀀스를 생성할 때 사용하는 시퀀스 빌더를 제공**

- 시퀀스 : List, Set과 비슷한 개념이지만 **필요할 때마다 값을 계산하는 지연(lazy) 처리**를 한다.
    - 연산을 최소한으로 수행 (**필요할 때만 수행**)
    - 메모리 사용이 효율적

### sequence

- 시퀀스는 sequence라는 함수를 이용해 정의 (람다 표현식 형태) ⇒ DSL 적용
- sequence 빌더는 인자로 `suspend SequenceScope<T>.() -> Unit` 만을 받아 후행람다 표기가 가능하다
- 람다 내부에는 yield 함수를 호출하여 시퀀스의 값을 생성</br>
**⭐ 시퀀스 빌더는 yield 반환 함수 이외의 중단 함수를 사용하면 안된다**

### EX 1)

```kotlin
val seq= sequence {
	yield(1)
	yield(2)
	yield(3)
}

fun main() {
	for(num in seq) {
			print(num)
		}
}
// 출력 결과
123
```

- 위 코드에서 사용한 sequence 함수는 DSL (Domain-Specific-Language)이다
- 인자는 수신 객체 람다 (suspend SequenceScope<T>.() → Unit)
- SequenceScope 수신 객체는 yield, yieldAll 메서드를 가지고 있다

**⭐ 위 코드에서 숫자가 미리 생성되는 것이 아니라 필요할 때 마다 생성된다는 것을 명심**

- yield 로 값을 반환하는 코드를 넣지 않으면 seq에 sequence 람다 함수로 값을 저장할 수 없다

### EX 2)

```kotlin
val seq= sequence {
    println("Generating first")
    yield(1)
    println("Generating second")
    yield(2)
    println("Generating third")
    yield(3)
}

fun main() {
    for (num in seq) {
        println("next number is $num")
    }
}
// 출력결과
Generating first
next number is 1
Generating second
next number is 2
Generating third
next number is 3
```

- 이전에 다른 숫자를 찾기 위해 **멈췄던 지점에서 코드가 다시 실행**된다 (중단 체제가 있기 때문에 가능한 일)
- yield(1)에 주석 처리를 한다면 next number is 1은 출력 되지 않는다

### EX 3)

```kotlin
val seq= sequence {
    println("Generating first")
    yield(1)
    println("Generating second")
    yield(2)
    println("Generating third")
    yield(3)
    println("Done")
}

fun main() {
   val iterator=seq.iterator()
    println("Starting")
    val first=iterator.next()
    println("First: $first")
    val second=iterator.next()
    println("Second: $second")
    val third=iterator.next()
    println("Third: $third")
}
// 출력결과
Starting
Generating first
First: 1
Generating second
Second: 2
Generating third
Third: 3
```

- 여기선 seq 내부의 Done이 호출되지 않는다

- 위 코드에서 foruth 변수를 하나 더 만들고 iterator.next()를 호출하면 다음과 같은 에러가 발생

```kotlin
Exception in thread "main" java.util.NoSuchElementException
	at kotlin.sequences.SequenceBuilderIterator.nextNotReady(SequenceBuilder.kt:163)
	at kotlin.sequences.SequenceBuilderIterator.next(SequenceBuilder.kt:146)
	at ~.main(rxexample.kt:22)
	at ~.main(rxexample.kt)
```

## 시퀀스 빌더 실제 사용 예시

### 1. 피보나치 수열

```kotlin
val fibonacci: Sequence<BigInteger> = sequence {
    var first=0.toBigInteger()
    var second=1.toBigInteger()
    while (true) {
        yield(first)
        val temp=first
        first+=second
        second=temp
    }
}

fun main() {
    print(fibonacci.take(10).toList())
}
// 출력 결과
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

- main 함수에서 toList( )라는 중단 함수를 호출 해야 연산이 진행된다 ⇒ 필요할 때 연산이 진행되는 특징

### 2. 난수 및 임의의 문자열 생성

```kotlin
// 난수 생성
fun randomNumbers(
    seed: Long = System.currentTimeMillis()
) : Sequence<Int> = sequence { 
    val random= Random(seed)
    while (true) {
        yield(random.nextInt())
    }
}

// 임의의 문자열 생성
fun randomUniqueStrings(
    length : Int,
    seed: Long=System.currentTimeMillis()
) : Sequence<String> = sequence {
    val random=Random(seed)
    val charPool = ('a'..'z') + ('A'..'Z') + ('0'..'9')
    while (true) {
        val randomString=(1..length)
            .map { i -> random.nextInt(charPool.size) }
            .map(charPool::get)
            .joinToString("")
        yield(randomString)
    }
}.distinct()

fun main() {

    val sqi= randomNumbers(0L)
    val sqs= randomUniqueStrings(7,1L)

    println("${sqs.iterator().next()} : ${sqi.iterator().next()}")
}
```

## 정리

- ⭐ Sequence 빌더는 yield(반환)가 아닌 중단 함수를 사용하면 안된다
    - 중단이 필요하다면 Flow를 사용하는 것이 낫다

```kotlin
fun allUsersFlow(
	api:UserApi
	// Flow Builder
): Flow<User> = flow {
		var page=0
		do {
			val users=api.takePage(page++)  // 중단함수
			emitAll(users)
		} while(!users.isNullOrEmpty())
}
```

---

## 📎 읽어보기

- SequenceScope
[SequenceScope - Kotlin Programming Language (kotlinlang.org)](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/-sequence-scope/)

- take 메서드
[take - Kotlin Programming Language (kotlinlang.org)](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/take.html)
