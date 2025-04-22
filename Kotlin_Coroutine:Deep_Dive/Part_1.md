# 1부 코틀린 코루틴 이해하기

## 1장 코틀린 코루틴을 배워야 하는 이유

코틀린 코루틴이 도입한 핵심 기능은 코루틴을 특정 지점에서 멈추고 이후에 재개할 수 있다는 것이다. 스레드가 블락되는 것이 아니라 코루틴이 중단되는 것이기 때문에, 스레드를 블락하지 않는다. 스레드는 명시적으로 생성해야 하고, 유지되어야 하며, 스레드를 위한 메모리 또한 할당되어야 하기 때문에 비용이 크다.

## 2장 시퀀스 빌더

코루틴의 시퀀스는 List나 Set과 같은 컬렉션이랑 비슷한 개념이지만, 필요할 때마다 값을 하나씩 계산하는 지연(lazy) 처리를 한다. 시퀀스의 특징은 다음과 같다.

- 요구되는 연산을 최소한으로 수행
- 무한정이 될 수 있음
- 메모리 사용이 효율적

```kotlin
val seq = sequence {
	println("Generating first")
	yield(1)
	println("Generating second")
	yield(2)
	println("Generating third")
	yield(3)
	println("Done")
}

fun main() {
	for (num in seq) {
		println("The next number is $num")
	}
}

// Generating first
// The next number is 1
// Generating second
// The next number is 2
// Generating third
// The next number is 3
// Done
```

```kotlin
val seq = sequence {
	println("Generating first")
	yield(1)
	println("Generating second")
	yield(2)
	println("Generating third")
	yield(3)
	println("Done")
}

fun main() {
	val iterator = seq.iterator()
	println("Starting")
	val first = iterator.next()
	println("First: $first")
	val second = iterator.next()
	println("Second: $second")
	// ...
}

// Prints:
// Starting
// Generating first
// First: 1
// Generating second
// Second: 2
```

플로우는 시퀀스 빌더와 비슷하지만, 여러 가지 코루틴 기능을 지원한다.

---

### 🔍 `Sequence`란?

`Sequence`는 Kotlin에서 **지연 평가(lazy evaluation)** 를 지원하는 컬렉션 처리 방식

즉, 데이터를 **필요할 때마다 하나씩 처리**해서 전체를 한 번에 연산하지 않음

---

### ✅ 주요 특징

#### 1. **Lazy Evaluation (지연 연산)**

- `filter`, `map` 같은 중간 연산은 **실제 연산을 하지 않음**.
- 최종 연산(`toList()`, `first()`, `any()` 등)이 호출될 때만 연산이 시작됨.
- 각 요소는 **파이프라인을 따라 하나씩 흐름**.

```kotlin
val seq = listOf(1, 2, 3, 4, 5).asSequence()
    .filter { println("filter $it"); it % 2 == 1 }
    .map { println("map $it"); it * 2 }

// 아직 아무것도 출력 안 됨
val result = seq.toList() // 이때 출력 시작

```

---

#### 2. **중간 연산 vs 최종 연산**

- **중간 연산**: `map`, `filter`, `take`, `drop`, `flatMap` 등 → 결과를 바로 계산하지 않음
- **최종 연산**: `toList`, `count`, `first`, `any`, `none`, `find` 등 → 이때 연산 수행

---

#### 3. **효율적인 파이프라인 처리**

- 각 요소가 `filter → map → 최종연산` 순서로 **하나씩** 흐름.
- 불필요한 연산 생략 가능 (→ 성능 향상).

```kotlin
(1..1000).asSequence()
    .filter { it % 2 == 0 }
    .map { it * it }
    .first { it > 1000 } // 필요한 값만 평가하고 바로 종료

```

---

#### 4. **컬렉션과의 차이점**

| 특성 | 컬렉션 (List 등) | Sequence |
| --- | --- | --- |
| 연산 방식 | **Eager** (전체 연산) | **Lazy** (요소별 연산) |
| 성능 | 중간 연산 누적됨 | 중간 연산마다 평가 |
| 예시 | `list.map.filter.toList()` | `list.asSequence().map.filter.toList()` |

---

#### 5. **toList(), toSet() 등을 쓰면 lazy 종료됨**

- `Sequence`는 최종 연산을 만나야 평가가 끝나고, 그때 **List나 Set으로 변환**되면 더는 lazy가 아님.

---

### ✅ 언제 쓰면 좋을까?

- 대량의 데이터를 처리할 때
- 필터/맵 체인이 길어서 중간 결과가 많을 때
- 계산량을 줄이고 싶을 때
- `first`, `find`, `any`처럼 **앞에서 멈출 수 있는 연산**을 할 때

