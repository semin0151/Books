## 6장 코루틴 빌더

### launch 빌더

`launch`가 작동하는 방식은 `thread` 함수를 호출하여 새로운 스레드를 시작하는 것과 비슷하다. 코루틴을 시작하면 불꽃놀이를 할 때 불꽃이 하늘 위로 각자 퍼지는 것처럼 별개로 실행된다. 작동하는 방식은 데몬 스레드와 어느정도 비슷하지만 훨씬 가볍다.

### runBlocking 빌더

`runBlocking`은 스레드를 블로킹 한다. `runBlocking` 내부에서 `delay(1000L)`을 호출하면`Thread.sleep(1000L)`과 비슷하게 동작한다. 일반적인 경우에서는 사용하지 않으며 특수한 경우로는 아래 2가지가 있다.

- 프로그램이 끝나는 걸 방지하기 위해 스레드를 블로킹
- 유닛 테스트를 위해 스레드를 블로킹
    - 유닛 테스트에서는 `runTest`를 주로 사용한다.

### async 빌더

`async`는 `launch`와 비슷하지만 값을 생성하도록 설계되어 있다. `Deferred<T>` 타입의 객체를 리턴하며, `T`는 생성되는 값의 타입이다. `Deferred`에는 작업이 끝나면 값을 반환하는 중단 메서드인 `await`이 있다.

```kotlin
fun main() = runBlocking {
	val resultDeferred: Deferred<Int> = GlobalScope.async {
		delay(1000L)
		42
	}
	// do other task....
	val result: Int = resultDeferred.await() // (1초 후)
	println(result)
	// You can also write it simply as follows:
	println(resultDeferred.await())
```

`launch`와 비슷하게 호출되자마자 코루틴을 즉시 시작한다. `Deferred`는 값이 생성되면 값을 내부에 저장하기 때문에 `await`에서 값이 반환되는 즉시 사용할 수 있다. 하지만 값이 생성되기 전에 `await`을 호출하면 값이 나올 때 까지 기다리게 된다.

```kotlin
fun main() = runBlocking {
	val res1 = GlobalScope.async {
		delay(1000L)
		"Text 1"
	}
	val res2 = GlobalScope.async {
		delay(3000L)
		"Text 2"
	}
	val res3 = GlobalScope.async {
		delay(2000L)
		"Text 3"
	}
	println(res1.await())
	println(res2.await())
	println(res3.await())
}
// (1초 후)
// Text 1
// (2초 후)
// Text 2
// Text 3
```

주로 두 가지 다른 곳에서 데이터를 얻어와 합치는 경우처럼, 두 작업을 병렬로 실행할 때 주로 사용된다.

```kotlin
scope.launch {
	val news = async {
		newsRepo.getNews()
			.sortedByDescending { it.date }
	}
	val newsSummary = newsRepo.getNewsSummary()
	view.showNews(
		newsSummary,
		news.await()
	)
}
```

### 구조화된 동시성

`runBlocking`, `launch`, `async` 빌더 함수를 보면 아래와 같이 구현되어 있다.

```kotlin
fun <T> runBlocking(
	context: CoroutineContext = EmptyCoroutineContext,
	block: suspend CoroutineScope.() -> T
): T

fun CoroutineScope.launch(
	context: CoroutineContext = EmptyCoroutineContext,
	start: CoroutineStart = CoroutineStart.DEFAULT,
	block: suspend CoroutineScope.() -> T
): Job

fun CoroutineScope.async(
	context: CoroutineContext = EmptyCoroutineContext,
	start: CoroutineStart = CoroutineStart.DEFAULT,
	block: suspend CoroutineScope.() -> T
): Deferred<T>
```

`launch`, `async`의 `block`은 리시버 타입이 `CoroutineScope`인 함수형 타입이다. 또한 `launch`, `async`는 `CoroutineScope`의 확장함수이다. 부모 스코프에서 제공되는 리시버를 통해 호출될 수 있음을 의미한다. 부모는 자식들을 위한 스코프를 제공하고, 자식들을 해당 스코프 내에서 호출한다. 이를 통해 **구조화된 동시성**이라는 관계가 성립한다.

부모-자식 관계의 가장 중요한 특징은 다음과 같다.

- 자식은 부모로부터 컨텍스트를 상속받는다.(7장 ‘코루틴 컨텍스트’)
- 부모는 모든 자식이 작업을 마칠 때까지 기다린다.(8장 ‘잡과 자식 코루틴 기다리기’)
- 부모 코루틴이 취소되면 자식 코루틴도 취소된다.(9장 ‘취소’)
- 자식 코루틴에서 에러가 발생하면, 부모 코루틴 또한 에러로 소멸한다.(10장 ‘예외 처리’)

