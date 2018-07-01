---
layout: post
title: Spring Boot에서 테스트를 - 1
author: kwSeo
tags: [JAVA, Spring, SpringBoot]
---

# Spring Boot에서 테스트를 - 1

Spring Boot는 애플리케이션을 테스트하기 위한 많은 유틸 어노테이션을 제공합니다. Spring boot에서 테스트 모듈dms spring-boot-test와 spring-boot-test-autoconfigure가 존재합니다.
대부분의 경우는 spring-boot-starter-test만으로도 충분합니다. spring-boot-starter-test는 spring boot의 테스트에 사용되는 Starter 패키지입니다.
spring-boot-starter-test는 JUnit는 물론이고, AssertJ, Hamcrest를 포함한 여러 유용한 라이브러리를 포함하고 있습니다.

## 포함되어있는 주요 라이브러리들

기존 Spring framework에서 사용하던 spring-test 이외에도 여러 유용한 라이브러리를 포함하고 있습니다.
가지고 있는 라이브러리에는 Mocktio도 포함되어있는데 기본적으로 Mocktio 1.x 버전을 사용합니다. 하지만 원한다면 2.x 버전를 사용하는 것도 가능합니다.

- JUnit
- Spring Test & Spring Boot Test
- AssertJ
- Hamcrest
- Mockito
- JSONassert
- JsonPath

## @SpringBootTest 어노테이션

spring-boot-test는 @SpringBootTest라는 어노테이션을 제공합니다. 이 어노테이션을 사용하면 테스트에 사용할 ApplicationContext를 쉽게 생성하고 조작할 수 있습니다.  기존 spring-test에서 사용하던 @ContextConfiguration의 발전된 기능이라고 할 수 있습니다. 
@SpringBootTest는 매우 다양한 기능을 제공합니다. 전체 빈 중 특정 빈을 선택하여 생성한다던지, 특정 빈을 Mock으로 대체한다던지, 테스트에 사용할 프로퍼티 파일을 선택하거나 특정 속성만 추가한다던지, 특정 Configuration을 선택하여 설정할 수도 있습니다. 또한, 주요 기능으로 테스트 웹 환경을 자동으로 설정해주는 기능이 있습니다. 
앞에서 언급한 다양한 기능들을 사용하기 위해서 첫 번째로 가장 중요한 것은 @SpringBootTest 기능은 반드시 @RunWith(SpringRunner.class)와 함께 사용해야 된다는 것입니다.

### Bean

`@SpringBootTest` 어노테이션을 사용하면 테스트에 사용할 빈을 아주 손쉽게 생성할 수 있습니다. `@SpringBootTest` 어노테이션은 classes라는 속성을 제공합니다. 해당 속성을 통해서 빈을 생성할 클래스들을 지정할 수 있습니다. classes 속성에 `@Configuration` 어노테이션을 사용하는 클래스가 있다면 내부에서 `@Bean` 어노테이션을 통해서 생성되는 빈도 모두 등록이 됩니다. 만일 classes 속성을 통해서 클래스를 지정하지 않으면 애플리케이션 상에 정의된 모든 빈을 생성합니다.

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = {ArticleServiceImpl.class, CommonConfig.class})
public class SomeClassTest {
    // Service로서 등록된 빈
    @Autowired
    private ArticleServiceImpl articleServiceImpl;
    // CommonConfig에서 생성되는 빈
    @Autowired
    private RestTemplate restTemplate;
}
```

### TestConfiguration 

기존에 정의했던 Configuration을 커스터마이징하고 싶은 경우 TestConfiguration 기능을 사용할 수 있습니다. TestConfiguration은 ComponentScan 과정에서 생성될 것이며 해당 자신이 속한 테스트가 실행될때 정의된 빈을 생성하여 등록할 것입니다.

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class TestConfigArticleServiceImplTest {
    @MockBean
    private ArticleDao articleDao;
    @Autowired
    private RestTemplate restTemplate;
    @Autowired
    private ArticleServiceImpl articleServiceImpl;

    @Test
    public void test() {
        String good = restTemplate.getForObject("test", String.class);
        assertThat(good).isEqualTo("Good");
    }

    @TestConfiguration
    public static class TestConfig {
        @Bean
        public RestTemplate restTemplate() {
            return new RestTemplate() {
                @Override
                public <T> T getForObject(String url, Class<T> responseType, Object... uriVariables) throws RestClientException {
                    System.out.println("Good");
                    if (responseType == String.class) {
                        return (T) "Good";
                    } else {
                        throw new IllegalArgumentException();
                    }
                }
            };
        }
    }
}
```

