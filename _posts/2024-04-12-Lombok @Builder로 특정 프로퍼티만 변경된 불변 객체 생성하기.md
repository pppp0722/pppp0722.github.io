---
title: "Lombok @Builder로 특정 프로퍼티만 변경된 불변 객체 생성하기"
date: 2024-04-12 18:53:00 +0900
categories: [Java]
tags: [Java, Lombok]
---

불변 객체는 상태 관리가 쉬우며 thread-safe하다는 장점이 있기 때문에 자주 사용됩니다.

> Java에서 14버전부터 도입된 `Record`로 쉽게 불변 객체 생성 가능

<br>

하지만 불변 객체의 특성상 Setter를 사용할 수 없기 때문에 특정 프로퍼티만 변경된 새로운 객체를 생성할 때 수고스러움이 존재합니다.

예를 들어 Client Credentials는 인코딩하여 Header로 주어지고, 나머지 정보는 Body로 주어지는 Oauth 2.0 토큰 발급 상황을 예로 들어보겠습니다.

```
POST /oauth/token HTTP/1.1
Host: example.com
Authorization: Basic Base64Encoded(ClientID:ClientSecret)
Content-Type: application/json

{
  "grant_type": "client_credentials",
  "scope": "scope1 scope2"
}
```

<br>

현재 DTO가 불변 객체로 설계되었다면, Controller에서 Application Service로 토큰 발급을 요청하기 전에 헤더의 값을 요청 객체에 포함하는 과정이 필요합니다.

따라서 실제 요청 받은 Request DTO에 Client Credentials를 설정해야 하지만, DTO는 불변 객체이므로 기존 Request DTO를 토대로 새로운 객체를 생성해야 합니다.

```java
public record ExampleRequestDto(
        String clientId,
        String clientSecret,
        String grantType,
        String scope
) {
    // 기존 Request DTO를 토대로 헤더의 Client Credentials를 넣은 새로운 불변 객체 생성
    public ExampleRequestDto withClientCredentials(String clientId, String clientSecret) {
        return new ExampleRequestDto(
                clientId, clientSecret, this.grantType, this.scope);
    }
}
```

<br>

예시 코드는 4개의 프로퍼티를 가지므로 코드가 비교적 간단하지만, 더 많은 프로퍼티를 가지는 경우는 복잡해질 수 있습니다.

만약 빌더 패턴을 사용한다면,

```java
@Builder
public record ExampleRequestDto(
        String clientId,
        String clientSecret,
        String grantType,
        String scope,
        String extraProperty1,
        String extraProperty2,
        String extraProperty3,
        String extraProperty4
) {
    // 기존 Request DTO를 토대로 헤더의 Client Credentials를 넣은 새로운 불변 객체 생성
    public ExampleRequestDto withClientCredentials(String clientId, String clientSecret) {
        return ExampleRequestDto.builder()
            .clientId(clientId)
            .clientSecret(clientSecret)
            .grantType(this.grantType)
            .scope(this.scope)
            .extraProperty1(this.extraProperty1)
            .extraProperty2(this.extraProperty2)
            .extraProperty3(this.extraProperty3)
            .extraProperty4(this.extraProperty4)
            .build();
    }
}
```

<br>

코드가 많이 복잡한 모습을 확인할 수 있습니다.

그렇다면 이러한 Lombok @Builder를 사용하는 상황에 조금 더 깔끔한 코드를 작성하려면 어떻게 해야 할까요?

<br>

---

# @Builder(toBuilder = true)

@Builder 어노테이션의 toBuilder 속성을 true로 만들면, toBuilder 메서드를 사용할 수 있습니다.

`toBuilder()` 메서드는 기존 객체의 프로퍼티를 가지고 있는 새로운 Builder 객체를 만듭니다.

따라서 변경이 필요한 프로퍼티만 수정하면 되므로 깔끔한 코드 작성이 가능해집니다.

```java
@Builder(toBuilder = true)
public record ExampleRequestDto(
        String clientId,
        String clientSecret,
        String grantType,
        String scope,
        String extraProperty1,
        String extraProperty2,
        String extraProperty3,
        String extraProperty4
) {
    // 기존 Request DTO를 토대로 헤더의 Client Credentials를 넣은 새로운 불변 객체 생성
    public ExampleRequestDto withClientCredentials(String clientId, String clientSecret) {
      return this.toBuilder()
        .clientId(clientId)
        .clientSecret(clientSecret)
        .build();
    }
}
```

<br>

앞선 코드와 비교하면 변경해야 할 프로퍼티만 선언하면 되므로, 훨씬 간단해진 코드를 확인할 수 있습니다.

toBuilder 속성이 false인 빌더 객체와 비교하면, 단순히 클래스에 다음 메서드만 추가됩니다.

```java
    public ExampleRequestDtoBuilder toBuilder() {
        return new ExampleRequestDtoBuilder().clientId(this.clientId).clientSecret(this.clientSecret) ... 길어서 생략
    }
```

<br>


이처럼 불변 객체 사용 시 다음과 같은 상황이라면,
- 프로퍼티가 많아서 빌더 패턴을 사용해야 함
- 기존 객체의 프로퍼티를 유지하되 특정 프로퍼티를 수정해야 함

Lombok @Builder의 toBuilder() 메서드 사용을 고려하는 것도 좋은 방법이라고 생각합니다!

<br>

---
