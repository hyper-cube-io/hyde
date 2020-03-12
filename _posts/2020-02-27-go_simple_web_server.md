---
layout: post
title: Go로 간단히 웹서버 만들기
author: kwSeo
tags: [go, web]
---

# Go로 간단히 웹서버 만들기

## 개요
요즘 Go 언어의 주가가 지속적으로 오르면서 Go 언어에 대해서 들어보지 못한 사람은 없을 것이다. 
Go 언어는 배우기 쉽고 간단함에도 불구하고 매우 좋은 성능을 자랑한다. (그 특유의 단순함으로 인해 개발자마다 극명하게 호불호가 갈리기도 한다.) 또한, 내부적으로 가지고 있는 기본 라이브러리도 풍부해서 생산성도 좋다. 여기에서는 따로 의존성없이 Go의 기본 라이브러리만 사용해서 간단히 웹서버를 개발해볼 것이다. Go 언어를 할 줄 몰라도 다른 프로그래밍 언어를 접해본 적이 있다면 코드를 대략적으로 이해하는데 어려움은 없을 것이다.


## Getting Started

### Hello World
늘 그렇듯이 `Hello World` 예제부터 살펴보자. 아래의 코드는 `/hello` 경로로 요청시 "Hello World!!"라는 문자열을 반환하는 웹서버의 Go 코드이다.

``` go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
    log.Println("starting Web Server")      // (1)

    http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {    // (2)
        fmt.Fprintln(w, "Hello World!!")
    })

    err := http.ListenAndServe(":8080", nil)    // (3)
    if err != nil {             // (4)
        log.Fatalln(err)
    }
}
```

여기서 핵심은 `net/http` 패키지이다. `net/http` 패키지는 Go에서 HTTP 서버 및 클라이언트와 관련된 기능을 제공하는 기본 내장 모듈이다. Go 역시 다른 언어들처럼 보다 간편하고 좋은 성능을 위해서 Gin, Echo 등의 웹 프레임워크가 존재하지만 간단한 웹서버의 경우 `net/http` 만으로도 충분할 것이다.

위 코드를 하나씩 살펴보자. `(1)`은 간단하게 서버시작을 알리는 로그이다. 역시 내장된 `log` 모듈을 사용하여 로그를 남기고 있다. `(2)`부터 중요한 부분이다. `net/http` 모듈을 사용하여 `/hello` 경로를 등록하고 있다. 두 번째 파라미터로 넘기는 값은 함수이다.  Go 언어는 high-order function을 지원한다. `/hello` 경로가 호출되면 함께 등록된 함수가 실행될 것이다. 이때 등록된 함수의 첫 번째 파라미터(`http.ResponseWriter`)는 응답과 관련된 기능을, 두 번째 파라미터(`r *http.Request`)는 들어온 요청과 관련된 기능을 제공한다. 함수 내에서는 간단하게 `Hello World`를 응답으로 내보내고 있다. 집고 넘어가야할 것이, `fmt.Println`이 아니다. `fmt.Fprintln`이다. 이 함수는 첫 번째 파라미터로 주어지는 Writer에 해당 문자열을 출력하는 기능을 수행한다. C 언어를 접해본 사람은 조금 익숙한 방식의 함수일 것이다. Java 언어에 익숙한 사람은 Go의 Writer를 Java의 OutputStream이라고 생각하면 쉽게 이해될 것이다. `(3)`은 8080 포트로 바인딩해서 서버를 시작하고 요청을 기다린다. 이때 `http.ListenAndServe` 함수의 반환값은 `error`이다. 대부분의 Go 함수는 내부적으로 예상되는 에러가 있다면 마지막 반환값으로 에러를 반환한다.
만약 에러 반환값이 nil(다른 언어의 null과 동일하다)이 아니라면 함수 수행 중 에러가 발생한 것이라는 의미이다.  그래서 `(4)`에서 err가 nil이 아닌 경우 Fatal 로그를 남기고 있다. 그리고 추가적으로 `:=` 연산자는 변수를 생성하고 바로 값을 할당하는 것을 의미한다.