ComponentScan을 통해서 감지되기 때문에 만일 `@SpringBootTest`의 classes 속성을 이용하여 특정 클래스만을 지정했을 경우에는 TestConfiguation은 감지되지 않습니다. 그런 경우 classes 속성에 직접 TestConfiguration을 추가해주어야 합니다. 하지만 더 좋은 방법은 `@Import` 어노테이션을 사용하는 것입니다. `@Import` 어노테이션을 통해서 직접 사용할 TestConfiguration을 명시할 수 있으며 특정 테스트 클래스의 내부 클래스가 아닌 별도의 클래스로 분리하여 여러 테스트에서 공유할 수도 있습니다.

```java
@TestConfiguration
public class TestConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate() {
            @Override
            public <T> T getForObject(String url, Class<T> responseType, Object... uriVariables) throws RestClientException {
                System.out.println("Good");
                if (responseType == String.class) {
                    return (T) "Good";
                } else {
                    throw new IllegalArgumentException();
                }
            }
        };
    }
}
```

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = ArticleServiceImpl.class)
@Import(TestConfig.class)
public class TestConfigArticleServiceImplTest {
    @MockBean
    private ArticleDao articleDao;
    @Autowired
    private RestTemplate restTemplate;
    @Autowired
    private ArticleServiceImpl articleServiceImpl;

    @Test
    public void test() {
        String good = restTemplate.getForObject("test", String.class);
        assertThat(good).isEqualTo("Good");
    }
}
```

### MockBean and SpyBean

spring-boot-test 패키지는 Mockito를 포함하고 있기 때문에 기존에 사용하던 방식대로 Mock 객체를 생성해서 테스트하는 방법도 있지만, spring-boot-test에서는 새로운 방법도 제공하고 있습니다. `@MockBean` 어노테이션을 사용해서 이름 그대로 Mock 객체를 빈으로써 등록할 수 있습니다. 그렇기 때문에 만일 `@MockBean`으로 선언된 빈을 주입받는다면(`@Autowired` 같은 어노테이션 등을 통해서) Spring의 ApplicationContext는 Mock 객체를 주입해줍니다.
새롭게 `@MockBean`을 선언하면 Mock 객체를 빈으로써 등록을 하지만, 만일 `@MockBean`으로 선언한 객체와 같은 이름과 타입으로 이미 빈으로 등록되어있다면 해당 빈은 선언한 Mock 빈으로 대체됩니다.

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = ArticleServiceImpl.class)
public class ArticleServiceImplTest {
    @MockBean
    private RestTemplate restTemplate;
    @MockBean
    private ArticleDao articleDao;
    @Autowired
    private ArticleServiceImpl articleServiceImpl;

    @Test
    public void testFindFromDB() {
        List<Article> expected = Arrays.asList(
                new Article(0, "author1", "title1", "content1", Timestamp.valueOf(LocalDateTime.now())),
                new Article(1, "author2", "title2", "content2", Timestamp.valueOf(LocalDateTime.now())));

        given(articleDao.findAll()).willReturn(expected);

        List<Article> articles = articleServiceImpl.findFromDB();
        assertThat(articles).isEqualTo(expected);
    }
}
```

### Properties 

Spring Boot는 기본적으로 클래스 경로상의 `application.properties`(또는 `application.yml`)를 통해 애플리케이션 설정을 수행합니다. 하지만 테스트 중에는 설정이 기존과 달라질 필요가 있는 경우가 많은데 이때를 위한 기능을 `SpringBootTest`에서 제공하고 있습니다. `SpringBootTest`는 properties라는 속성이 존재합니다. 이 속성을 이용해서 별도의 테스트를 위한 `application.properties`(또는 `application.yml`)을 지정할 수 있습니다.

```java
@RunWith(SpringBoot.class)
@SpringBootTest(properties = "classpath:application-test.yml")
public class SomeTest {
    ...
}
```