## 7장 코루틴 컨텍스트

`CoroutineScope`의 정의

```kotlin
public interface CoroutineScope {
	public val coroutineContext: CoroutineContext
}
```

`Continuation`의 정의 

```kotlin
public interface Continuation<in T> {
	public val context: CoroutineContext
	public fun resumeWith(result: Result<T>)
}
```

### CoroutineContext 인터페이스

`CoroutineContext`는 원소나 원소들의 집합을 나타내는 인터페이스이다. 내부적으로 `Map`처럼 작동하는 immutable한 key-value 구조이며, 그 구현체들이 `Element` 단위로 연결된 `LinkedList` 구조로 관리된다. 특이한 점은 `Element` 또한 `CoroutineContext`이다.

**주요 구현체**

- `Job`
- `CoroutineDispatcher`
- `CoroutineName`
- `CoroutineExceptionHandler`

### CoroutineContext에서 원소 찾기

```kotlin
fun main() {
	val ctx: CoroutineContext = CoroutineName("A name")
	
	val coroutineName: CoroutineName? = ctx[CoroutineName]
	// or [ctx.get(CoroutineName)]
	println(coroutineName?.name) // A name
	val job: Job? = ctx[Job]     // or ctx.get(Job)
	println(job)                 // null
```

여기서 get의 인자로 들어가는 타입은 `CoroutineName`, `Job`의 companion object이다. 

```kotlin
data class CoroutineName(
	val name: String
) : AbstractCoroutineContextElement(CoroutineName) {

	override fun toString(): String = "CoroutineName($name)"
	
	companion object Key : CoroutineContext.Key<CoroutineName>
}
```

코루틴 컨텍스트를 계산하는 간단한 공식은 다음과 같다.

> defaultContext + parentContext + childContext
> 

## 8장 잡과 자식 코루틴 기다리기

### Job이란 무엇인가?

잡(job)은 수명을 가지고 있으며 취소 가능하다. `Job`은 인터페이스이긴 하지만 구체적인 사용법과 상태를 가지고 있다는 점에서 추상 클래스처럼 다룰 수도 있다.

잡의 수명은 상태로 나타낸다.

- `NEW` : 선택 가능한 시작 상태
- `ACTIVE` : 기본적인 시작 상태
- `COMPLETING` : 완료 되었을 때
- `CANCELLING` : 취소 되었을 때
- `COMPLETED` : 완료 최종 상태
- `CANCELLED` : 취소 최종 상태

```kotlin
suspend fun main() = coroutineScope {
	val job = Job()
	println(job) // JobImpl{Active}@ADD
	job.complete()
	println(job) // JobImpl{Completed}@ADD
	
	val activeJob = launch {
		delay(1000)
	}
	println(activeJob) // StandaloneCoroutine{Active}@ADD
	activeJob.join()   // (1초 후)
	println(activeJob) // StandaloneCoroutine{Completed}@ADD
	
	val lazyJob = launch(start = CoroutineStart.LAZY) {
		delay(1000)
	}
	println(lazyJob)   // LazyStandaloneCoroutine{New}@ADD
	lazyJob.start()
	println(lazyJob)   // LazyStandaloneCoroutine{Active}@ADD
	lazyJob.join()     // (1초 후)
	println(lazyJob)   // LazyStandaloneCoroutine{Completed}@ADD
}
```

`Job`은 코루틴이 상속하지 않는 유일한 코루틴 컨텍스트이며, 이는 코루틴에서 아주 중요한 법칙이다. 모든 코루틴은 자신만의 `Job`을 생성하며 인자 또는 부모 코루틴으로부터 온 잡은 새로운 잡의 부모로 사용된다. 부모 잡은 자식 잡 모두를 참조할 수 있으며, 자식 또한 부모를 참조할 수 있다. 잡을 참조할 수 있는 부모-자식 관계가 있기 때문에 코루틴 스코프 내에서 취소와 예외 처리 구현이 가능하다.

### Job 클래스 계층 구조

```kotlin
interface Job : CoroutineContext.Element
interface CompletableJob : Job

// 실제 구현 클래스
open class JobSupport(...) : CompletableJob, CoroutineScope
class JobImpl(...) : JobSupport
class SupervisorJobImpl(...) : JobSupport

```