그런데 함수의 마지막 반환값은 또 무슨 말인가 싶은 사람도 있을 것이다. 기본적으로 Go 언어의 함수는 2개 이상의 반환값을 가질 수 있다. 아래의 예제처럼 여러 개의 반환값을 여러 개의 변수로 한 번에 받아서 사용할 수 있다.

```go
func DoSomething(arg string) (string, int, error) {
    ...
}

result1, result2, err := DoSomething("hello")
if err != nil {
    ...
}
```

여기서 Go를 접해보지 못한 사람의 경우 `*http.Request`처럼 타입 앞에 붙은 `*`가 무엇을 의미하는지 궁금할 것이다. 이는 C 언어처럼 해당 타입의 포인터 임을 의미한다. Go 언어는 포인터를 지원한다. 걱정하지는 말자. Go 언어의 포인터는 훨씬 간단하며 일반 객체를 다루는 것과 크게 다르지 않다. 대부분 단순한 용도로만 사용된다. 그럼 왜 첫 번째 파라미터는 일반 타입이고 두 번째는 포인터인가하는 의문이 생길 수도 있는데, 이는 Go 언어에서 더 깊숙히 들어가야하는 것들이니 일단 넘어가도록 하자.

또 다른 궁금증은 함수명이나 타입명이 소문자로 시작하는 것도 있고 대문자로 시작하는 것도 있다는 것에 의문을 표할 것이다. 이는 절대 오타가 아니다. Go 언어는 개발자 간에 코딩 컨벤션을 통일하는 것을 선호한다. 그래서 일부 기능은 함수 또는 타입의 이름이 무엇이냐에 따라서 갈린다. 대문자로 시작하는 이름은 Exported로, Java로 치면 public 메소드 또는 클래스라고 생각하면 된다. 반대로 소문자로 시작한다면 같은 패키지 내에서만 사용할 수 있다.

### Routing
앞에서 봤던 것처럼 `net/http`에서도 HandleFunc라는 함수를 이용하여 경로와 실행될 함수를 매핑할 수 있다.
함수를 넘겨주는 방법 외에도 직접 Handler를 구현하는 방법도 존재한다.

#### Handler
Handler 인터페이스는 아래와 같은 형태를 하고 있다.

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

별거없다. 간단하게 ServeHTTP 함수를 구현하면 된다. 한 번 Handler를 구현해서 사용해보자.

``` go
type MyHandler struct {
    myField string
}

func (m *MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    ...
}
```

위 예제는 MyHandler라는 타입은 선언하고 MyHandler에 ServeHTTP를 정의하고 있다. Go 언어는 Duck Typing 언어이기 때문에 명시적으로 특정 인터페이스를 구현한다고 선언할 필요없이 해당 인터페이스가 정의하는 함수와 동일한 함수를 구현하고 있다면 해당 인터페이스 타입으로 취급한다. 이제 핸들러를 등록해보자.

```go
func main() {
    myHandler := MyBandler[}
    http.Handle("/my-handler", myHandler)
    ...
}
```

HandleFunc 대신 Handle이라는 함수를 사용하고 있다. 이를 통해서 간단하게 핸들러를 등록하여 사용할 수 있다.

#### Predefined Handler
사실 Go 언어는 보편적으로 많이 사용되는 성격의 핸들러를 미리 구현하여 제공하고 있다.

##### FileServer
이름에서 알 수 있듯이 정적 파일 서버 역할을 하는 햄들러이다. 파라미터로 경로를 지정해주면 해당 경로 내의 파일을 호스팅해준다. 아래의 코드는 사용 예이다.

```go
http.Handle("/", http.FileServer(http.Dir("/tmp")))
```

##### NotFoundHandler
이름 그대로 404 NOT_FOUND를 반환하는 핸들러이다.

