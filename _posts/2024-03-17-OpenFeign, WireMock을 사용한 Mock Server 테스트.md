---
title: "OpenFeign, WireMock을 사용한 Mock Server 테스트"
author: ilhwanee
date: 2024-03-17 21:53:00 +0900
categories: [Spring]
tags: [Spring, OpenFeign, WireMock]
---

백엔드를 개발하다 보면 MSA 환경에서 타 Service의 리소스를 요청하거나 Open API를 사용하는 등, 타 Server로 API 호출은 자주 이뤄집니다.

따라서 HTTP 클라이언트는 자주 사용되는, 서버 개발에 필수적인 요소입니다.

Java 기반인 Spring Framework를 사용하는 경우 다음 HTTP 클라이언트 라이브러리를 고려할 수 있습니다.
```
- RestTemplate: Spring Framework에서 제공하는 전통적인 방식의 HTTP 클라이언트
- WebClient: Spring5에서 소개된 reactive HTTP 클라이언트
- OpenFeign: Netflix가 개발 시작한, 선언적 방식으로 사용할 수 있는 HTTP 클라이언트
```

<br>

`RestTemplate`는 오랜 기간 동안 사랑 받아온 라이브러리로 **대규모 커뮤니티와 다양한 레퍼런스가 존재**하지만, 블로킹 I/O를 사용하기 때문에 서버의 응답이 느릴수록 성능이 안좋습니다.

`WebClient`는 비동기 논블로킹 I/O를 기반으로 **높은 동시성을 요구하거나 서버의 응답이 느릴수록 RestTemplate과 큰 성능 차이**를 보입니다. 또한 Spring이 밀고 있는 HTTP 클라이언트이며 RestTemplate은 WebClient의 등장 이후로 레거시 취급을 받기도 합니다.

`OpenFeign`은 RestTemplate과 동일하게 블로킹 I/O를 사용합니다. **선언적 API를 제공하기 때문에 배우기 쉽고 가독성이 좋다는 장점**이 있습니다. 또한 인터페이스 형태이기 때문에 mocking이 쉽습니다.

<br>

그 중에서 이번 시간에는 Netflix가 처음 개발을 시작하여 현재는 Spring Cloud 프로젝트로 이관된 **OpenFeign**을 다뤄보겠습니다.

**WireMock**을 사용하여 mock server를 띄운 간단한 FeignClient 사용 테스트를 작성할 것입니다.

> mock server를 띄우지 않고 FeignClient를 mocking하는 것또한 좋은 방법

<br>

테스트 환경은 다음과 같습니다.
```
- org.springframework.boot 3.0.13
- org.springframework.cloud:spring-cloud-starter-openfeign 4.0.3
- org.springframework.cloud:spring-cloud-contract-wiremock:4.0.4
- JUnit5
```

<br>

---

# Setting

기본적인 설정은 어렵지 않습니다.

```java
@SpringBootApplication
@EnableFeignClients
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

라이브러리 추가에 성공했다면, `@EnableFeignClients`를 추가해 `@FeignClient`가 스캔될 수 있게 합니다.

`FeignClient`에 대해선 아래에서 설명하겠습니다.

<br>

다른 서버의 요청 시 timeout 시간이나 어떤 인코더, 디코더를 사용할지 등의 설정은 configuration class와 configuration properties 모두 가능합니다.

```java
@Configuration
public class FooConfiguration {
    @Bean
    public Contract feignContract() {
        return new feign.Contract.Default();
    }

    @Bean
    public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
        return new BasicAuthRequestInterceptor("user", "password");
    }
}
```

```
spring:
    cloud:
        openfeign:
            client:
                config:
                    feignName:
                        url: http://remote-service.com
                        connectTimeout: 5000
                        readTimeout: 5000
                        loggerLevel: full
                        errorDecoder: com.example.SimpleErrorDecoder
                        retryer: com.example.SimpleRetryer
                        defaultQueryParameters:
                            query: queryValue
                        defaultRequestHeaders:
                            header: headerValue
                        requestInterceptors:
                            - com.example.FooRequestInterceptor
                            - com.example.BarRequestInterceptor
                        responseInterceptor: com.example.BazResponseInterceptor
                        dismiss404: false
                        encoder: com.example.SimpleEncoder
                        decoder: com.example.SimpleDecoder
                        contract: com.example.SimpleContract
                        capabilities:
                            - com.example.FooCapability
                            - com.example.BarCapability
                        queryMapEncoder: com.example.SimpleQueryMapEncoder
                        micrometer.enabled: false
```

> 더욱 자세한 설정 방법은 [공식문서](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/#spring-cloud-feign-overriding-defaults) 참고

<br>

---

# FeignClient

다음은 `FeignClient`의 코드 예시입니다.

```java
@FeignClient(
        name = "test-client",
        url = "${url.test}",
        configuration = TestClientConfig.class
)
public interface TestClient {

