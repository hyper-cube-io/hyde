---
layout: post
title: Spring 5 with Kotlin
author: kwSeo
tags: [spring, kotlin]
---

# 깔끔한 Spring 5 Webflux

Spring 5에 들어가면서 많은 변화가 있었습니다.
기본적으로 Webflux라는 비동기, 리엑티브 웹 프레임워크가 등장했고, Pivotal & Spring의 Reactive 구현체인 Reactor가 본격적으로 적용되기 시작했습니다. 게다가 Spring boot부터는 공식적으로 JSP를 지원하지 않더니, Spring 5를 사용하는 Spring boot 2부터는 기본 내장 웹서버가 톰캣에서 네티로 바뀌었습니다. 그렇기 때문에 기본적으로 서블릿도 사용하지 않죠. 그러기에 Webflux에서는 독자적으로 구현한 ServerRequest, ServerResponse 등을 사용합니다.
(물론 Spring MVC가 완전히 없어진 것은 아니기 때문에 원한다면 추가적은 설정을 통해서 이전과 같이 JSP와 톰캣을 사용할 수는 있습니다.)

Webflux에서는 기존 Spring MVC에서 사용하던 방법과 동일하게 어노테이션(@Controller, @RequestMapping 등)을 통해서 컨트롤러를 구현할 수 있지만 새로운 RouterFunction이라는 것을 이용해서 구현할 수도 있습니다. SpringOne platform의 키노트에서는 'XML은 똥이다'라고 말하면서 @Configuration 어노테이션과 함께 빈 설정을 코드내에서 하도록 권장하더니, 이번에는 어노테이션도 마음에 안들기 시작했는지 컨트롤러를 어노테이션을 통해 구현하는 것이 아닌 함수형 페러다임을 따라서 구현하는 방법을 내놓았습니다.

![XML is ddong]({{site.url}}/public/posts_images/xml_is_ddong.png)

**Phil Webb; Pivotal Spring Boot Lead**


![annotation to functional & KotlinDSL]({{site.url}}/public/posts_images/annotation_to_functional_and_kotlin.png)

**Sébastien Deleuze; Pivotal as a Spring Framework and Reactor commiter**

아래의 코드는 Spring Webflux에서 RouterFunction을 구현하는 예제입니다.

```java
    @Bean
    public RouterFunction<ServerResponse> routerFunction(SomeHandler someHandler) {
        return nest(accept(TEXT_HTML), 
                    route(GET("/"), request -> ok().contentType(TEXT_HTML).render("index")))
                .andNest(path("/api/v1/some").and(accept(APPLICATION_JSON)),
                    route(GET("/"), someHandler::getDataAll)
                        .andRoute(GET("/{id}"), someHandler::getData)
                        .andRoute(POST("/{id}"), someHandler::postData)
                        .andRoute(DELETE("/{id}"), someHandler::deleteData)
                        .andRoute(PUT("/{id}"), someHandler::putData));
    }
```

웹서버에서 제공하는 API를 한곳에서 볼 수 있고 URL과 처리 로직간에 어노테이션이 아니라 함수를 지정함으로써 보다 명확하기는 하지만... 아직 저는 RouterFunction이 깔끔하다는 또는 가독성이 좋다는 말은 들어보지 못한 것 같습니다. 저도 그다지 깔끔하다는 생각이 들지는 않네요. 그런데 Spring 5에는 저 코드를 깔끔하고 보기 좋게 작성할 수 있는 방법을 제공합니다. 사실 위 Sébastien Deleuze의 PPT에도 힌트가 있습니다. 바로 개발에 Java가 아닌 Kotlin을 이용하는 것이지요!


# Spring 5 with Kotlin

최근 여러 유명 프레임워크에서 Kotlin을 정식으로 지원하겠다는 소식이 들려왔습니다.
대표적으로 Spring Framework는 5 버전부터 Kotlin을 정식으로 지원하며, Spark, Vert.x도 Kotlin을 지원하기 시작했습니다.
여기에서는 그중 Spring Framework를 중심으로 Kotlin를 사용해 확장된 기능에 대해서 알아보고자 합니다.

