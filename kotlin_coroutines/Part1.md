# 코틀린 코루틴을 배워야 하는 이유

- 기존의 RxJava, Reactor와 같은 라이브러리가 비동기 처리에 사용
- Java 언어 자체로도 MultiThread를 지원하고 있고 콜백 함수를 통해서도 비동기적으로 구현 가능

 

Why? 코루틴을 사용?

```kotlin
MultiPlatform에서 작동시킬 수 있고 약간의 수고로 코루틴이 제공하는 대부분의 기능 사용 가능

-> RxJava나 콜백에서 기대할 수 없는 기능들이 존재

코루틴을 도입한다고 해서 기존 코드를 광범위하게 뜯어 고칠 필요도 없음
```

 

### 안드로이드에서의 코루틴 사용 예시

```kotlin
fun onCreate() {
	val news=getNewsFromApi()
	val sortedNews=news
					.sortedByDescending {it.pulishedAt }
	view.showNews(sortedNews)
}
```

- Android에서는 view를 다루는 Thread가 단 하나 존재 (Main/UI Thread)
    - 이 스레드를 막으면 **ANR 문제가 발생**하여 앱이 죽어버린다

현재 상태에서 getNewsFromApi() 메서드 호출 시 Main Thread를 blocking할 것이다 (네트워크 처리)

다른 스레드에서 작업하더라도 showNews 함수를 호출할 때 정보를 받아오지 못해 크래시가 발생한다.

### Thread 전환 방식

```kotlin
fun onCreate() {
 thread {
	val news=getNewsFromApi()
	val sortedNews=news
					.sortedByDescending {it.pulishedAt }
		// Main 스레드에서 동작하게 하도록 하는 방
	  runOnUiThread { 
			view.showNews(sortedNews)
		}
	}
}
```

위와 같이 thread 전환 방식 { thread, runOnUiThread }를 사용할 순 있지만 문제가 있다

- 스레드 실행 시 **멈출 수 있는 방법이 없어** 메모리 누수 가능성 존재
- 스레드를 많이 생성하면 Cost가 늘어난다 (메모리 사용량 증가)
- 스레드 전환 시 **Context Switching은 오버헤드를 발생시킨다**
- 코드가 쓸데없이 길어져 가독성이 떨어진다

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0849f66f-6511-452a-a7bc-8f320eef93fa/d408a3da-9f2c-4a44-9a7f-0a9730c64361/Untitled.png)

### Ex)

View를 열었다가 재빨리 닫아버린다면 뷰가 열린 동안 데이터를 처리하는 스레드가 다수 생성되며 이 스레드를 제거(중지)하지 않으면 스레드는 계속 동작하게 된다. 이는 불필요한 작업이며 백그라운드에서 예외를 발생시키거나 예상하기 어려운 결과가 발생할 수 있다.

→ 스레드를 많이 생성, 작업을 멈추지 못해 누수 발생

### 콜백 방식

- 함수의 작업이 끝났을 때 호출 될 콜백 함수를 넘겨주는 방식

```kotlin
fun onCreate() {
	getNewsFromApi { news -> 
		val sortedNews= news  // 함수의 작업이 끝나야 다음 함수가 동작한다.
				.sortedByDescending { it.publishedAt }
	view.showNews(sortedNews)
	}
}

//
fun onCreate() {
	starteadCallbacks += getNewsFromApi { news -> 
		val sortedNews = news
					.sortedByDescending { it.publishedAt }
		view.showNews(sortedNews)
	}
}

// 다중 Callback 
// Config와 User 를 가져오는 작업을 동시에 처리할 수 없다
fun showNews() {
	getConfigFromApi { config -> {
		getNewsFromApi(config) { news -> {
			getUserFromApi { user -> 
				view.showNews(user, news)
			}
		}
	}
}
```

- Call-Back 구조로는 여러 가지 작업을 할 때 동시에 처리할 수 없다
- 중간에 작업을 취소해야 한다면 이를 구현하기 정말 어렵다
- 콜백 지옥에 빠지면 코드를 읽기 어려워 진다
- 콜백 사용 시 작업의 순서를 다루기 힘들어진다

## RxJava & Reactive Stream

- 데이터 스트림 내에서 일어나는 **모든 연산을 시작, 처리, 관찰**할 수 있는 확장 라이브러리
    - 스레드 전환과 동시성 처리를 지원한다 (연산 병렬 처리 가능)