- `Job`: 코루틴의 생명주기 관리 인터페이스 (취소, 완료 등)
- `CompletableJob`: `complete()` 같은 수동 완료 기능 추가
- `JobSupport`: 상태(state), 핸들러(NodeList), 부모-자식 관계 모두 구현
- `JobImpl`, `SupervisorJobImpl`: 실제 코루틴에서 사용되는 구현체

### 자식들 기다리기

잡의 첫 번째 중요한 이점은 코루틴이 완료될 때까지 기다리는 데 사용될 수 있다는 점이다. join 메서드를 사용하면 지정한 `Job`이 **Completed**나 **Cancelled**와 같은 마지막 상태에 도달할 때까지 기다릴 수 있다.

```kotlin
fun main(): Unit = runBlocking{
    launch {
        delay(1000)
        println("Test1")
    }
    
    launch {
        delay(2000)
        println("Test2")
    }
    
    val children = coroutineContext[Job]?.children
    val childrenNum = children?.count()
    println("Number of children: $childrenNum")
    children?.forEach { it.join() }
    println("All tests are done")
}
// Number of children: 2
// (1초 후)
// Test1
// (1초 후)
// Test2
// All tests are done
```

### 부모-자식 관계 구조화

- `Job(child)`를 생성 시, 부모 Job에 자식으로 **등록**됨
- 구조화된 동시성 기반: 부모가 취소되면 자식도 자동 취소됨

```kotlin
val parent = Job()
val child = Job(parent)
```

- `attachChild()` → `ChildHandleNode(child)`를 부모에 등록
- 자식 Job은 `parentHandle`을 통해 부모 참조

---

### 부모-자식 연결 구조 = LinkedList

- `JobSupport.state`가 `Incomplete`일 때 `NodeList`라는 **lock-free linked list** 사용
- 자식은 `ChildHandleNode` 형태로 **부모 Job의 NodeList에 추가됨**
- `NodeList`에는 자식뿐만 아니라 일반 핸들러도 함께 들어감

```
JobSupport(state=Incomplete)
 └── NodeList
      ├── ChildHandleNode(child1: Job)
      ├── ChildHandleNode(child2: Job)
      └── CompletionHandlerNode(..
```

- 논리적으로는 트리(Tree), 자료구조적으로는 **Linked List + 참조 기반 구조**

### 잡 팩토리 함수

`Job()`은 팩토리 함수의 좋은 예이다. `Job`은 인터페이스이므로 생성자를 가질수 없다. `Job`은 생성자처럼 보이는 간단한 함수로, 가짜 생성자이다. 

```kotlin
public fun Job(parent: Job? = null): CompletableJob

interface CompletableJob {
	fun complete(): Boolean
	fun completeExceptionally(exception: Throwable): Boolean
}
```

`CompletableJob`의 메서드

- `complete(): Boolean` - 잡을 완료하는 데 사용. 모든 자식 코루틴은 작업이 완료될 때까지 실행 상태를 유지하지만, `complete()`를 호출한 잡에서 새로운 코루틴이 시작될 수는 없다. 이미 완료되었으면 true, 아니면 false를 반환한다.
- `completeExceptionally(exception: Throwable): Boolean` - 인자로 받은 예외로 잡을 완료시킨다. 모든 자식 코루틴은 주어진 예외를 래핑한 `CancellationException`으로 즉시 취소된다. 이미 완료되었으면 true, 아니면 false를 반환한다.

### Job vs SupervisorJob 내부 구현 차이 (핵심: `childCancelled`)

`JobSupport` 내부의 다음 함수가 **핵심 차이**를 만듦:

```kotlin
// Job
override fun childCancelled(cause: Throwable): Boolean {
    if (cause is CancellationException) return true
    return cancelImpl(cause) && handlesException
}

// SupervisorJob
override fun childCancelled(cause: Throwable): Boolean {
    return false
}
```

- `Job`은 자식이 예외로 종료되면 `parent.cancel()` 실행됨 (전파됨)
- `SupervisorJob`은 자식이 예외로 종료돼도 무시하고 **다른 자식에 영향 없음**

즉:

| 항목 | Job | SupervisorJob |
| --- | --- | --- |
| 자식 취소 전파 | 부모까지 전파 (`cancelImpl`) | 부모는 영향 없음 (`no-op`) |
| `childCancelled` | 부모 cancel 트리거 | 아무 동작 없음 |

## 9장 취소

### 기본적인 취소

Job 인터페이스는 취소하게 하는 `cancel()` 메서드를 가지고 있다. `cancel()` 메서드를 호출하면 다음과 같은 효과를 가져올 수 있다.

