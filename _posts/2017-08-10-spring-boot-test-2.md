---
layout: post
title: Spring Boot에서 테스트를 - 2
author: kwSeo
tags: [JAVA, Spring, SpringBoot]
---


# Spring Boot에서 테스트를 - 2

이전 문서에서는 Spring Boot에 새롭게 추가된 `spring-boot-test-starter`에 포함되어있는 모듈들과 기본적인 Spring Boot 테스트 모듈에 대해서 알아보았습니다. 이번에는 좀 더 세부적으로 들어가서 JSON 테스트, WebMvc 테스트, JPA 테스트, JDBC 테스트, Mongo 테스트, RestClient 테스트에 대해서 알아볼 것 입니다.

## @JsonTest

`@JsonTest` 어노테이션을 사용하면 보다 편하게 JSON serialization과 deserialization을 테스트해볼 수 있습니다. `@JsonTest` 어노테이션은 `ObjectMapper`와 `@JsonComponent` 빈을 포함한 Jackson의 테스트를 위한 모듈들을 자동으로 설정합니다.
테스트를 위한 빈으로 `JacksonTester`, `GsonTester`, `BasicJsonTester` 등이 있습니다. 이를 주입받아서 사용하면 보다 편리하게 JSON을 테스트해볼 수 있습니다. 그리고 Assertj는 JSON을 위한 기능들을 제공합니다(JSONassert, JsonPath를 기반으로한). 아래는 JSON serialize와 deserialize를 테스트하는 예제입니다.

```java
@RunWith(SpringRunner.class)
@JsonTest
public class ArticleJsonTest {
    @Autowired
    private JacksonTester<Article> json;

    @Test
    public void testSerialize() throws IOException {
        Article article = new Article(
                1,
                "kwseo",
                "good",
                "good article",
                Timestamp.valueOf((LocalDateTime.now())));

        // assertThat(json.write(article)).isEqualToJson("expected.json");  직접 파일과 비교
        assertThat(json.write(article)).hasJsonPathStringValue("@.author");
        assertThat(json.write(article))
                .extractingJsonPathStringValue("@.title")
                .isEqualTo("good");
    }

    @Test
    public void testDeserialize() throws IOException {
        Article article = new Article(
                1,
                "kwseo",
                "good",
                "good article",
                new Timestamp(1499655600000L));
        String jsonString = "{\"id\": 1, \"author\": \"kwseo\", \"title\": \"good\", \"content\": \"good article\", \"createdDate\": 1499655600000}";

        assertThat(json.parse(jsonString)).isEqualTo(article);
        assertThat(json.parseObject(jsonString).getAuthor()).isEqualTo("kwseo");
    }
}
```

## @WebMvcTest
이전 문서에서 client-side에서 API를 테스트하는 `TestRestTemplate`을 살펴봤다면, 이번에는 server-side에서 API를 테스트하는 `@WebMvcTest` 어노테이션에 대해서 알아볼 것 입니다. 해당 어노테이션은 기존에 `spring-test`에서 컨트롤러를 테스트할 때 많이 사용하던 `MockMvc`에 관한 설정을 자동으로 수행해주는 어노테이션입니다. `@WebMvcTest` 어노테이션을 사용하면 테스트에 사용할 `@Controller` 클래스와 `@ControllerAdvice`, `@JsonComponent`, `@Filter`, `WebMvcConfigurer`, `HandlerMethodArgumentResolver` 등을 스캔합니다. 그리고 `MockMvc`를 자동으로 설정하여 빈으로 등록합니다.

```java
@RunWith(SpringRunner.class)
@WebMvcTest(ArticleApiController.class)
public class ArticleApiControllerTest {
    @Autowired
    private MockMvc mvc;
    @MockBean
    private ArticleService articleService;

    @Test
    public void testGetArticles() throws Exception {
        List<Article> articles = asList(
                new Article(1, "kwseo", "good", "good content", now()),
                new Article(2, "kwseo", "haha", "good haha", now()));

        given(articleService.findFromDB(eq("kwseo"))).willReturn(articles);

        mvc.perform(get("/api/articles?author=kwseo"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("@[*].author", containsInAnyOrder("kwseo", "kwseo")));
    }

    private Timestamp now() {
        return Timestamp.valueOf(LocalDateTime.now());
    }
}
```

### Async Web Test

컨트롤러에서 `Future`나 `DeferredResult`의 객체를 반환하면 HTTP 요청과 응답은 비동기로 동작합니다. 기존과 다른 방식으로 동작하기에 `MockMvc`로 테스트 방법도 약간의 변화가 필요합니다. 


```java
    ...
    @Test
    public void testGetArticle() throws Exception {
        Article expected = new Article(1, "kwseo", "good", "good content", now());

        given(articleService.findOneFromRemote(eq(1))).willReturn(expected);

        MvcResult result = mvc.perform(get("/api/articles/1")).andReturn();
        mvc.perform(asyncDispatch(result))      // asyncDispatch 필요
            .andExpect(status().isOk())
            .andExpect(jsonPath("@.id").value(1));
    }
    ...
```

위 코드처럼 `MockMvc`로 요청을 한 뒤 `MvcResult`로 받어서 `asyncDispatch`로 감싸줄 필요가 있습니다.

## @DataJpaTest

