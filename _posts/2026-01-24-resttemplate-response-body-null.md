---
title: "[Spring] ClientHttpResponse의 Body는 왜 한 번만 읽을 수 있을까?"
excerpt_separator: "<!--more-->"
categories:
  - development

tags:
  - Spring
  - RestTemplate
  - ClientHttpRequestInterceptor
  - BufferingClientHttpRequestFactory
---
26년 새해 각오는 주 1회가 목표였으나, 2주 만에 블로그를 작성하게 됐네요. 😂

오늘은 업무 중 겪게 된 간단한 트러블슈팅 내용을 올려볼까 합니다. 

그동안 다양한 API 연동 작업을 해왔지만 몰랐던 내용을 트러블슈팅 과정 중 알게 되었고, 같이 일하는 다른 개발자도 생소한 내용이라고 하여 기록을 해봅니다.

<!--more-->

트러블슈팅 과정을 간략하게 설명을 드리면, 신규 기능을 위해 새로운 업체와 API 통신 연동이 필요했고,

해당 업체 API 가이드에 모든 API 호출 전 1회성 토큰(accessToken)을 획득하여, 호출 API 에 추가를 하여 호출해야 인증이 통과되는 방식이었습니다.

### 개발 과정

기본적으로 작업이 필요한 모듈은 restTemplate 을 사용하고 있어서, 신규 업체를 위한 restTemplate 을 빈으로 새로 만들고 해당 restTemplate 에 interceptor 를 추가하여 위에 설명한 accessToken 관련 로직을 공통화해서 구현할 계획이었습니다.

요즘 모두들 사용하는 ai agent 를 사용하여 요건을 전달하고 관련 restTemplate 빈 생성과 해당 restTemplate 을 사용하는 전용 adapter 및 API 가이드 문서의 request, response 를 복사하여, 각 API 를 테스트할 수 있는 controller 까지 만들도록 요청을 했습니다.

그렇게 개발된 코드를 간략히 확인 후 테스트하는 과정 중 바로 오늘 블로그를 작성하는 이슈가 발생했습니다.

### Response Body 가 사라졌어요!

이슈는 바로 token 을 요청한 API 응답을 파싱하는 과정에서 Json 형식이 아니라는 에러가 발생하였고,

agent 에 작업 요청 시 기존에 사용하던 코드 파일을 첨부하였기에 다른 restTemplate 과 동일하게 logging 을 위한 interceptor 가 있었고, 

로그를 통해 정상적으로 응답을 받는 것과 생성된 token 도 확인이 됐는데, 그 응답을 파싱하는 과정에 에러가 발생하니 당황스러운 상황이었습니다.

당시 agent 에서 생성해준 문제의 코드는 아래와 같았습니다.

```kotlin
class TokenInterceptor(
    private val restTemplateLoggingInterceptor: RestTemplateLoggingInterceptor,
) : ClientHttpRequestInterceptor {

    private val tokenRestTemplate: RestTemplate by lazy {
        val restTemplate = RestTemplate()
        restTemplate.interceptors.add(restTemplateLoggingInterceptor)
        restTemplate
    }

    override fun intercept(request: HttpRequest, body: ByteArray, execution: ClientHttpRequestExecution): ClientHttpResponse {
        val token = fetchToken() // 별도 API를 통해 토큰을 받아오는 로직
        var modifiedBody = body
        // 토큰을 accessToken 항목에 설정하는 코드
        ...
        
        return execution.execute(request, modifiedBody)
    }

    private fun fetchToken(): String? {
        ...
        val response = tokenRestTemplate.postForEntity(url, requestDto, TokenResponseDto::class.java)
        ...
    }
}
```

눈치가 빠른 분들은 벌써 원인을 찾으셨을 거라 생각합니다. 😭

### ClientHttpRequestInterceptor 와 ClientHttpResponse의 One-time Read

원인은 바로 interceptor 내에서 사용된 restTemplate 생성 방식이었습니다.