- 호출한 코루틴은 첫 번째 중단점에서 잡을 끝낸다.
- 잡이 자식을 가지고 있다면, 그들 또한 취소된다. 하지만 부모는 영향을 받지 않는다.
- 잡이 취소되면, 취소된 잡은 새로운 코루틴의 부모로 사용될 수 없다.(최종 상태이기 때문)

```kotlin
suspend fun main(): Unit = coroutineScope { 
	val job = launch {
        repeat(1_000) { i ->
            delay(100)
            Thread.sleep(100)
           	println("Printing $i")
        }
    }
    
    delay(1000)
    job.cancel()
    job.join()
    println("Cancelled successfully")
}

// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Printing 4
// Cancelled successfully
```

cancel이 호출된 뒤 다음 작업을 진행하기 전에 취소 과정이 완료되는 걸 기다리기 위해 join을 사용하는 것이 일반적이다. join을 호출하지 않으면 경쟁 상태(race condition)가 될 수도 있다.

```kotlin
suspend fun main(): Unit = coroutineScope { 
	val job = launch {
        repeat(1_000) { i ->
            delay(100)
            Thread.sleep(100)
           	println("Printing $i")
        }
    }
    
    delay(1000)
    job.cancel()
    // job.join()
    println("Cancelled successfully")
}

// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Cancelled successfully
// Printing 4
```

### 취소는 어떻게 동작하는가?

잡이 취소되면 ‘**Cancelling**’ 상태로 바뀐다. 상태가 바뀐 뒤 첫 번째 중단점에서 `CancellationException` 예외를 던진다.

### 취소 중 코루틴을 한번 더 호출하기

- `withContext()`
    - 코드 블록의 컨텍스트를 변경, `withContext` 내부는 취소될 수 없는 `Job`인 `NonCancellable` 객체를 사용
- `job.invokeOnCompletion`
    - `Job`이 최종 상태에 도달했을 때 호출될 핸들러

### 중단될 수 없는 걸 중단하기

- `yield()`
    - 코루틴을 중단하고 즉시 재실행하는 함수
- `job.isActive`
- `job.ensureActive`

### suspendCancellableCoroutine

`suspendCoroutine`과 비슷하지만, 컨티뉴에이션 객체를 몇 가지 메서드가 추가된 `CancellableContinuation<T>`로 래핑한다.

```kotlin
suspend fun someTask() = suspendCancellableCoroutine { cont ->
	cont.invokeOnCancellation {
		// 정리 작업을 수행
	}
	// 나머지 구현 부분
}	
```

## 10장 예외 처리

예외는 자식에서 부모로 전파되며, 부모가 취소되면 자식도 취소되기 때문에 쌍방으로 전파된다. 예외 전파가 정지되지 않으면 계통 구조상 모든 코루틴이 취소되게 된다.

### 코루틴 종료 멈추기

**SupervisorJob**

- 자식에서 발생한 모든 예외를 무시할 수 있다.

**SupervisorScope**

- 코루틴 빌더를 `supervisorScope`로 래핑
- 중단 함수이며, 중단 함수 본체를 래핑하는 역할
- 일반적으로 서로 무관한 다수 작업을 스코프 내에서 실행

### CancellationException은 부모까지 전파되지 않는다

부모 까지 전파되지 않기 때문에 “Will be printed”가 호출된다.

```kotlin
object MyNonPropagatingException: CaancellationException()

suspend fun main(): Unit = coroutineScope {
	launch {
		launch {
			delay(2000)
			println("Will not be printed")
		}
		throw MyNonPropagatingException
	}
	launch {
		delay(2000)
		println("Will be printed")
	}
}

// (2초 후)
// Will be printed
```

### 코루틴 예외 핸들러

예외 전파를 중단시키지는 않지만 예외가 발생했을 때 해야할 것들(기본적으로 예외 스택 트레이스를 출력)을 정의하는데 `CoroutineExceptionHandler`를 사용하면 편리하다.

```kotlin
fun main(): Unit = runBlocking {
	val handler = CoroutineExceptionHandler { ctx, exception ->
		println("Caught $exception")
	}
	
	val scope = CoroutineScope(SupervisorJob() + handler)
	scope.launch {
		delay(1000)
		throw Error("Some Error")
	}
	
	scope.launch {
		delay(2000)
		println("Will be printed")
	}
	
	delay(3000)
}

// Caught java.lang.Error: Some error
// Will be printed
```

## 11장 코루틴 스코프 함수

## 12장 디스패처

## 13장 코루틴 스코프 만들기

## 14장 공유 상태로 인한 문제

## 15장 코틀린 코루틴 테스트하기
