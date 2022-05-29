---
title: Chapter 7. 병렬 데이터 처리와 성능
date: '2022-04-25'
tags: ['TIL', 'Java', 'Modern Java In Action']
draft: true
summary: Chapter 7. 병렬 데이터 처리와 성능
---

# Chapter 7. 병렬 데이터 처리와 성능

앞서 `데이터 컬렉션`을 `선언형`으로 제어하는 방법을 살펴보았습니다.  
또한, `외부 반복`을 `내부 반복`으로 바꾸면 `네이티브 자바 라이브러리`가 `Stream` 요소의 처리를 제어할 수 있으므로 개발자는 `Collection`의 처리 속도를 높이기 위해 따로 고민할 필요가
없습니다.

> 💡 가장 큰 특징은 `멀티코어`를 활용해서 `Pipeline` 연산을 실행할 수 있다는 점입니다.

`Java 7`이 등장하기 이전에는 데이터를 `병렬`로 처리하기 위해선, 데이터를 `서브파트`로 `분할`하고 각 `Thread`로 할당 한 후, 의도치 않은 `race condition`이 발생하지 않도록
적절한 `동기화`를 추가한 뒤, `부분 결과`를 합쳐야 했습니다.

`Java 7`은 더 쉽게 `병렬화`를 수행하면서 에러를 최소화할 수 있도록 `Fork/Join Framework` 기능을 제공합니다.

`Stream`을 이용하면 `순차 스트림`에서 `병렬 스트림`으로 자연스럽게 변경할 수 있습니다.

## 7.1 병렬 스트림

`Collector`에 `parallelStream`을 호출하면 `병렬 스트림`이 생성됩니다.

`병렬 스트림`이란 각각의 `Thread`에서 처리할 수 있도록 `Stream`요소를 `여러 청크`로 분할한 `Stream`입니다. 따라서, `병렬 스트림`을 이용하면 모든 `멀티코어 프로세서`가 각각의 `청크`를
처리하도록 할당할 수 있습니다.

아래처럼 `무한 스트림`을 만든 다음 인수로 주어진 크기로 `Stream`을 제한하고, 두 숫자를 더하는 `BinaryOperator`로 `reducing`작업을 수행할 수 있다.

```java
class Foo {
	public long sequentialSum(long n) {
		return Stream.iterate(1L, i -> i + 1)
				.limit(n)
				.reduce(0L, Long::sum);
	}
}
```

기존의 자바 코드는 아래처럼 작성할 수 있습니다.

```java
class Foo {
	public long iterativeSum(long n) {
		long result = 0;
		for (long i = 1L; i <= n; i++) {
			result += i;
		}
		return result;
	}
}
```

특히 `n`이 커진다면 `병렬`로 처리하는 것이 더 좋습니다.

위의 코드를 `병렬 처리` 하려면 결과 변수를 어떻게 `동기화`를 하고, 몇 개의 `Thread`를 사용해야 하며, 숫자는 어떻게 생성할지 등등 많은 고민이 필요합니다.

`병렬 스트림`을 이용하면 이런 걱정 없이 모든 문제를 쉽게 해결할 수 있습니다.

### 7.1.1 순차 스트림을 병렬 스트림으로 변환하기

`순차 스트림`에 `parallel` 메소드를 호출하면 기존의 `함수형 reducing 연산`이 `병렬`로 처리됩니다.

```java
public class ParallelStreams {
	public static long parallelSum(long n) {
		return Stream.iterate(1L, i -> i + 1)
				.limit(n)
				.parallel()
				.reduce(Long::sum);
	}
}
```

이전 코드와 다른 점은 `Stream`이 여러 `청크`로 분할되어 `reducing 연산`을 `병렬`로 처리할 수 있다는 점입니다.

`parallel` 메소드를 호출하면 내부적으로 병렬로 수행해야 한다는 것을 의미하는 `boolean flag`가 설정됩니다. 반대로, `sequential` 메소드를 실행하면 `순차 스트림`으로 변경할 수
있습니다.

`parallel`과 `sequential` 메소드 중 최종적을 호출 된 메소드가 전체 `Pipeline`에 영향을 미칩니다.

> #### 💡 병렬 스트림에서 사용하는 스레드 풀 설정
>
> `병렬 스트림`은 내부적으로 `ForkJoinPool`을 사용합니다.
>
> `ForkJoinPool`은 프로세서 수, 즉 `Runtime.getRuntime().availableProcessors()`가 반환하는 값에 상응하는 `Thread`를 갖습니다.
>
> 특별한 이유가 업다면 `ForkJoinPool`의 `기본값`을 그대로 사용할 것을 `권장`합니다.