```kotlin
fun onCreate() {
		// disposable은 사용자가 스크린을 빠져나갈 경우 취소를 위해 필요
    disposable += getNewsFromApi()
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .map { news->
            news.sortedByDescending { it.publishedAt }
        }
        .subscribe { sortedNews -> 
            view.showNews(sortedNews)
        }
}

// Single Type 또는 Observable으로 wrapping 이 필요하다
fun getNewsFromApi() : Single<List<News>>
```

- RxJava는 구현하기 아주 복잡하다
- 학습 곡선이 매우 크다

### 코틀린 코루틴의 사용

- 코루틴의 핵심은 **특정 지점에서 멈추고 이후에 재개**할 수 있다는 것이다
(코루틴은 중단했다가 다시 실행할 수 있는 컴포넌트라고 할 수 있다)
    - 이때 코루틴을 중단 시키더라도 **스레드는 Blocking 되지 않으며** 다른 작업이 가능하다

```kotlin
fun onCreate() {
		// 코루틴을 실행할 수 있는 Scope 필요
    viewModelScope.launch {
        val news = getNewsFromApi()
        val sortedNews = news
            .sortedByDescending { it.publishedAt }
        view.showNews(sortedNews)
    }
}
```

### 3개의 End Point를 호출해야 할 때 코루틴 사용 예시

```kotlin
fun onCreate() {
    viewModelScope.launch {
        val config=getConfigFromApi()
        val news=getNewsFromApi(confing)
        val user=getUserFromApi()
        view.showNews(user,news)
    }
}
```

- 위 함수는 코루틴을 사용하여 동작이 가능하지만 각 메서드가 1초씩 걸린다고 했을 때 총 3초가 걸리게 된다

```kotlin
fun onCreate() {
    viewModelScope.launch {
        val config = async { getConfigFromApi() }
        val news = async { getNewsFromApi(confing.await()) }
        val user = async { getUserFromApi() }
        view.showNews(user.await(), news.await())
    }
}
```

- 다음과 같이 async/await를 써주면 병렬로 호출이 가능하다 (2초 소요)
async : 요청을 처리하기 위해 만들어진 코루틴을 **즉시 시작**하는 함수 { await과 같은 함수를 호출하여 결과를 기다린다 }

### for 문이나 컬렉션 처리 시 코루틴 사용 예시

1. 모든 페이지를 동시에 받아오는 메서드

```kotlin
fun showAllNews() {
    viewModelScope.launch {
        val allNews = (0 until getNumberOfPages())
            .map { page-> async { getNewsFromApi(page)} }
            // await 호출
            .flatMap { it.await() }
        view.showAllNews(allNews)
    }
}
```

1. 페이지 별 순차적으로 받아오는 메서드

```kotlin
fun showPageFromFirst() {
    viewModelScope.launch {
       for (page in 0 until getNumberOfPages()) {
           val news= getNewsFromApi(page)
           view.showNextPage(news)
       }
    }
}
```

## EX

- 스레드를 내리는 방식

```kotlin
fun main() {
    var count=0
    repeat(10) {
        thread {
            Thread.sleep(1000L)
            println("${Thread.currentThread().name} ${count++}")
        }
    }
}

// 출력결과 -> 실행할 때 마다 스레드가 다르게 나올 것이다
Thread-2 0
Thread-0 7
Thread-9 6
Thread-8 5
Thread-1 4
Thread-5 3
Thread-3 2
Thread-7 0
Thread-6 0
Thread-4 1
```

- 스레드가 print문 실행 마다 하나씩 생성 된다.

- 코루틴을 내리는 방식

```kotlin
fun main() = runBlocking {
    var count=0
    repeat(10) {
        launch {
            delay(1000L)
            println("${Thread.currentThread().name} : ${count++}")
        }
    }
}
// 출력 결과
main : 0
main : 1
main : 2
main : 3
main : 4
main : 5
main : 6
main : 7
main : 8
main : 9
```

- 값도 순차적으로 나오고 main 스레드 하나에서 동작한다

---

## 용어

### RxJava

- RxJava is a Java VM implementation of [Reactive Extensions](http://reactivex.io/): a library for composing asynchronous and event-based programs by using observable sequences.

→ NetFlix 에서 만든 비동기 처리를 위한 라이브러리

---

## 🔗 읽어보기

### Reactive Programming 이란?

[리액티브 프로그래밍(Reactive Programming)에 대해서 (sk.com)](https://devocean.sk.com/blog/techBoardDetail.do?ID=165099&boardType=techBlog)
