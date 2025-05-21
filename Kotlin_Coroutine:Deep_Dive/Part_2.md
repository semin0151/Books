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

### coroutineScope

`coroutineScope`는 스코프를 시작하는 중단함수이며, 인자로 들어온 함수가 생성한 값을 반환한다.

```kotlin
fun main() = runBlocking {
	val a = coroutineScope {
		delay(1000)
		10
	}
	println("a is calculated")
	val b = coroutineScope = {
		delay(1000)
		20
	}
	println(a)
	println(b)
}

// (1초 후)
// a is calculated
// (1초 후)
// 10
// 20
```

`coroutineScope`는 아래와 같은 특성을 가진다. 중단 함수에서 병렬로 작업을 수행할 경우 사용하는 것이 좋다.

- 부모로부터 컨텍스트를 상속받는다.
- 자신의 작업을 끝내기 전까지 모든 자식을 기다린다.
- 부모가 취소되면 자식들 모두를 취소한다.

### 코루틴 스코프 함수

| 코루틴 빌더
(runBlocking 제외) | 코루틴 스코프 함수 |
| --- | --- |
| launch, async, produce | coroutineScope, 
supervisorScope,
withContext, withTimeout |
| coroutineScope의 확장 함수 | 중단 함수 |
| coroutineScope 리시버의 코루틴 컨텍스트를 사용 | 중단 함수의 컨티뉴에이션 객체가 가진 코루틴 컨텍스트를 사용 |
| 예외는 Job을 통해 부모로 전파됨 | 일반 함수와 같은 방식으로 예외를 던짐 |
| 비동기인 코루틴을 시작함 | 코루틴 빌더가 호출된 곳에서 코루틴을 시작함 |

### withContext

`withContext` 함수는 `coroutineScope`와 비슷하지만 스코프의 컨텍스트를 변경할 수 있다는 점에서 다르다. `withContext`의 인자로 컨텍스트를 제공하면 부모 스코프의 컨텍스트를 대체한다. `withContext(EmptyCoroutineContext)`와 `coroutineScope()`의 동작은 거의 일치한다.

### supervisorScope

`coroutineScope`에서 컨텍스트의 `Job`을 `SupervisorJob`으로 오버라이딩 한 것. 자식 코루틴이 예외를 던져도 취소되지 않는다.

### withTimeout

타임아웃 값을 인자로 넘겨주면 람다식을 취소하고 `TimeoutCancellationException`을 던진다.

## 12장 디스패처

디스패처는 코루틴이 실행되어야 할 스레드(또는 스레드 풀)를 결정한다.

### 기본 디스패처

`Dispatchers.Default`

- CPU 집약적인 연산을 수행하도록 설계
- 컴퓨터 CPU와 동일한 개수의 스레드풀을 가짐
- 프로세스가 가지고 있는 코어 수의 스레드

### 메인 디스패처

`Dispatchers.Main`

- 안드로이드에서의 UI스레드
- 유일한 스레드

`Dispatchers.Main.immediate`

- 코루틴을 배정하는 것 자체가 큐에 들어갔다 나오기 때문에 약간의 비용이 발생하게 된다.
- 해당 디스패처는 현재 스레드가 메인 스레드일 경우 즉시 실행된다.

### I/O 디스패처

`Dispatchers.IO`

- I/O 연산으로 스레드 블로킹할 때 사용하도록 설계
- 64개 혹은 더 많은 코어가 있다면 해당 코어 수의 스레드
- **Java/Android 생태계의 "블로킹 I/O API"들과의 호환성을 위해 스레드를 배정받아 blocking-safe하게 실행**
- 흔히 알고 있는 경량 스레드에 해당되지 않음

### 커스텀 스레드 풀

`limitedParallelism` 함수를 사용하면 독립적인 스레드 풀을 가진 새로운 디스패처를 만든다. 인자로 스레드 개수를 지정할 수 있다.

```kotlin
suspend fun main(): Unit = coroutineScope {
	launch {
		printCoroutinesTime(Dispatchers.IO)
		// Dispatchers.IO took: 2074
	}
	
	launch {
		val dispatcher = Dispatchers.IO
			.limitedParallelism(100)
		printCoroutinesTime(dispatcher)
		// LimitedDispatcher@XXX took: 1082
	}
}
```

### 싱글스레드로 제한된 디스패처

