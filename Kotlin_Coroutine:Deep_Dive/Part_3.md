## 16장 채널

코루틴끼리의 통신을 위한 기본적인 방법으로 `Channel` API가 있다. 많은 사람들은 파이프로 떠올리지만, 이 책에서는 책을 교환하는 공공 책장으로 비유하고 있다.

`Channel`은 서로 다른 2개의 인터페이스를 구현한 하나의 인터페이스이다.

```kotlin
interface SendChannel<in E> {
	suspend fun send(element: E)
	fun close(): Boolean
	// ...
}

interface ReceiveChannel<out E> {
	suspend fun receive(): E
	fun cancel(cause: CancellationException? = null)
	// ...
}

interface Channel<E>: SendChannel<E>, ReceiveChannel<E>
```

- `SendChannel`은 원소를 보내거나(또는 더하거나) 채널을 닫는 용도로 사용된다.
    - `send`는 채널의 용량이 다 찼을 때 중단된다.
- `ReceiveChannel`은 원소를 받을 때(또는 꺼낼 때) 사용된다.
    - `receive`를 호출했을 때 원소가 없다면 코루틴은 원소가 들어올 때까지 중단된다.
- 채널이 닫힐 때까지 원소를 받기 위해 `for` 루프 혹은 `cosumeEach`를 사용한다.
- `produce` 빌더는 코루틴이 어떻게 종료되든 상관 없이(끝나거나, 중단되거나, 취소되거나) 채널을 닫는다.

### 채널 타입

- `Unlimited` : 제한이 없는  용량 버퍼를 가지는 타입. send가 중단되지 않음.
- `Buffered` : 특정 크기 용량 크기(기본값은 64)로 설정되는 타입.
- `Rendezvous`(기본 타입) : 용량이 0인 타입. 송신자와 수신자가 만날때만 원소를 교환.
- `Conflated` : 버퍼 크기가 1인 타입. 새로운 원소가 이전 원소를 대체.

### 버퍼 오버플로일 때

오버플로 관련 옵션은 다음과 같다.

- `SUSPEND`(기본 옵션) : 버퍼가 가득 찼을 때, send 메서드가 중단됨
- `DROP_OLDEST` : 버퍼가 가득 찼을 때, 가장 오래된 원소가 제거
- `DROP_LATEST` : 버퍼가 가득 찼을 때, 가장 최근의 원소가 제거

### 전달되지 않은 원소 핸들러

`onUndeliveredElement` 파라미터를 통해 처리되지 않은 원소를 핸들링 할 수 있다. 주로 채널에서 보낸 자원을 닫을 때 사용한다.

```kotlin
// 이렇게
val channel = Channel<Resource>(capacity) { resource ->
	resource.close()
}

// 또는 이렇게
val channel = Channel<Resource>(
	capacity,
	onUndeliveredElement = { resource ->
		resource.close()
	}
}
```

### 팬아웃(Fan-out)

for 루프를 사용하여 여러개의 코루틴이 하나의 채널로부터 원소를 받을 수 있음. 

```kotlin
fun CoroutineScope.produceNumbers() = produce {
	repeat(10) {
		delat(100)
		send(it)
	}
}

fun CoroutineScope.launchProcessor(
	id: Int, 
	channel: ReceiveChannel<Int>
) = launch {
	for (msg in channel) {
		println("#$id received $msg")
	}
}

suspend fun main(): Unit = coroutineScope {
	val channel = produceNumbers()
	repeat(3) { id ->
		delay(10)
		launchProcessor(id, channel)
	}
}

// #0 received 0
// #1 received 1
// #2 received 2
// #0 received 3
// #1 received 4
// #2 received 5
// #0 received 6
// ...
```

### 팬인(Fan-in)

fanIn 함수를 사용하여 여러개의 코루틴이 하나의 채널로 원소를 전송할 수 있음.

```kotlin
suspend fun sendString(
	channel: SendChannel<String>,
	text: String,
	time: Long
) {
	while (true) {
		delay(time)
		channel.send(text)
	}
}

fun main() = runBlocking {
	val channel = Channel<String>()
	launch { sendString(channel, "foo", 200L) }
	launch { sendString(channel, "Bar!", 500L) }
	repeat(50) {
		println(channel.receive())
	}
	coroutineContext.cancelChildren()
}

// (200 밀리초 후)
// foo
// (200 밀리초 후)
// foo
// (100 밀리초 후)
// Bar!
// (100 밀리초 후)
// foo
// (200 밀리초 후)
// ...

fun <T>CoroutineScope.fanIn(
	channels: List<ReceiveChannel<T>>
): ReceiveChannel<T> = produce {
	for (channel in channels) {
		launch {
			for (elem in channel) {
				send(elem)
			}
		}
	}
}
```

## 17장 셀렉트

여러 개의 소스에 데이터를 요청한 뒤, 가장 빠른 응답만 얻어 사용하는 경우 `select` 함수를 사용한다.

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

// (1초 후)
// Data2
```

`select` 함수에서 `onSend`를 활용하면 버퍼에 공간이 있는 채널을 선택해 데이터를 전송하는 용도로 사용할 수도 있다.

```kotlin
fun main(): Unit = runBlocking {
	val c1 = Channel<Char>(capacity = 2)
	val c2 = Channel<Char>(capacity = 2)
	
	launch {
		for (c in 'A'..'H') {
			delay(400)
			select<Unit> {
				c1.onSend(c) { println("Sent $c to 1") }
				c2.onSend(c) { println("Sent $c to 2") }
			}
		}
	}
	
	launch {
		while (true) {
			delay(1000)
			val c = select<String> {
				c1.onReceive { "$it from 1")
				c2.onReceive { "$it from 2")
			}
			println("Received $c")
		}
	}
}