저도 몰랐지만, ClientHttpResponse의 Body 는 InputStream 형태로 제공되어, 한 번만 읽을 수 있는 특징이 있었습니다.

왜 한 번만 읽을 수 있는지에 대한 AI 의 답변은 아래와 같았습니다.

>왜 한 번만 읽을 수 있나요?
>1. InputStream의 특성
>``` java
>   // InputStream은 "스트림" 방식으로 데이터를 읽습니다
>   // 읽은 데이터는 버퍼에서 사라지고, 내부 포인터가 앞으로 이동합니다
>   InputStream inputStream = response.getBody();
>   byte[] data = inputStream.readAllBytes(); // 모든 데이터를 읽음 → 포인터가 끝으로 이동
>   // 다시 읽으려고 하면?
>   byte[] data2 = inputStream.readAllBytes(); // 빈 배열 반환 (읽을 데이터 없음)
> ```
> 2. 네트워크 스트림의 특수성
> HTTP 응답 Body는 네트워크 소켓에서 직접 읽어오는 스트림입니다
> 일반 파일과 달리 reset()이나 mark()를 지원하지 않습니다
> 데이터가 한 번 읽히면 네트워크 버퍼에서 제거됩니다

앞에서 힌트를 드린 것과 같이 tokenRestTemplate 을 생성하는 코드를 보면 아래 한 줄이 있습니다.
```kotlin
restTemplate.interceptors.add(restTemplateLoggingInterceptor)
```
바로 restTemplate 응답을 logging 하는 Interceptor 에 의해 이미 Body 를 한 번 읽었기 때문에, fetchToken 의 응답은 비어버리는 것이었습니다. 🤦🏻

### 해결 방법
앞서 말씀드린 대로 기존 코드들에 logging 용 interceptor 에서 사용하고 있었기에 해당 코드를 참고하니 바로 해결책이 나왔습니다.

```kotlin
// 문제 코드
val restTemplate = RestTemplate()
// 기존 코드
val restTemplate = RestTemplate(BufferingClientHttpRequestFactory(SimpleClientHttpRequestFactory()))
```
BufferingClientHttpRequestFactory 를 통해 restTemplate 을 생성하여 호출하면 아래 흐름으로 HTTP 응답 Body 를 메모리에 버퍼링하여 여러 번 읽을 수 있게 해주는 BufferingClientHttpResponseWrapper 가 생성됩니다.

```java
BufferingClientHttpRequestFactory
        │
        ▼
BufferingClientHttpRequest (내부 클래스)
        │
        ▼ execute() 호출 시
BufferingClientHttpResponseWrapper 생성
```
BufferingClientHttpResponseWrapper 클래스 구현을 아래와 같이 보면 응답을 저장할 수 있는 버퍼가 있고, 

Body 를 요청할 때 매번 버퍼를 응답으로 전달하기 때문에 이슈가 발생하지 않는 것이었습니다.

```java
final class BufferingClientHttpResponseWrapper implements ClientHttpResponse {

  private final ClientHttpResponse response;

  @Nullable
  private byte[] body; // 버퍼 저장소
  
  ...
  
  @Override
  public InputStream getBody() throws IOException {
    if (this.body == null) {
      this.body = StreamUtils.copyToByteArray(this.response.getBody());
    }
    return new ByteArrayInputStream(this.body); // 위에 저장된 버퍼를 요청할 때마다 InputStream 새로운 객체를 제공
  }
```

### 마무리

누군가에게는 너무 쉬운 내용일 수 있고, 누군가에게는 저처럼 생소한 분도 계실 거라 생각합니다.

지난번 포스팅에서 말씀드린 것처럼 아무도 모르는 얘기를 포스팅하는게 아니라 그저 제가 겪은 일을 조금 더 자주 올리는게 올해 목표이기에

다 아는 내용이라도 넓은 아량 베풀어 주시길 바랍니다.

또 빠른 시일 내로 다음 포스팅으로 찾아오겠습니다.

그럼 이만. 🥕👋🏼🖐🏼