커스텀 스레드 풀을 제한하여 싱글 스레드에서 동작되는 디스패처를 만들 수 있다.

```kotlin
var i = 0

suspend fun main(): Unit = coroutineScope {
	repeat(10_000) {
		launch(Dispatchers.IO) {
			i++
		}
	}
	delay(1000)
	println(i) // ~9930
}

var i = 0

suspend fun main(): Unit = coroutineScope {
	val dispatcher = Dispatchers.Default
		.limitedParallelism(1)
	repeat(10_000) {
		launch(dispatcher) {
			i++
		}
	}
	delay(1000)
	println(i) // 10000
}
```

### 컨티뉴에이션 인터셉터

디스패칭을 관할하는 `CoroutineContext.Element`. 중단되었을 때 `interceptContinuation()` 메서드로 컨티뉴에이션 객체를 수정하고 포장한다. `releaseInterceptedContinuation()` 메서드는 컨티뉴에이션이 종료되었을 때 호출된다.

## 13장 코루틴 스코프 만들기

### CoroutineScope 팩토리 함수

코루틴 스코프 객체를 만드는 가장 쉬운 방법은 `CoroutineScope` 팩토리 함수를 사용하는 것이다. 이 함수는 컨텍스트를 넘겨받아 스코프를 만든다.(잡이 컨텍스트에 없으면 구조화된 동시성을 위해 Job을 추가할 수도 있다.)

```kotlin
public fun CoroutineScope(
	context: CoroutineContext
): CoroutineScope = 
	ContextScope(
		if (context[Job] != null) context
		else context + Job()
	)

internal class ContextScope(
	context: CoroutineContext
): CoroutineScope {
	override val coroutineContext: CoroutineContext = context
	override fun toString(): String =
		"CoroutineScope(coroutineContext=$coroutineContext)"
```

### 안드로이드에서의 스코프

- `viewModelScope`, `lifecycleScope`
    - 뷰모델 또는 라이프사이클이 종료되었을 때 `Job을` 취소 시킴
    - `Dispatchers.Main`과 `SupervisorJob`을 사용

## 14장 공유 상태로 인한 문제

```kotlin
var counter = 0

fun main() = runBlocking {
	massiveRun {
		counter++
	}
	println(counter) // ~567231
}

suspend fun massiveRun(action: suspend () -> Unit) = 
	withContext(Dispatchers.Default) {
		repeat(1000) {
			launch {
				repeat(1000) { action() }
			}
		}
	}
```

위 코드는 동기화가 보장되지 않아 예상되는 값인 1,000,000보다 더 작은 값이 나온다.

### 동기화 방법

**synchronized**

- 전통적인 방법
- suspend 함수를 사용할 수 없음
- 스레드를 블락 함

**Atomic**

- 전체 연산에서 원자성이 보장되지 않음
- 제공되는 함수를 활용
- 객체의 구조가 복잡한 경우에는 사용 불가(primitive 변수 또는 하나의 레퍼런스만 사용)

**싱글 스레드로 제한된 디스패처**

- **코스 그레인드 스레드 한정(coarse-grained thread confinement)**
    - 함수 전체를 싱글 스레드 디스패처로 래핑
    - 여러 스레드에서 병렬로 시작된 작업에 대해 디스패처 제한을  거는 경우 함수 실행이 느려짐
- **파인 그레인드 스레드 한정(fine-grained thread confinement)**
    - 상태를 변경하는 구문들만 싱글 스레드 디스패처로 래핑
    - 번거롭지만 크리티컬 섹션이 아닌 부분이 블로킹 되거나 CPU 집약적인 경우에 더 나은 성능을 제공
    - 일반적인 중단 함수는 큰 성능 차이 없음

**뮤텍스**

- `kotlinx.coroutines`에서 제공하는 `interface`
- OS에서의 뮤텍스와 동일한 개념
- 스레드 대신 코루틴을 블락함
- 뮤텍스 내부에서 뮤텍스를 사용할 경우 교착 상태에 빠지게 됨
    - 따라서 뮤텍스를 활용할 때는 코스 그레인드 방식을 지양해야 함

**세마포어**

- `kotlinx.coroutines`에서 제공하는 `interface`
- OS에서의 세마포어와 동일한 개념
- 동시 요청의 처리 수를 제한할 때 활용

## 15장 코틀린 코루틴 테스트하기