// Sent A to 1
// Sent B to 1
// Received A from 1
// Sent C to 1
// Sent D to 2
// Received B from 1
// Sent E to 1
// Sent F to 2
// Received C from 1
// Sent G to 1
// Received D from 1
// Sent H to 1
// Received E from 1
// Received F from 1
// Received G from 2
// Received H from 2
```

## 18장 핫 데이터 소스와 콜드 데이터 소스

| 핫 | 콜드 |
| --- | --- |
| 컬렉션(List, Set) | Sequence, Stream |
| Channel | Flow, RxJava 스트림 |
- Hot Stream
    - 데이터의 소비와 무관하게 원소를 생성
- Cold Stream
    - 요청이 있을 때만 작업을 수행

## 19장 플로우란 무엇인가?

플로우(flow)는 비동기적으로 계산해야 할 값의 스트림을 나타낸다. Flow 인터페이스 자체는 떠다니는 원소들을 모으는 역할을 하며, 플로우의 끝에 도달할 때까지 각 값을 처리하는걸 의미한다.

```kotlin
interface Flow<out T> {
	suspend fun collect(collector: FlowCollector<T>)
}
```

### 플로우의 특징

- Flow의 최종 연산은 스레드를 블로킹하는 대신 코루틴을 중단시킴
- 코루틴 컨텍스트를 활용하고 예외를 처리하는 등의 코루틴 기능도 제공
- Flow의 처리는 취소가 가능하며, 구조화된 동시성을 기본적으로 갖추고 있음

### 플로우 명명법

```kotlin
suspend fun main() {
	flow { emit("Message 1") }                        // 플로우 빌더
		.onEach { println(it) }                         // 중간 연산
		.onStart{ println("Do Something before") }      // 중간 연산
		.onCompletion { println("Do something after") } // 중간 연산
		.catch { emit("Error") }                        // 중간 연산
		.collect { println("Collected $it") }           // 최종 연산
```

## 20장 플로우의 실제 구현

Flow의 실제 구현은 아래와 거의 동일하다.

```kotlin
fun interface FlowCollector<T> {
	suspend fun emit(value: T)
}

interface Flow<T> {
	suspend fun collect(collector: FlowCollector<T>)
}

fun <T> flow(
	builder: suspend FlowCollector<T>.() -> Unit
) = object : Flow<T> {
	override suspend fun collect(
		collector: FlowCollector<T>
	) {
		collector.builder()
	}
}

suspend fun main() {
	val f: Flow<String> = flow {
		emit("A")
		emit("B")
		emit("C")
	}
	f.collect { println(it) } // ABC
	f.collect { println(it) } // ABC
}
```

## 21장 플로우 만들기

### 원시값을 가지는 플로우

- `flowOf()`
- `emptyFlow()`

### 플로우 빌더

- `flow { … }`

### 채널플로우(channelFlow)

원소를 처리하고 있을 때 미리 페이지를 받아오고 싶은 경우가 있다. 네트워크 요청이 더 빈번하지만, 결과를 더 빠르게 얻어올 수 있다. 이렇게 하려면 데이터를 생성하고 소비하는 과정이 별개로 진행되어야 한다. 이는 채널과 같은 핫 데이터 스트림의 전형적인 특징이다. 따라서 채널과 플로우를 합친 형태가 필요하다. `channelFlow` 채널과 플로우의 특성을 동시에 가진다

```kotlin
fun allUsersFlow(api: UserApi): Flow<User> = channelFlow {
	var page = 0
	do {
		println("Fetching page $page")
		val users = api.takePage(page++)
		users.forEach { send(it) }
	} while (!users.isNullOrEmpty())
}

suspend fun main() {
	val api = FakeUserApi()
	val users = allUsersFlow(api)
	val user = users
		.first {
			println("Checking $it")
			delay(1000)
			it.name == "User3"
		}
	println(user)
}

// Fetching page 0
// (1초 후)
// Checking User(name=User0)
// Fetching page 1
// (1초 후)
// Checking User(name=User1)
// Fetching page 2
// (1초 후)
// Checking User(name=User2)
// Fetching page 3
// (1초 후)
// Checking User(name=User3)
// Fetching page 4
// (1초 후)
// User(name=User3)
```

### 콜백플로우(callbackFlow)

사용자의 클릭이나 활동 변화를 감지해야 하는 이벤트 플로우가 필요한 경우가 있다. 감지하는 프로세스는 이벤트를 처리하는 프로세스와 독립적이어야 한다. `channelFlow`를 사용해도 되지만, 이럴 때는 `callbackFlow`를 사용하는게 좋다.

`callbackFlow`는 콜백을 래핑하는데 몇가지 유용한 기능들을 제공한다.

- `awaitClose { … }` : 채널이 닫힐 때까지 중단되는 함수
- `trySendBlocking(value)` : send와 중단하는 대신 블로킹하여 중단함수가 아닌 함수에서도 사용할 수 있음
- `close()` : 채널을 닫음
- `cancel(throwable)` : 채널을 종료하고 플로우에 예외를 던진다.

## 22장 플로우 생명주기 함수

## 23장 플로우 처리

## 24장 공유플로우와 상태플로우

## 25장 플로우 테스트하기