```go
http.Handle("/resources", http.NotFoundHandler())
```

##### RedirectHandler
역시 이름에서 바로 알 수 있듯이 redirect를 처리하는 핸들러이다.

```go
http.Handle("/redirect-test", http.RedirectHandler("https://google.com", http.StatusTemporaryRedirect))
```

##### TimeoutHandler
주어진 시간내 요청이 마무리되지 않으면 타임아웃을 발생시킨다.

```go
http.Handle("/timeout-test", http.TimeoutHandler(MyHandler{}, 3*time.Second, "test timeout"))
```

##### ReverseProxy
들어온 요청을 주어진 URL로 프록시한다.  `net/http` 패키지가 아닌 `net/httputil` 패키지임을 유의한다.

```go
    parsedUrl, err := url.Parse("http://localhost:8888")
    if err != nil {
        log.Fatal(err)
    }
    http.Handle("/proxy-test", httputil.NewSingleHostReverseProxy(parsedUrl))
```

### Parameters
이번에는 요청과 응답을 다루는 방법에 대해서 알아보고자 한다. 앞에서 핸들러 또는 핸들러 함수를 구현할 때 사용했던 `http.Request`가 중심이 될 것이다.

#### Query Parameters
`http.Request`의 URL 필드를 통해서 얻을 수 있다.

``` go
func ServeHTTP(w http.ResponseWriter, r *http.Request) {
    value := r.URL.Query().Get("key")
    ...
}
```

#### Request Body
Go 언어는 기본적으로 JSON을 다룰 수 있는 모듈을 제공한다. `encode/json` 패키지를 통해 해당 기능을 사용할 수 있다. 아래의 예제는 Content-Type이 `application/json`인 요청 body 값을 직접 정의한 `SomeType`이라는 구조체로 변환하는 코드이다. 

``` go

type SomeType struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

...

body := SomeType{}
err := json.NewDecoder(r.Body).Decode(&body)
if err != nil {
    ...
}
```

`SomeType` 구조체의 필드 뒤에 붙은 `json...` 값은 해당 필드의 태그이다. Java의 어노테이션(`@`)이라고 `SomeType`의 객체를 생성하고 그 주소값(`&`)을 Decode 함수에 전달하다. 이제 Decode 함수 내에서 주어진 포인터를 사용하여 알맞은 값을 할당해줄 것이다.

### Response

#### Text
`http.ResponseWriter`는 Go 언어의 `io.Writer` 인터페이스를 구현하고 있다. 때문에 앞에서 봤던 것처럼 `fmt` 패키지의 함수들과 함께 사용할 수 있다.

```go
n, err := fmt.Fprintf(w, "hello world: %d", arg)
```

#### JSON
Request Body를 JSON으로 바꾸던 것과 비슷한 방법을 통해서 JSON으로 응답을 할 수 있다.
아래의 코드를 통해서 요청자는 JSON 포맷의 데이터를 받게 될 것이다.

```go
body := SomeType{
    Name: "kwseo",
    Age: 10,
}
err := json.NewEncoder(w).Encode(body)
```
### net/httputil

### 끝마침
지금까지 웹서버에서 가장 필수적이라고 할 수 있는 API 라우팅 설정과 요청/응답을 다루는 방법을 `net/http` 패키지(약간의 `net/httputil`도...)를 통해서 간단히 알아보았다. 이외에도 더 많은 기능들을 제공하지만, 이정도만으로도 작은 웹서버를 만드는 것은 별로 어렵지 않을 것이다. 나중에 기회가 된다면 더 소개해보도록 하겠다.
이 만큼의 기능만으로도 유용하게 사용할 수 있겠지만, 앞에서도 언급했듯이 고급 기능과 더 간편한 사용을 원한다면 그땐 gin-gonic, echo, fastHTTP 등 잘 만들어진 프레임워크를 가져다 사용하는 것이 더 편할 것이다.