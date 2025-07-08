## 26장 일반적인 사용 예제

### 데이터/어댑터 계층

라이브러리에서 코루틴 관련 기능을 제공하는 경우 아래 2가지를 활용

- 중단 함수 사용
    - Retrofit
- Flow 사용
    - Room, DataStore

라이브러리에서 코루틴 관련 기능을 제공하지 않는 경우는 `suspendCancellableCoroutine`을 사용해 콜백 함수를 중단 함수로 변환

```kotlin
suspend fun requestNews(): News {
	return suspendCancellableCoroutine<News> { cont ->
		val call = requestNewsApi(
			onSuccess = { news -> cont.resume(news) },
			onError = { e -> cont.resumeWithException(e) },
		)
		cont.invokeOnCancellation {
			call.cancel()
		}
	}
}
```

### 표현/API/UI 계층

플로우를 사용할 때는 `onEach`에서 변경을 처리하고, `launchIn`에서 또 다른 코루틴에서 플로우를 시작하며, `onStart`로 플로우가 시작될 때 특정 행동을 지시하고, `onCompletion`으로 플로우가 완료되었을 때 행동을 정의하며, `catch`로 예외를 처리한다.

```kotlin
fun updateNews() {
	newsFlow()
		.onStart { showProgress() }
		.onCompletion { hideProgress() }
		.onEach { showNews() }
		.catch { handleError() }
		.launchIn(viewModelScope)
}
```

## 27장 코루틴 활용 비법

## 28장 다른 언어에서의 코루틴 사용법

## 29장 코루틴을 시작하는 것과 중단 함수 중 어떤것이 나을까?

코루틴을 시작하면 구조화된 동시성이 지켜지지 않는다. 중단 함수를 사용하면 구조화된 동시성이 지켜진다. 실행되는 중단함수와 유기적으로 엮여있는지, 별도로 동작해야 하는지를 확인해서 사용해야 한다.

## 30장 모범 사례

### async 코루틴 빌더 뒤에 await를 호출하지 마세요

### withContext(EmptyCoroutineContext) 대신 coroutineScope를 사용하세요

### awaitAll을 사용하세요

### 중단 함수는 어떤 스레드에서 호출되어도 안전해야 합니다

### Dispatchers.Main 대신 Dispatchers.Main.immdiate를 사용하세요

### 무거운 함수에서는 yield를 사용하는 것을 기억하세요

### 중단 함수는 자식 코루틴이 완료되는 걸 기다립니다

### Job은 상속되지 않으며, 부모 관계를 위해 사용됩니다

### 구조화된 동시성을 깨뜨리지 마세요

### CoroutineScope를 만들 때는 SupervisorJob을 사용하세요

### 스코프의 자식은 취소할 수 있습니다

### 스코프를 사용하기 전에, 어떤 조건에서 취소가 되는지 알아야 합니다

### GlobalScope를 사용하지 마세요

### 스코프를 만들 때를 제외하고, Job 빌더를 사용하지 마세요

### Flow를 반환하는 함수가 중단 함수가 되어서는 안됩니다

### 하나의 값만 필요하다면 플로우 대신 중단 함수를 사용하세요
