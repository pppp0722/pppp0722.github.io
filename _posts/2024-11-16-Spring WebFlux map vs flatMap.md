---
title: "Spring WebFlux map vs flatMap"
date: 2024-11-16 23:50:00 +0900
categories: [Spring]
tags: [Spring, Spring WebFlux]
---

Spring WebFlux에서 Reactive Stream 내 데이터 변환에는 `map` 메서드와 `flatMap` 메서드가 사용됩니다.

이번 시간에는 데이터 변환에 사용되는 두 메서드의 차이에 대해 정리해보고자 합니다.

<br>

### map

`map` 메서드는 데이터를 **동기적**으로 변환하기 위해 사용됩니다.

`Stream`, `Optional`과 비슷하게 내부의 데이터만 변환합니다.

```java
Optional<String> optional = Optional.of("hello")
  .map(String::toUpperCase);

Stream<String> stream = Stream.of("hello")
  .map(String::toUpperCase);

Mono<String> mono = Mono.just("hello")
  .map(String::toUpperCase)
```

<br>

당연하게도, 반환 값이 래핑되지 않은 단순 값이기 때문에 `Mono` 및 `Flux`를 반환할 수 없어 비동기 작업이 불가능합니다.

```java
Mono<User> mono = Mono.just("ilhwanee")
  .map(userId -> {
      Mono<User> userMono = userRepository.findUserByUserId(userId);
      return userMono;; // Error!
  });
```

따라서 `map`안에서 Mono 및 Flux를 사용하는 경우 `block` 메서드를 사용해서 값을 꺼내야 하므로 Reactive Stream의 논블로킹 특성이 깨지게 되어 성능 저하의 원인이 됩니다.

<br>

이러한 특성으로 인해 `map` 메서드는 다음과 같은 상황에서 사용됩니다.
- 데이터 변환
- 단순한 계산 작업

<br>

그렇다면 `flatMap`은 어떨까요?

<br>

---

### flatMap

`flatMap`은 Reactive Stream 내 데이터를 **비동기적**으로 변환합니다.

`Flux`에서 데이터 개수 증가가 가능하며, 비슷한 메서드인 `flatMapMany`를 사용하면 `Mono -> Flux`로 변환 가능합니다.

> `Flux -> Mono` 변환은 `reduce` 사용

```java
// flatMap 사용
Mono<Order> mono = Mono.just(uuid)
  .flatMap(id -> {
      Mono<Order> orderMono = orderRepository.findyOrderById(id);
      return orderMono;
  });

// Flux 데이터 개수 증가
  Flux<String> flux2 = Flux.just("hi", "bye")
  .flatMap(value -> Flux.fromArray(value.split("")));

  flux.subscribe(System.out::println);
/** 출력
 * h
 * i
 * b
 * y
 * e
 */

// Mono에서 Flux로 변환
Flux<Post> flux1 = Mono.just("ilhwanee")
  .flatMapMany(author -> {
      Flux<Post> postFlux = postRepository.findPostsByAuthor(author);
      return postFlux;
  });
```

<br>

`flatMap`은 `map`과 달리 비동기 호출이 가능하기 때문에,
- 비동기 호출
- 외부 서비스 API 호출
- DB 작업

등에 사용됩니다.

<br>

---

### 결론

Reactive Redis, WebClient 사용 등 비동기 호출하여 반환 값이 `Mono` 및 `Flux`인 경우 `flatMap`을,

내부 데이터만 변환하는 동기적으로 해결 가능한 경우는 `map`을 사용하면 됩니다!

<br>

---