    @GetMapping(value = "/test")
    ResponseEntity<TestResponseDto> test();
}
```

호출할 서버의 API 명세를 `@FeignClient` 어노테이션이 달린 인터페이스에 작성하면, OpenFeign은 런타임에서 해당 명세의 구현체를 자동으로 생성하고 사용하는 **동적 프록시 생성** 방법을 사용하여 실제 클라이언트를 호출합니다.

따라서 구현 방법을 몰라도 쉽게 사용할 수 있으며, 이 때 인터페이스는 Spring Web의 RestController와 유사하여 더욱 쉽게 사용할 수 있습니다.

> 사용하는 클래스와 어노테이션도 RestController와 유사함

<br>

`@FeignClient`의 주요 속성은 다음과 같습니다.
```
- name: FeignClient의 이름
- url: 실제 클라이언트의 URL
- configuration: HttpClient, Encoder, ErrorDecoder 등 FeignClient 설정 클래스
```

>더욱 자세한 내용은 [공식문서](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/#spring-cloud-feign) 참고

<br>

앞서 설명드렸던 configuration class를 `@FeignClient`의 속성으로 추가하여 해당 클라이언트와 통신 시 설정을 추가할 수 있습니다.

특히 클라이언트가 에러를 응답할 때 핸들링이 필요한데, 이를 configuration class에서 `ErrorDecoder`를 등록하거나, 호출 시에 발생하는 `FeignException`을 처리하는 방법이 있습니다.
>FeignException은 다음으로 세분화
> - `FeignClientException`: 4XX 에러
> - `FeignServerException`: 5XX 에러

Exception을 처리하는 방법은 직관성이 떨어지기에, ErrorDecoder를 사용하는 방법이 주로 사용됩니다.

```java
public class MyErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        if (response.status() == 404) {
            return new NotFoundException("Not Found");
        } else if (response.status() == 500) {
            return new InternalServerErrorException("Internal Server Error");
        }
        // 기본적으로 FeignException 반환
        return FeignException.errorStatus(methodKey, response);
    }
}
```

ErrorDecoder는 에러 응답에서 처리할 수 있는 예외로 반환하여 사용하며, `RestControllerAdvice`와 함께 사용하면 궁합이 좋습니다.

<br>

마지막으로 위 코드 예시의 `TestClient`의 `test()` 메서드를 호출하면, 런타임에서 다음 동작이 이뤄집니다.
```
1. 메서드 호출
2. "${url.test}/test"에 GET 요청
3. 실제 클라이언트가 요청을 처리하고 응답
4. 응답 받은 HTTP Response를 반환 타입인 ResponseEntity<TestResponseDTO>로 매핑
5. return
```

<br>

---

# WireMock

이제 테스트로 넘어가겠습니다.

WireMock을 간단히 사용하기 위해서는 어노테이션을 사용하여 설정합니다.

```java
@AutoConfigureWireMock(port = 0)
@TestPropertySource(properties = {"url.test=http://localhost:${wiremock.server.port}"})
@ActiveProfiles("test")
class Test { ... }
```

사용한 어노테이션은 다음과 같습니다.
- `@AutoConfigureWireMock(port = 0)`: WireMock 설정을 자동으로 해주며, 해당 어노테이션으로 설정했으면 `WireMock` 클래스의 static 메서드를 사용하여 stub 가능 (port가 0이면 랜덤 port)
- `@TestPropertySource(properties = "...")`: 현재 application.yml 등 설정 파일에는 `url.test`의 값이 실제 클라이언트의 주소로 설정되어 있어, 테스트 환경에는 테스트 클래스에 `@TestPropertySource`를 달아 테스트 프로파일에서 설정을 덮어 써야 mock server로 요청

<br>

mock server를 띄울 설정을 완료되었으니, 테스트 코드를 작성해보겠습니다.

WireMock은 다음과 같이 `WireMock.stubFor`을 사용해 stub을 설정할 수 있습니다.

```java
// given
stubFor(get(urlEqualTo("/test"))
    .willReturn(aResponse()
        .withStatus(HttpStatus.OK.value())
        .withHeader("Content-Type", MediaType.APPLICATION_JSON_VALUE)
        .withBody("{\"success\":true}")));
```

해당 stub은 method가 get에 path가 /test인 요청에 대한 가짜 응답을 만듭니다.

특히 urlEqualsTo의 값은 FeignClient에서 설정한 값과 동일해야 함을 주의해야 합니다.

<br>

이제 `TestClient`를 호출하고 assertion을 작성하면 테스트 코드 작성이 완료됩니다.

```java
// when
ResponseEntity<TestResponseDto> response = testClient.test();

// then
assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
assertThat(response.getBody()).isNotNull();
assertThat(response.getBody().getSuccess()).isTrue();
```

<br>

---
