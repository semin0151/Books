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