Spring Data JPA를 테스트하고자 한다면 `@DataJpaTest` 기능을 사용해볼 수 있습니다. 이 어노테이션과 함께 테스트를 수행하면 기본적으로 in-memory embedded database를 생성하고 `@Entity` 클래스를 스캔합니다. 일반적인 다른 컴포넌트들은 스캔하지 않습니다. 
참고로 `@DataJpaTest`는 `@Transactional` 어노테이션을 포함하고 있습니다. 그래서 테스트가 완료되면 자동으로 롤백하을 하기 위해서 직접 `@Transactional` 어노테이션을 달아줄 필요가 없습니다. 그런데 만약 `@Transactional` 기능이 필요하지 않다면 아래와 같이 줄 수 있습니다.

```java
@RunWith(SpringRunner.class)
@DataJpaTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public class SomejpaTest {
    ...
}
```

`@DataJpaTest` 기능을 사용하면 `@Entity`를 스캔하고 repository를 설정하는 것 이외에도 테스트를 위한 `TestEntityManager`라는 빈이 생성됩니다. 이 빈을 사용해서 테스트에 이용한 데이터를 정의할 수 있습니다. 아래는 `@DataJpaTest`를 사용하여 테스트를 수행하는 예제입니다.

```java
@RunWith(SpringRunner.class)
@DataJpaTest
public class ArticleDaoTest {
    @Autowired
    private TestEntityManager entityManager;
    @Autowired
    private ArticleDao articleDao;

    @Test
    public void test() {
        Article articleByKwseo = new Article(1, "kwseo", "good", "hello", Timestamp.valueOf(LocalDateTime.now()));
        Article articleByKim = new Article(2, "kim", "good", "hello", Timestamp.valueOf(LocalDateTime.now()));
        entityManager.persist(articleByKwseo);
        entityManager.persist(articleByKim);


        List<Article> articles = articleDao.findByAuthor("kwseo");
        assertThat(articles)
                .isNotEmpty()
                .hasSize(1)
                .contains(articleByKwseo)
                .doesNotContain(articleByKim);
    }
}
```

만약 테스트에 in-memory embedded database를 사용하지 않고 real database를 사용하고자 하는 경우, `@AutoConfigureTestDatabase` 어노테이션을 사용하면 손쉽게 설정할 수 있습니다.

```java
@RunWith(SpringRunner.class)
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
public class SomeJpaTest {
    ...
}
```


## @JdbcTest

Spring Data JPA를 사용하지 않더라도 데이터베이스 테스트를 해볼 수 있습니다. `@JdbcTest`는 `@DataJpaTest`와 비슷한 설정을 수행하지만 순수 JDBC 테스트를 준비합니다. `@JdbcTest` 어노테이션을 사용하면 마찬가지로 im-memory embedded database가 설정되며, 테스트를 위한 `JdbcTemplate`이 생성됩니다.


## @DataMongoTest

최근 점점 많은 인기를 얻고 잇는 NoSQL DB인 MongoDB에 대해서도 편리한 테스트 기능을 제공합니다. `@DataMongoTest` 어노테이션이 이를 위한 기능을 제공하며 설정하는 내용은 `@DatajpaTest`와 유사합니다. 위에 다른 데이터 테스트 모듈과 유사하게 im-memory embedded MongoDB를 사용하지만, `@DataMongoTest`는 `@Entity`가 아닌 `@Document`를 스캔하며 `MongoTemplate`을 생성합니다.

```java
@RunWith(SpringRunner.class)
@DataMongoTest
public class SomeMongoTest {
    @Autowired 
    private MongoTemplate mongoTemplate;
    ...
}
```

만일 in-memory embedded MongoDB를 사용하는 것은 원하지 않고 외부에 직접 구축한 MongoDB을 사용하고자 한다면 아래와 같이 속성을 추가하면 됩니다.

```java
@DataMongoTest(excludeAutoConfiguration = EmbeddedMongoAutoConfiguration.class)
```

## @RestClientTest

`@RestClientTest` 기능은 자신이 서버 입장이 아니라 클라이언트 입장이 되는 코드를 테스트할때 유용합니다. 예를 들면, Apache HttpClient나 Spring의 RestTemplate을 사용하여 외부 서버에 웹 요청을 보내는 경우가 있습니다. `@RestClientTest`는 요청에 반응하는 가상의 Mock 서버를 만든다고 생각하면 됩니다. 내부 코드에서 웹 요청이 발생할 경우 `@RestClientTest`로 인해서 생성된 가상의 서버가 응답을 해줍니다. 물론 그 가상의 서버가 어떤식으로 응답을 할지 정의할 수 있습니다. 이를 사용하면 보다 `RestTemplate` 같은 객체를 Mock 객체로 바꿔서 테스트하는 것보다 리얼 환경에 가깝게 단위 테스트를 수행할 수 있습니다. 이 기능을 사용하면 자동으로 `MockRestServiceServer`라는 빈이 생성되며 이를 이용하면 손쉽게 요청과 응답에 대한 설정을 할 수 있습니다.

```java
@RunWith(SpringRunner.class)
@RestClientTest(ArticleServiceImpl.class)
public class ArticleServiceImplWithRestClientTest {
    @MockBean
    private ArticleDao dao;
    @Autowired
    private ArticleServiceImpl service;
    @Autowired
    private MockRestServiceServer server;

    @Test
    public void testGetFindOneFromRemote() throws Exception {
        String articleJson = "{ \"id\": 1, \"author\": \"kwseo\", \"title\": \"gogogo\", \"content\": \"good\", \"date\": 1502322765 }";

        server.expect(requestTo("http://sample.com/some/articles/1"))
            .andRespond(withSuccess(articleJson, MediaType.APPLICATION_JSON));

        Article article = service.findOneFromRemote(1);
        assertThat(article.getId()).isEqualTo(1);
    }
}
```