---

### ⚠️ 주의점

- 너무 많은 중간 연산을 사용할 경우, **지연 연산이 쌓여 오히려 성능이 나빠질 수 있음**
- `Sequence`는 내부적으로 **Iterator 기반**이라 **랜덤 접근이 불가** (ex. `get(index)` 안 됨)

---

## 3장 중단은 어떻게 작동할까?

코루틴은 중단되었을 때 Continuation 객체를 반환한다. 이 객체는 게임을 저장하는 것과 비슷하다. Continuation을 이용하면 멈췄던 곳에서 다시 코루틴을 실행할 수 있다.

```kotlin
suspend fun main() {
	println("Before")
	
	suspendCoroutine<Unit> { continuation -> 
		println("Before too")
		continuation.resume(Unit)
	}
	
	println("After")
}

// Before
// Before too
// After
```

delay 함수의 구현

```kotlin
private val executor = 
	Executors.newSingleThreadScheduleExecutor {
		Thread(it, "scheduler").apply { isDeamon = true }
	}
	
suspend fun delay(timeMillis: Long): Unit = 
	suspendCoroutine { cont -> 
		executor.schedule({
			cont.resume(Unit)
		}, timeMillis, TimeUnit.MILLISECONDS)
	}
	
suspend fun main() {
	println("Before")
	
	delay(1000)
	
	println("After")
}

// Before
// (1초 후)
// After
```

> `delay()`의 JVM 버전에서는 과거(혹은 단순화된 예제/교육용 구현)에서 `Executors.newSingleThreadScheduledExecutor()`를 내부에서 직접 사용하는 방식이 사용된 적 있음.
> 
> 
> 하지만 현재의 **표준 kotlinx.coroutines 구현**에서는 `DefaultExecutor`라는 **전역 스레드 기반의 `EventLoop`**를 사용.
> 

---

### 🔍 왜 책에서는 `Executors.newSingleThreadScheduledExecutor()`를 쓴다고 했을까?

#### 가능한 이유:

1. **책이 `delay()`의 기본 원리를 설명하기 위해 단순화한 예시**로 든 것일 수 있음.
    
    > "delay는 내부적으로 예약 작업을 스케줄하기 때문에, 이렇게도 구현할 수 있다"는 의미로.
    > 
2. 예전 버전의 `kotlinx.coroutines` 구현에서는 실제로 다음과 같이 되어 있던 적도 있음:
    
    ```kotlin
    val executor = Executors.newSingleThreadScheduledExecutor()
    executor.schedule({ continuation.resume(Unit) }, delay, TimeUnit.MILLISECONDS)
    
    ```
    
3. 또는 `custom dispatcher`, `custom delay` 구현 예시로 설명했을 수도 있음.

---

### 🧠 현재 공식 구현 확인 (kotlinx.coroutines 1.6.x~1.8 기준)

`delay()`는 다음 흐름으로 실행됨:

```kotlin
suspend fun delay(timeMillis: Long): Unit {
    if (timeMillis <= 0) return
    return suspendCancellableCoroutine { cont ->
        cont.context.delay.scheduleResumeAfterDelay(timeMillis, cont)
    }
}

```

여기서 `cont.context.delay`는 보통 `DefaultDelay`이며,

`DefaultDelay`는 실제로 `DefaultExecutor`라는 전역 스케줄러에서 작업을 예약.

---

### 🔬 `DefaultExecutor`는 어떤 식으로 동작할까?

```kotlin
object DefaultExecutor : EventLoopImplBase(), Runnable {
    init {
        thread(name = "kotlinx.coroutines.DefaultExecutor", daemon = true) { run() }
    }

    override fun scheduleResumeAfterDelay(timeMillis: Long, continuation: CancellableContinuation<Unit>) {
        // 시간 후에 실행할 작업 등록
        schedule(timeMillis) { continuation.resume(Unit) }
    }
}

```

- 내부에서 실제로 스레드를 **단 하나만 생성** (`daemon = true`)
- `Executors.newSingleThreadScheduledExecutor()`를 직접 사용하진 않지만,
구조적으로 **같은 방식으로 동작하는 "자체 구현"된 전역 스케줄러**

---

### ✅ 결론 요약

