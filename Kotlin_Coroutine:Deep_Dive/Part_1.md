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

## 4장 코루틴의 실제 구현

코루틴의 내부 구현에서 중요한 점

- 중단 함수는 함수가 시작할 때와 중단 함수가 호출되었을 때 상태를 가진다는 점에서 상태 머신(state machine)과 비슷하다.
- 컨티뉴에이션(continuation) 객체는 상태를 나타내는 숫자와 로컬 데이터를 가지고 있다.
- 함수의 컨티뉴에이션 객체가 이 함수를 부르는 다른 함수의 컨티뉴에이션 객체를 장식(decorate)한다. 그 결과, 모든 컨티뉴에이션 객체는 실행을 재개하거나 재개된 함수를 완료할 때 사용되는 콜 스택으로 사용된다.

## 5장 코루틴: 언어 차원에서의 지원 vs 라이브러리

| 언어 차원에서의 지원 | kotlinx.coroutines 라이브러리 |
| --- | --- |
| 컴파일러가 지원하며 코틀린 기본 라이브러리에 포함되어 있다. | 의존성을 별도로 추가해야 한다. |
| kotlin.coroutines 패키지에 포함되어 있다. | kotlinx.coroutine 패키지에 포함되어 있다. |
| Continuation 또는 suspendCoroutines과 같은 몇몇 기본적인 것들과 suspend 키워드를 최소한으로 제공한다. | launch, async, Deferred처럼 다양한 기능을 제공한다. |
| 직접 사용하기 아주 어렵다. | 직접 사용하기 편리하게 설계되어 있다. |
| 거의 모든 동시성 스타일이 허용된다. | 단 하나의 명확한 동시성 스타일을 위해 설계되어 있다. |