![kotlin in spring]({{site.url}}/public/posts_images/kotlin_in_spring.png)

**spring 5에는 0.4%나 Kotlin으로 작성되었다. 이는 Spring에서 무려 2번째로 많이 사용된 언어라는 의미한다!**

![spring initializr]({{site.url}}/public/posts_images/spring_initializr_with_kotlin.png)

**Kotlin을 위해 구성된 Spring 프로젝트 템플릿은 [SPRING INITIALIZR](https://start.spring.io/)에서 손쉽게 구성하여 받을 수 있다.**


## Spring Webflux

### RouterFunction

물론 특별히 프레임워크나 라이브러리에서 Kotlin을 지원하지 않더라도 Java와 호환성이 좋기 때문에 별로 큰 문제없이 Kotlin을 사용할 수는 있습니다. Spring 5에서 일부 모듈이 Kotlin으로 개발되었다고는 하지만 해당 모듈들을 살펴보면 실제 핵심 로직을 구현한 것은 아닙니다. Kotlin으로 작성된 부분은 주로 기존 기능을 확장하여 Java로는 불가능하거나 매우 장황해지는 부분을 보다 편하고 깔끔한 코드로 작성할 수 있도록 돕는 역할을 하는 모듈이 대부분입니다. Java로는 표현하는데 한계가 있던 부분들을 Kotlin으로 개선한 것이죠. 

이 확장 모듈 중에는 Webflux에서 사용할 수 있는 모듈이 있으며 그중 가장 대표적인 것이 RouterFunctionDSL입니다. RouterFunctionDSL을 사용하면 위에서 잠깐 살펴봤던 RouterFunction을 보다 깔끔하게 작성할 수 있습니다. 위에서 Java로 작성한 RouterFunction을 Kotlin으로 재작성해보겠습니다.

```kotlin
    @Bean
    fun routerFunction(someHandler: SomeHandler) = router {
        accept(TEXT_HTML).nest {
            GET("/") { ok().contentType(TEXT_HTML).render("index") }
        }

        ("/api/v1/some" and accept(APPLICATION_JSON)).nest {
            GET("/", someHandler::getDataAll)
            GET("/{id}", someHandler::getData)
            POST("/{id}", someHandler::postData)
            DELETE("/{id}", someHandler::deleteData)
            PUT("/{id}", someHandler::putData)
        }
    }
```

Kotlin의 확장 기능을 통해서 보다 보기 편하게 RouterFunction이 개선되었습니다! 코드를 장황하게 만들던 의미없는 코드들이 많이 줄어들고 정말 API를 작성하는데 필요한 부분만 남아서 한눈에 알아보기에도 좋네요. 개인적으로 기존에 사용하던 어노테이션을 통해서 컨트롤러를 구현하는 것보다 이 코드가 더 깔끔한 것 같습니다.

### Model

RouterFunction을 사용하지 않고 기존의 어노테이션을 사용하더라도 작은 편한점이 하나 있습니다. 깨알 같은 변화이지만 가독성은 더 좋아지는 것 같네요.

**Java**

```java
@RequestMapping("/someUrl")
public String getSome(Model model) {
    model.addAttribute("name", "kwseo");
    model.addAttribute("items", service.findItems());
    return "somePage";
}
```

**Kotlin**

```kotlin
@RequestMapping("/someUrl")
fun good(model: Model): String {
    model["name"] = "kwseo"
    model["items"] = service.findItems()
    return "somePage"
}
```

## Bean
빈을 생성하기 위한 간단한 모듈도 제공합니다. 아래와 같이 간단한 표현으로 빈을 생성할 수 있지만, 지금까지 `@Bean`을 통해서 빈을 생성하여 ApplicationContext를 직접 다루지 않는 개발자에게는 별로 큰 메리트처럼 보이지는 않습니다. 

```kotlin
val someContext = beans {
    bean<SomeClass1>()
    bean<SomeClass2>()
    bean {
        // code ...
        SomeClass(ref())
    }
    
    profile("test") {
        bean<SomeClassForTest>()
    }
}
```

하지만 Spring boot 2와 함께 사용한다면 run할때 등록해주는 방법이 있습니다. Spring boot 2에서는 더 유연하게 main 함수를 작성할 수 있는 확장 모듈을 제공합니다.

```kotlin
@SpringBootApplication
class SomeWebApplication

fun main(args: Array<String>) {
    runApplication<SomeWebApplication>(*args) {
        addInitializers(someContext)
    }
}
```

## RestTemplate & WebClient
Spring에서 제공하는 웹 클라이언트 클래스인 RestTemplate을 사용한다면 응답 값의 데이터 타입이 조금이라도 복잡해지면 코드가 장황해집니다. `getForObject` 같은 간단한 메소드를 사용할 수 없고 아래의 코드와 같이 같은 `exchange`와 함께 `ParameterizedTypeReference<...>() {}`라는 정말 이름이 길어서 손가락을 아프게하는 클래스를 사용해야할 때가 있습니다. 이 역시 Kotlin으로 확장되어 손가락을 덜 아프게 해줍니다.

**Java**

```java
 ResponseEntity<List<SomeData>> response = restTemplate.exchange("/someUrl", HttpMethod.GET, httpEntity, new ParameterizedTypeReference<List<SomeData>>(){});
```

**Kotlin**

```kotlin
val response = restTemplate.getForEntity<List<SomeData>>("/someUrl")
``` 

그리고 Spring5부터는 AsyncRestTemplate deprecated되고 WebClient가 새롭게 등장하였습니다. 이도 역시 Kotlin의 뛰어난 타입추론 기능으로 인해서 불필요한 코드를 깨알같이 줄일 수 있습니다.

**Java**

```java
Flux<SomeData> users  = client.get().retrieve().bodyToFlux(SomeData.class)
```

**Kotlin**

```kotlin
val users : Flux<SomeData> = client.get().retrieve().bodyToFlux()
```

## Test

참고로 코틀린의 특성 덕분에 테스트를 수행하는 메소드의 이름을 보다 자유롭게 작성할 수 있습니다. 물론 무엇을 테스트하는 코드인지 주석을 달아주는 방법도 좋지만 테스트 메소드들을 한 곳에 모아서 볼때(예를 들면, mvn test 또는 Intellij에서 전체 테스트 코드를 실행했을 때) 메소드명만 봐도 어떤 역할을 하는지 쉽게 알 수 있을 것입니다.

```kotlin
    @Test
    fun `Some function should be successful when called`() {
        // ...
    }

    @Test
    fun `이 테스트는 abc에 대한 테스트이며 반드시 성공해야한다!`() {
        println("GOOD")
    }
```

---

이외에도 다른 Kotlin이 확장 모듈이 존재하지만 여기서는 이런 것도 있구나하고 간단히 소개하는 자리이기 때문에 전부 구구절절 설명하지는 않을 것입이다(사실, 아직 많지는 않아서 그냥 구글에서 몇 개 찾아보는 것이 더 효율적일 것입니다). 몇몇 Kotlin 확장 모듈 및 Kotlin이 가진 이점 덕분에 Spring 5에서는 보다 가독성이 좋은 깔끔한 코드를 작성할 수 있으며, Kotlin을 통해 보다 유연한 클래스 및 함수를 설계할 수 있을 것입니다. 또한, Kotlin의 가장 큰 장점 중 하나인 Null safety를 통해서 보다 견고한 애플리케이션을 개발하고, 불필요한 코드들을 제거한 간결한 코드 덕에 생산성도 향상될 것 입니다. 정말 깔끔하네요.