| 항목 | 내용 |
| --- | --- |
| 책 내용은? | `delay()`가 `Executors.newSingleThreadScheduledExecutor()`를 쓴다고 설명 |
| 실제 구현은? | `DefaultExecutor`라는 **직접 구현된 전역 싱글 스레드 타이머**를 사용 |
| 왜 다른가요? | 개념적으로 동일한 구조이기 때문에, 설명을 단순화한 것 |
| 지금도 Executors 써도 되나요? | ✅ 직접 디스패처나 커스텀 딜레이 구현 시 사용 가능함 |

### ⏰ 내부 메커니즘 (실제 예시 기반)

#### `DefaultExecutor`는 다음과 같은 내부 루프를 가짐:

```kotlin

override fun run() {
    while (true) {
        val now = System.nanoTime()
        val task = queue.poll() // 예약된 작업 중 가장 빠른 거
        if (task != null && task.time <= now) {
            task.run() // 이 안에서 continuation.resume(Unit)
        } else {
            // 아직 시간이 안 됐으면 스레드 sleep (짧게)
            Thread.sleep(1)
        }
    }
}

```

---

## 4장 코루틴의 실제 구현

코루틴의 내부 구현에서 중요한 점

- 중단 함수는 함수가 시작할 때와 중단 함수가 호출되었을 때 상태를 가진다는 점에서 상태 머신(state machine)과 비슷하다.
- 컨티뉴에이션(continuation) 객체는 상태를 나타내는 숫자와 로컬 데이터를 가지고 있다.
- 함수의 컨티뉴에이션 객체가 이 함수를 부르는 다른 함수의 컨티뉴에이션 객체를 장식(decorate)한다. 그 결과, 모든 컨티뉴에이션 객체는 실행을 재개하거나 재개된 함수를 완료할 때 사용되는 콜 스택으로 사용된다.

### ✅ `suspend` 함수와 `resumeWith` 요약

- `suspend` 함수는 **중단 가능성만 제공**하며, 실제로 중단되려면 내부에 **중단점(suspension point)**이 있어야 함.
- 중단점이 존재할 때, **작업이 완료되는 시점에 `Continuation.resumeWith(...)`이 호출되어 코루틴이 재개됨.**

---

### ✅ 중단점(suspension point)의 예

| 방식 | 설명 | 중단점 있음? |
| --- | --- | --- |
| `delay()`, `withContext(...)` | 코루틴 기본 제공 suspend 함수 | ✅ 있음 |
| `suspendCoroutine { ... }` | 중단 직접 구현, 콜백 처리 | ✅ 있음 |
| 무거운 연산 (`for` 루프 등)만 있는 suspend 함수 | 계산만 하고 중단점 없음 | ❌ 없음 → 중단되지 않음 |

---

### ✅ Android에서 `suspend` 함수 내부 동작

| 라이브러리 / API | 중단 방식 | resumeWith 호출 방식 |
| --- | --- | --- |
| **Retrofit** | `suspendCancellableCoroutine` 사용 | 콜백에서 `resume` 또는 `resumeWithException` 호출 |
| **Room** | `suspendCancellableCoroutine` + Executor | DB 작업 완료 시 `resume(result)` 호출 |
| **withContext** | 내부적으로 중단점 제공 | 작업 완료 후 resumeWith 호출 |
| **DataStore** | Flow + suspend 연산자 사용 | emit 시점 등에서 자동 resumeWith |
| **직접 구현 시** | `suspendCoroutine` 또는 `suspendCancellableCoroutine` 사용 | 우리가 직접 `resume` 호출 |

---

### ✅ 핵심 정리

- `resumeWith()`은 **중단된 코루틴을 다시 이어주는 함수**
- 대부분의 Android `suspend` API들은 내부적으로 **직접 `resumeWith()`을 명시적으로 호출**하고 있음
- 그래서 개발자는 `suspend` 함수만 호출하면 중단과 재개가 **자동으로 처리됨**

## 5장 코루틴: 언어 차원에서의 지원 vs 라이브러리

| 언어 차원에서의 지원 | kotlinx.coroutines 라이브러리 |
| --- | --- |
| 컴파일러가 지원하며 코틀린 기본 라이브러리에 포함되어 있다. | 의존성을 별도로 추가해야 한다. |
| kotlin.coroutines 패키지에 포함되어 있다. | kotlinx.coroutine 패키지에 포함되어 있다. |
| Continuation 또는 suspendCoroutines과 같은 몇몇 기본적인 것들과 suspend 키워드를 최소한으로 제공한다. | launch, async, Deferred처럼 다양한 기능을 제공한다. |
| 직접 사용하기 아주 어렵다. | 직접 사용하기 편리하게 설계되어 있다. |
| 거의 모든 동시성 스타일이 허용된다. | 단 하나의 명확한 동시성 스타일을 위해 설계되어 있다. |