### Web Environment test

앞에서 언급했듯이 @SpringBootTest 어노테이션을 사용하면 손쉽게 웹 테스트 환경을 구성할 수 있습니다. @SpringBootTest의 webEnvironment 파라미터를 이용해서 손쉽게 웹 테스트 환경을 선택할 수 있습니다. 제공하는 설정 값은 아래와 같습니다.

- MOCK
    - WebApplicationContext를 로드하며 내장된 서블릿 컨테이너가 아닌 Mock 서블릿을 제공합니다. @AutoConfigureMockMvc 어노테이션을 함께 사용하면 별다른 설정없이 간편하게 MockMvc를 사용한 테스트를 진행할 수 있습니다.
- RANDOM_PORT
    - EmbeddedWebApplicationContext를 로드하며 실제 서블릿 환경을 구성합니다. 생성된 서블릿 컨테이너는 임의의 포트는 listen합니다.
- DEFINED_PORT
    - RAMDOM_PORT와 동일하게 실제 서블릿 환경을 구성하지만, 포트는 애플리케이션 프로퍼티에서 지정한 포트를 listen합니다(application.properties 또는 application.yml에서 지정한 포트)
- NONE
    - 일반적인 ApplicationContext를 로드하며 아무런 서블릿 환경을 구성하지 않습니다.

### TestRestTemplate

`@SpringBootTest`와 `TestRestTemplate`을 사용한다면 편리하게 웹 통합 테스트를 할 수 있다. `TestRestTemplate`은 이름에서 알 수 있듯이 `RestTemplate`의 테스트를 위한 버전입니다. `@SpringBootTest`에서 Web Environment 설정을 하였다면 `TestRestTemplate`은 그에 맞춰서 자동으로 설정되어 빈이 생성됩니다.

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class RestApiTest {
    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    public void test() {
        ResponseEntity<Article> response = restTemplate.getForEntity("/api/articles/1", Article.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isNotNull();
        ...
    }
}
```

기존에 컨트롤러를 테스트하는데 많이 사용되던 `MockMvc`와 어떤 차이가 있는지 궁금할 것입니다. 가장 큰 차이점이라면 Servlet Container를 사용하느냐 안하느냐의 차이입니다. `MockMvc`는 Servlet Container를 생성하지 않습니다. 반면, `@SpringBootTest`와 `TestRestTemplate`은 Servlet Container를 사용합니다. 그래서 마치 실제 서버가 동작하는 것처럼(물론 몇몇 빈은 Mock 객체로 대체될 수는 있습니다) 테스트를 수행할 수 있습니다. 또한, 테스트를 하는 관점도 서로 다릅니다. `MockMvc`는 서버 입장에서 구현한 API를 통해 비지니스 로직이 문제없이 수행되는지 테스트를 할 수 있다면, `TestRestTemplate`은 클라이언트 입장에서 `RestTemplate`을 사용하듯이 테스트를 수행할 수 있습니다.

### 트랜젝션

이때 주의해야할 점이 있는데 바로 @Transactional 어노테이션입니다. spring-boot-test는 그저 spring-test를 확장한 것이기 때문에 @Test 어노테이션과 함께 @Transactional 어노테이션을 함께 사용하면 테스트가 끝날때 rollback됩니다. 하지만 RANDOM_PORT나 DEFINED_PORT로 테스트를 설정하면 실제 테스트 서버는 별도의 스레드에서 수행되기 때문에 rollback이 이루어지지 않습니다.

### ApplicationContext 캐시

참고로 @SpringBootTest 기능으로 인해서 생성된 ApplicationContext를 캐시됩니다. 만약에 @SpringBootTest의 설정이 동일하다면 동일한 ApplicationContext를 사용하게 됩니다.

## 다음 문서에서는

`spring-boot-test`는 테스트를 효율적이게 할 수 있는 많은 도구들을 제공합니다. 지금까지는 테스트를 하기위한 기초적인 부분들을 정리하였습니다. 다음 문서에서는 아래의 나열한 기능들에 대해서 살펴보겠습니다.

- @WebMvcTest
- @DataJpaTest
- @JdbcTest
- @DataMongoTest
- @JsonTest
- @RestClientTest


