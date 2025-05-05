**05-05-25**

O Guia Mostra uam comparação da aplicação class do spring bott vs micronaut.

## spring boot aplication class

```java

package example.micronaut;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

## micronaut aplication class

```java

package example.micronaut;

import io.micronaut.runtime.Micronaut;

public class Application {

    public static void main(String[] args) {
        Micronaut.run(Application.class, args);
    }
}

```


A sintaxe do micronaut é mais simples, não precisa de anotações como @SpringBootApplication, o micronaut já sabe que a classe é a aplicação principal.

## Manually define a bean - spring boot

## Implementação

```java

package example.micronaut;

public interface Greeter {
    String greet();
}

```

``` java

package example.micronaut;

public class HelloGreeter implements Greeter {
    @Override
    public String greet() {
        return "Hello";
    }
}

```
## Spring Boot



spring, @Configuration criar um bean

``` java

package example.micronaut;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class GreeterFactory {

    @Bean
    Greeter helloGreeter() {
        return new HelloGreeter();
    }
}
```

### Test

``` java

package example.micronaut;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
class GreeterTest {

    @Autowired
    Greeter greeter;

    @Test
    void helloGreeterIsInjectedAsBeanOfTypeGreeter() {
        assertNotNull(greeter);
        assertEquals("Hello", greeter.greet());
    }

}
```

1. **@SpringBootTest** - inicializa o contexto do spring e injeta o bean.
2. **@Autowired** - injeta o bean do spring.

## Micronaut @Factory

``` java

package example.micronaut;

import io.micronaut.context.annotation.Factory;
import io.micronaut.context.annotation.Bean;

@Factory
class GreeterFactory {
    @Bean
    Greeter helloGreeter() {
        return new HelloGreeter();
    }
}
```

1. **@Factory** - criar um bean do micronaut.

### Micronaut Test

``` java

package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@MicronautTest
class GreeterTest {

    @Inject
    Greeter greeter;

    @Test
    void helloGreeterIsInjectedAsBeanOfTypeGreeter() {
        assertNotNull(greeter);
        assertEquals("Hello", greeter.greet());
    }

}
```

1. **@MicronautTest** - inicializa o contexto do micronaut e injeta o bean.
2. **@Inject** - injeta o bean do micronaut.

## Conclusão

Esse guia ilustra que a Api @Configuration do spring e factory em ambos frameworks são muito similares, o que muda é a anotação e a forma de injetar o bean. de modo simples, vamos podemos utilizar o @Factory do micronaut para criar um bean e o @Inject para injetar o bean.


## Referências

- [Manually define a Bean](https://guides.micronaut.io/latest/spring-boot-to-micronaut-at-configuration-at-bean-at-factory-maven-java.html)


# mark a classe como um bean

## Criação da interface

``` java
package example.micronaut;

public interface Greeter {
    String greet();
}
```


## Spring boot

Nos spring utilizamos a anotação @Component para marcar a classe como um bean.

``` java

package example.micronaut;
import org.springframework.stereotype.Component;

@Component
public class HelloGreeter implements Greeter {
    @Override
    public String greet() {
        return "Hello";
    }
}
```

### Test

``` java

package example.micronaut;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
class GreeterTest {

    @Autowired
    Greeter greeter;

    @Test
    void helloGreeterIsInjectedAsBeanOfTypeGreeter() {
        assertNotNull(greeter);
        assertEquals("Hello", greeter.greet());
    }

}
```

## Micronaut

No micronaut utilizamos a anotação @Singleton para marcar a classe como um bean.

``` java

package example.micronaut;

import jakarta.inject.Singleton;

@Singleton
public class HelloGreeter implements Greeter {
    @Override
    public String greet() {
        return "Hello";
    }
}

```


1. **@Singleton** - marca a classe como um bean do micronaut, jakata.inject.Singleton

### Test

``` java

package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@MicronautTest
class GreeterTest {

    @Inject
    Greeter greeter;

    @Test
    void helloGreeterIsInjectedAsBeanOfTypeGreeter() {
        assertNotNull(greeter);
        assertEquals("Hello", greeter.greet());
    }

}
```

## Conclusão

O guia mostra a diferença entre a geração de beans do spring e do micronaut, enquanto o spring utilizar a anotação @Componente que é uma anotação constomizada para o spring, o micronaut utiliza a anotação @Singleton que é uma anotação do jakarta.inject.Singleton.

> O micronaut gerar is informações necessárias para preencher os pontos de injeção no momento da compilação.


> 	O Spring depende da varredura de classpath para encontrar classes anotadas com @Component. No aplicativo Spring Boot, HelloGreeter é detectado porque o aplicativo contém uma classe Applicationcom a @SpringBootApplication anotação. A @SpringBootApplication anotação aplica a @ComponentScan anotação, que instrui o Spring a varrer o pacote onde a Applicationclasse está localizada e seus subpacotes.


# Bulding Uris

é muito utis para construir uris.

## Spring Boot

``` java

package example.micronaut;

import org.junit.jupiter.api.Test;
import org.springframework.web.util.UriComponentsBuilder;

import static org.junit.jupiter.api.Assertions.assertEquals;

public class UriComponentsBuilderTest {

    @Test
    void youCanUseUriComponentsBuilderToBuildUris() {
        String isbn = "1680502395";
        assertEquals("/book/1680502395?lang=es", UriComponentsBuilder.fromUriString("/book")
                .path("/" + isbn)
                .queryParam("lang", "es")
                .build()
                .toUriString());
    }
}
```


## Micronaut

``` java

package example.micronaut;

import io.micronaut.http.uri.UriBuilder;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;

public class UriBuilderTest {

    @Test
    void youCanUseUriBuilderToBuildUris() {
        String isbn = "1680502395";
        assertEquals("/book/1680502395?lang=es", UriBuilder.of("/book")
                .path(isbn)
                .queryParam("lang", "es")
                .build()
                .toString());
    }
}
```

## Conclusão

A sintaxe do micronaut é mais simples, o nome da classe é mais simles.

# Run a spring bott application como uma aplicação micronaut

a dependencia `spring-boot`, permiter usar as tradicionais spring annotations.

# Micronaut data from a spring boot application

basta usar o micronaut spring boot starte para usar micronaut features com spring boot.

# Testing serialização spirng boot vs micronaunt framework

Esse guia compara como os teste serialização do spring e do micronaut


## Record

```java

package example.micronaut;

record SaasSubscription(Long id, String name, Integer cents) {
}

```


## micronaut

```java

package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

@Serdeable
record SaasSubscription(Long id, String name, Integer cents) {
}

```

1. **@Serdeable** - anota o record para ser serializado.

## json esperado

```json

{
  "id": 1,
  "name": "Micronaut",
  "cents": 1000
}
```


## Tests


### Spring Boot

```java

package example.micronaut;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.json.JsonTest;
import org.springframework.boot.test.json.JacksonTester;

import java.io.IOException;

import static org.assertj.core.api.Assertions.assertThat;

@JsonTest
class SaasSubscriptionJsonTest {

    @Autowired
    private JacksonTester<SaasSubscription> json;

    @Test
    void saasSubscriptionSerializationTest() throws IOException {
        SaasSubscription subscription = new SaasSubscription(99L, "Professional", 4900);
        assertThat(json.write(subscription)).isStrictlyEqualToJson("expected.json");
        assertThat(json.write(subscription)).hasJsonPathNumberValue("@.id");
        assertThat(json.write(subscription)).extractingJsonPathNumberValue("@.id")
                .isEqualTo(99);
        assertThat(json.write(subscription)).hasJsonPathStringValue("@.name");
        assertThat(json.write(subscription)).extractingJsonPathStringValue("@.name")
                .isEqualTo("Professional");
        assertThat(json.write(subscription)).hasJsonPathNumberValue("@.cents");
        assertThat(json.write(subscription)).extractingJsonPathNumberValue("@.cents")
                .isEqualTo(4900);
    }

    @Test
    void saasSubscriptionDeserializationTest() throws IOException {
        String expected = """
           {
               "id":100,
               "name": "Advanced",
               "cents":2900
           }
           """;
        assertThat(json.parse(expected))
                .isEqualTo(new SaasSubscription(100L, "Advanced", 2900));
        assertThat(json.parseObject(expected).id()).isEqualTo(100);
        assertThat(json.parseObject(expected).name()).isEqualTo("Advanced");
        assertThat(json.parseObject(expected).cents()).isEqualTo(2900);
    }
}
```

1. **@JsonTest** - é um auto-configures um ObjectMapper
2. **JacksonTester** - é um teste de unidade para serialização e desserialização de objetos JSON.



### Micronaut

A única personalização que você pode especificar com `@MicronautTest' é se deseja iniciar um servidor embarcado ou não.

```java
package example.micronaut;

import com.jayway.jsonpath.DocumentContext;
import com.jayway.jsonpath.JsonPath;
import io.micronaut.core.io.ResourceLoader;
import io.micronaut.json.JsonMapper;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.Reader;
import java.nio.charset.StandardCharsets;
import java.util.Optional;
import java.util.stream.Collectors;

import static org.assertj.core.api.Assertions.assertThat;

@MicronautTest(startApplication = false)
class SaasSubscriptionJsonTest {

    @Test
    void saasSubscriptionSerializationTest(JsonMapper json, ResourceLoader resourceLoader) throws IOException {
        SaasSubscription subscription = new SaasSubscription(99L, "Professional", 4900);
        String expected = getResourceAsString(resourceLoader, "expected.json");
        String result = json.writeValueAsString(subscription);
        assertThat(result).isEqualToIgnoringWhitespace(expected);
        DocumentContext documentContext = JsonPath.parse(result);
        Number id = documentContext.read("$.id");
        assertThat(id)
                .isNotNull()
                .isEqualTo(99);

        String name = documentContext.read("$.name");
        assertThat(name)
                .isNotNull()
                .isEqualTo("Professional");

        Number cents = documentContext.read("$.cents");
        assertThat(cents)
                .isNotNull()
                .isEqualTo(4900);
    }

    @Test
    void saasSubscriptionDeserializationTest(JsonMapper json) throws IOException {
        String expected = """
           {
               "id":100,
               "name": "Advanced",
               "cents":2900
           }
           """;
        assertThat(json.readValue(expected, SaasSubscription.class))
                .isEqualTo(new SaasSubscription(100L, "Advanced", 2900));
        assertThat(json.readValue(expected, SaasSubscription.class).id()).isEqualTo(100);
        assertThat(json.readValue(expected, SaasSubscription.class).name()).isEqualTo("Advanced");
        assertThat(json.readValue(expected, SaasSubscription.class).cents()).isEqualTo(2900);
    }

    private static String getResourceAsString(ResourceLoader resourceLoader, String resourceName) {
        return resourceLoader.getResourceAsStream(resourceName)
                .flatMap(stream -> {
                    try (InputStream is = stream;
                         Reader reader = new InputStreamReader(is, StandardCharsets.UTF_8);
                         BufferedReader bufferedReader = new BufferedReader(reader)) {
                        return Optional.of(bufferedReader.lines().collect(Collectors.joining("\n")));
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                    return Optional.empty();
                })
                .orElse("");
    }
}
```

> Cada @Test método é encapsulado em uma trasação que será encapsulado em uma trasação que será revertida. esse comportamento pode ser alterado utilizando o parametro transaction como false em @MicronautTest

1. **@MicronautTest** - para que o framework Micronaut inicialize o contexto da aplicação e o servidor embarcado.
2. por padrão, com o Junit5, os paâmentro s do método de teste serão resolvido para bens, se possível.


## Conclusão

A serialiazação no micronaut utilizar um biblioteca interna, para a leitura de json, pode utilizar [JsonPath](https://github.com/json-path/JsonPath), que é uma DSL para leitura de json.


# Implementango o get do spring boot no micronaut Rest api


## Controller

Vou criar uma tabela em markdown que mantém a mesma estrutura e conteúdo da tabela que você compartilhou. Aqui está:

## Tabela 1. Comparison between Spring Boot and Micronaut Framework

|  | Spring Boot | Micronaut |
|---|------------|-----------|
| Mark a bean as a controller | Annotate with `@RestController` and `@RequestMapping` | Annotate with `@Controller` |
| Identify a method as a GET endpoint | Annotate with `@GetMapping` | Annotate with `@Get` |
| Identify a method parameter as a path variable | Annotate with Spring's `@PathVariable` | Annotate with Micronaut's `@PathVariable` |
| Respond HTTP Responses with a status code and a body | Return a `ResponseEntity` | Return an `HttpResponse` |


>	Outra diferença importante é a visibilidade dos métodos do controlador. O Micronaut Framework não utiliza reflexão (o que resulta em melhor desempenho e melhor integração com tecnologias como GraalVM) . Portanto, ele exige que os métodos do controlador sejam públicos, protegidos ou privados (sem modificadores). Ao longo destes tutoriais, os métodos dos controladores Micronaut utilizam privados.


## Controller Spring Boot

```java
package example.micronaut;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController //<1>
@RequestMapping("/subscriptions") //<2>
class SaasSubscriptionController {

    @GetMapping("/{id}") //<3>
    private ResponseEntity<SaasSubscription> findById(@PathVariable Long id) { //<4>
        if (id.equals(99L)) {
            SaasSubscription subscription = new SaasSubscription(99L, "Advanced", 2900);
            return ResponseEntity.ok(subscription);
        }
        return ResponseEntity.notFound().build();
    }
}
```

1. **@RestController** - anota a classe como um controlador REST
2. **@RequestMapping** - identifica o path do controlador
3. **@GetMapping** - identifica o método como um endpoint GET
4. **@PathVariable** - identifica o parâmetro como um path variable

## Controller Micronaut

```java

package example.micronaut;

import io.micronaut.http.HttpResponse;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.PathVariable;

@Controller("/subscriptions")
class SaasSubscriptionController {

    @Get("/{id}")
    HttpResponse<SaasSubscription> findById(@PathVariable Long id) {
        if (id.equals(99L)) {
            SaasSubscription subscription = new SaasSubscription(99L, "Advanced", 2900);
            return HttpResponse.ok(subscription);
        }
        return HttpResponse.notFound();
    }
}
```

1. **@Controller** - anota a classe como um controlador REST
2. **@Get** - identifica o método como um endpoint GET
3. **@PathVariable** - identifica o parâmetro como um path variable
4. **HttpResponse** - o retorno do método é um HttpResponse, que é o retorno do micronaut

> 	The default HTTP Status code in a Micronaut controller method is 200. However, when a Micronaut controller’s method returns null, the application responds with a 404 status code.


``` java
package example.micronaut;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.PathVariable;

@Controller("/subscriptions")//<1>
class SaasSubscriptionController {

    @Get("/{id}")//<2>
    SaasSubscription findById(@PathVariable Long id) {//<3>
        if (id.equals(99L)) {
            return new SaasSubscription(99L, "Advanced", 2900);
        }
        return null;//<4>
    }
}
```

1. **@Controller** - anota a classe como um controlador REST
2. **@Get** - identifica o método como um endpoint GET
3. **@PathVariable** - identifica o parâmetro como um path variable
4. **null** - o retorno do método é null, o que resulta em um 404

O micronaut supoorta validação de rotas em tempo de compilação,

``` gradle
annotationProcessor("io.micronaut:micronaut-http-validation")
```

Por exemplo, se você substituir a @Get("/{id}")anotação por @Get("/{identifier}"), o aplicativo não será compilado.

## Testes

### Spring Boot

```java

package example.micronaut;

import com.jayway.jsonpath.DocumentContext;
import com.jayway.jsonpath.JsonPath;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class SaasSubscriptionControllerGetTest {

    @Autowired
    TestRestTemplate restTemplate;

    @Test
    void shouldReturnASaasSubscriptionWhenDataIsSaved() {
        ResponseEntity<String> response = restTemplate.getForEntity("/subscriptions/99", String.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);

        DocumentContext documentContext = JsonPath.parse(response.getBody());
        Number id = documentContext.read("$.id");
        assertThat(id).isNotNull();
        assertThat(id).isEqualTo(99);

        String name = documentContext.read("$.name");
        assertThat(name).isNotNull();
        assertThat(name).isEqualTo("Advanced");

        Integer cents = documentContext.read("$.cents");
        assertThat(cents).isEqualTo(2900);
    }

    @Test
    void shouldNotReturnASaasSubscriptionWithAnUnknownId() {
        ResponseEntity<String> response = restTemplate.getForEntity("/subscriptions/1000", String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
        assertThat(response.getBody()).isBlank();
    }
}
```

### Micronaut

```java

package example.micronaut;

import com.jayway.jsonpath.DocumentContext;
import com.jayway.jsonpath.JsonPath;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.AssertionsForClassTypes.catchThrowableOfType;

@MicronautTest //<1>
class SaasSubscriptionControllerGetTest {

    @Inject
    @Client("/")
    HttpClient httpClient;//<2>

    @Test
    void shouldReturnASaasSubscriptionWhenDataIsSaved() {
        BlockingHttpClient client = httpClient.toBlocking();
        HttpResponse<String> response = client.exchange("/subscriptions/99", String.class);
        assertThat(response.status().getCode()).isEqualTo(HttpStatus.OK.getCode());

        DocumentContext documentContext = JsonPath.parse(response.body());
        Number id = documentContext.read("$.id");
        assertThat(id).isNotNull();
        assertThat(id).isEqualTo(99);

        String name = documentContext.read("$.name");
        assertThat(name).isNotNull();
        assertThat(name).isEqualTo("Advanced");

        Integer cents = documentContext.read("$.cents");
        assertThat(cents).isEqualTo(2900);
    }

    @Test
    void shouldNotReturnASaasSubscriptionWithAnUnknownId() {
        BlockingHttpClient client = httpClient.toBlocking();
        HttpClientResponseException thrown = catchThrowableOfType(() -> // <3>
                client.exchange("/subscriptions/1000", String.class), HttpClientResponseException.class);
        assertThat(thrown.getStatus().getCode()).isEqualTo(HttpStatus.NOT_FOUND.getCode());
        assertThat(thrown.getResponse().getBody()).isEmpty();
    }
}
```

1. **@MicronautTest** - inicializa o contexto do micronaut e injeta o bean.
2. **HttpClient** - um bean que aponta para o servidor incorporado.
3. **HttpClientResponseException** - Quando o cliente HTTP recebe uma resposta com um código de status HTTP >= 400, ele lança um HttpClientResponseException. Você pode obter o status e o corpo da resposta a partir da exceção.


# [Post spring boot vs micronaut](https://guides.micronaut.io/latest/building-a-rest-api-spring-boot-vs-micronaut-post-gradle-java.html)

Micronaut Data é um data base access toolkit que usuar, Aheade compilationTime, para prè-computar consultas para interface de repositório, que não são então executadas por uma camada de tempo de execução final e leve.



## Controlador

é um endpoint Post que retorna a resposta 201.


## Tabela 1. Comparação entre Spring Boot e Micronaut Framework

|  | Bota de mola | Micronaut |
|---|------------|-----------|
| Identificar um método como um ponto de extremidade POST | `@PostMapping` | `@Post` |
| Ler e desserializar o corpo da solicitação para um parâmetro de método | `@RequestBody` | `@Body` |
| Fluid ApI para construir um URI | `UriComponentsBuilder` | `UriBuilder` |


## Spring Boot

```java
package example.micronaut;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.util.UriComponentsBuilder;

import java.net.URI;

@RestController
@RequestMapping("/subscriptions")
public class SaasSubscriptionPostController {

    private final SaasSubscriptionRepository repository;

    private SaasSubscriptionPostController(SaasSubscriptionRepository repository) {
        this.repository = repository;
    }

    @PostMapping
    private ResponseEntity<Void> createSaasSubscription(@RequestBody SaasSubscription newSaasSubscription,
                                                        UriComponentsBuilder ucb) {
        SaasSubscription subscription = repository.save(newSaasSubscription);
        URI locationOfSubscription = ucb
                .path("subscriptions/{id}")
                .buildAndExpand(subscription.id())
                .toUri();
        return ResponseEntity.created(locationOfSubscription).build();
    }
}
```


## Micronaut

```java
package example.micronaut;

import io.micronaut.http.HttpResponse;
import io.micronaut.http.annotation.Body;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Post;
import io.micronaut.http.uri.UriBuilder;

import java.net.URI;

@Controller("/subscriptions") //<1>
class SaasSubscriptionPostController {

    private final SaasSubscriptionRepository repository;

    SaasSubscriptionPostController(SaasSubscriptionRepository repository) { //<2>
        this.repository = repository;
    }

    @Post //<3>
    HttpResponse<?> createSaasSubscription(@Body SaasSubscription newSaasSubscription) { //<4>
        SaasSubscription savedSaasSubscription = repository.save(newSaasSubscription);
        URI locationOfNewSaasSubscription = UriBuilder.of("/subscriptions") //<5>
                .path(savedSaasSubscription.id().toString())
                .build();
        return HttpResponse.created(locationOfNewSaasSubscription);
    }
}

```

1. **@Controller** - anota a classe como um controlador REST para o path `/subscriptions`
2. **SaasSubscriptionPostController** - injeta o repositório no construtor
3. **@Post** - anota o método como um endpoint POST
4. **@Body** - anota o parâmetro como um body
5. **UriBuilder** - cria o URI para o novo recurso criado

## Testes

### Spring Boot

``` java
package example.micronaut;

import com.jayway.jsonpath.DocumentContext;
import com.jayway.jsonpath.JsonPath;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import java.net.URI;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class SaasSubscriptionPostControllerTest {

    @Autowired
    TestRestTemplate restTemplate;

    @Test
    void shouldCreateANewSaasSubscription() {
        SaasSubscription newSaasSubscription = new SaasSubscription(null, "Advanced", 2500);
        ResponseEntity<Void> createResponse = restTemplate.postForEntity("/subscriptions", newSaasSubscription, Void.class);
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        URI locationOfNewSaasSubscription = createResponse.getHeaders().getLocation();
        ResponseEntity<String> getResponse = restTemplate.getForEntity(locationOfNewSaasSubscription, String.class);
        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);

        DocumentContext documentContext = JsonPath.parse(getResponse.getBody());
        Number id = documentContext.read("$.id");
        assertThat(id).isNotNull();

        Integer cents = documentContext.read("$.cents");
        assertThat(cents).isEqualTo(2500);
    }
}
```


### Micronaut

```java

package example.micronaut;

import com.jayway.jsonpath.DocumentContext;
import com.jayway.jsonpath.JsonPath;
import io.micronaut.http.HttpHeaders;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

@MicronautTest
class SaasSubscriptionPostControllerTest {

    @Test
    void shouldCreateANewSaasSubscription(@Client("/") HttpClient httpClient) {
        BlockingHttpClient client = httpClient.toBlocking();
        SaasSubscription subscription = new SaasSubscription(100L, "Advanced", 2900);

        HttpResponse<Void> createResponse = client.exchange(HttpRequest.POST("/subscriptions", subscription), Void.class);
        assertThat(createResponse.getStatus().getCode()).isEqualTo(HttpStatus.CREATED.getCode());
        Optional<URI> locationOfNewSaasSubscriptionOptional = createResponse.getHeaders().get(HttpHeaders.LOCATION, URI.class);
        assertThat(locationOfNewSaasSubscriptionOptional).isPresent();

        URI locationOfNewSaasSubscription = locationOfNewSaasSubscriptionOptional.get();
        HttpResponse<String> getResponse = client.exchange(HttpRequest.GET(locationOfNewSaasSubscription), String.class);
        assertThat(getResponse.getStatus().getCode()).isEqualTo(HttpStatus.OK.getCode());

        DocumentContext documentContext = JsonPath.parse(getResponse.body());
        Number id = documentContext.read("$.id");
        assertThat(id).isNotNull();

        Integer cents = documentContext.read("$.cents");
        assertThat(cents).isEqualTo(2900);
    }
}
```

# Data spring vs micronaut

Micronaut data é um database acess toolkit que usa Ahead of time (AOT) para pre-computar as consultad para interfaces que são executadas em uma camada de tempo de execução leve e final.

## spring vs micronaut

### Sem modelo de execução

O spring data mantém um metamodelo de tempo de execução que usar refletion para modelar relacionamento entre entidades.

### Sem tradução de consultas

O Spring data uitlizar expressões regulares e correspondências de padrões em cominação com proxeisgerados em tempo de excecução para traduzir uma defnição de método em um interface java pra uma consulta em tempo de execução. No micronaut data, as consultas são traduzidas em tempo de compilação, o que resulta em um código mais rápido e leve.

### Sem porxies de reflexao ou tempo de execução
não usa proxies de refletion, observe que a implentação de suporte , por exemplo o hibernate, pode usar refletion

### segurança de tipo em tempo de compilação

O micronaut data verifica ativamente no momento da compilação se um método de repoistório pode ser implenetado.


## Dependências

``` gradle
annotationProcessor("io.micronaut.data:micronaut-data-processor")
implementation("io.micronaut.data:micronaut-data-jdbc")
runtimeOnly("com.h2database:h2")

```
## Entidades

### Spring Boot

``` java
package example.micronaut;

import org.springframework.data.annotation.Id;

record SaasSubscription(@Id Long id, String name, Integer cents) {
}

```

### Micronaut

``` java

package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.annotation.MappedEntity;

@Serdeable
@MappedEntity
record SaasSubscription(@Id Long id,
                        String name,
                        Integer cents) {
}
```

1. **@Serdeable** - anota o record para ser serializado.
2. **@MappedEntity** - anota a classe como uma entidade do micronaut data
3. **@Id** - anota o campo como um id da entidade

## Sql

é recomenda usar um gerenciado de eschema de banco de dados, como o liquibase ou o flyway, para gerenciar o esquema do banco de dados.

``` sql
CREATE TABLE IF NOT EXISTS saas_subscription
(
    id     BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name  VARCHAR(255) NOT NULL,
    cents NUMBER NOT NULL DEFAULT 0
);

INSERT INTO saas_subscription(id, name, cents) VALUES (99, 'Advanced', 2900);

```


## Repositório

### Spring Boot

``` java
package example.micronaut;

import org.springframework.data.repository.CrudRepository;

interface SaasSubscriptionRepository extends CrudRepository<SaasSubscription, Long> {
}
```

### Micronaut

``` java
package example.micronaut;

import io.micronaut.data.jdbc.annotation.JdbcRepository;
import io.micronaut.data.model.query.builder.sql.Dialect;
import io.micronaut.data.repository.CrudRepository;

@JdbcRepository(dialect = Dialect.H2)
interface SaasSubscriptionRepository extends CrudRepository<SaasSubscription, Long> {
}
```

1. **@JdbcRepository** - anota a interface como um repositório do micronaut data
2. **Dialect** - define o dialeto do banco de dados, nesse caso o H2
3. **CrudRepository** - é uma interface do micronaut data que estende você habilita a geração automática de operações CRUD (Criar, Ler, Atualizar, Excluir).


## Testes

``` java

package example.micronaut;

import com.jayway.jsonpath.DocumentContext;
import com.jayway.jsonpath.JsonPath;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.AssertionsForClassTypes.catchThrowableOfType;
import io.micronaut.test.annotation.Sql;

@Sql(value = {"classpath:schema.sql", "classpath:data.sql"}) //<1>
@MicronautTest //<2>
class SaasSubscriptionControllerGetTest {

    @Inject
    @Client("/")
    HttpClient httpClient; //<3>

    @Test
    void shouldReturnASaasSubscriptionWhenDataIsSaved() {
        BlockingHttpClient client = httpClient.toBlocking();
        HttpResponse<String> response = client.exchange("/subscriptions/99", String.class);
        assertThat(response.status().getCode()).isEqualTo(HttpStatus.OK.getCode());

        DocumentContext documentContext = JsonPath.parse(response.body());
        Number id = documentContext.read("$.id");
        assertThat(id).isNotNull();
        assertThat(id).isEqualTo(99);

        String name = documentContext.read("$.name");
        assertThat(name).isNotNull();
        assertThat(name).isEqualTo("Advanced");

        Integer cents = documentContext.read("$.cents");
        assertThat(cents).isEqualTo(2900);
    }

    @Test
    void shouldNotReturnASaasSubscriptionWithAnUnknownId() {
        BlockingHttpClient client = httpClient.toBlocking();
        HttpClientResponseException thrown = catchThrowableOfType(() ->
		// <4>
                client.exchange("/subscriptions/1000", String.class), HttpClientResponseException.class);
        assertThat(thrown.getStatus().getCode()).isEqualTo(HttpStatus.NOT_FOUND.getCode());
        assertThat(thrown.getResponse().getBody()).isEmpty();
    }
}

```

1. **@Sql** - anota o teste para executar o script SQL antes de executar o teste
2. **@MicronautTest** - inicializa o contexto do micronaut e injeta o bean.
3. **HttpClient** - um bean que aponta para o servidor incorporado.
4. **HttpClientResponseException** - Quando o cliente HTTP recebe uma resposta com um código de status HTTP >= 400, ele lança um HttpClientResponseException. Você pode obter o status e o corpo da resposta a partir da exceção.


Adicionar uma camada de persistência é fácil em ambas as estruturas, e a API é quase idêntica. No entanto, a abordagem sem reflexão e em tempo de compilação da Micronaut Data resulta em melhor desempenho, rastreamentos de pilha menores e consumo de memória reduzido.


## Referências
[Dados - Spring Boot vs Micronaut Framework - Construindo uma API REST](https://guides.micronaut.io/latest/building-a-rest-api-spring-boot-vs-micronaut-data-gradle-java.html)


# [Paginated List - Spring Boot vs Micronaut Framework - Building a Rest API](https://guides.micronaut.io/latest/building-a-rest-api-spring-boot-vs-micronaut-get-list-gradle-java.html)

## Repositorio



| | Spring Data | Micronaut Data |
|---|---|---|
| Created, Read, Update, Delegate Operations | `CrudRepository` | `CrudRepository` |
| Paginated Operations | `PagingAndSortingRepository` | `PageableRepository` |

*Tabela 1. Comparação entre Spring Boot e Micronaut Framework*

``` java
package example.micronaut;

import io.micronaut.data.jdbc.annotation.JdbcRepository;
import io.micronaut.data.model.query.builder.sql.Dialect;
import io.micronaut.data.repository.PageableRepository;

@JdbcRepository(dialect = Dialect.H2)
interface SaasSubscriptionRepository extends PageableRepository<SaasSubscription, Long> {
}
```

``` java
package example.micronaut;

import io.micronaut.data.model.Page;
import io.micronaut.data.model.Pageable;
import io.micronaut.data.model.Sort;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

@Controller("/subscriptions")
class SaasSubscriptionGetListController {

    private static final Sort CENTS = Sort.of(Sort.Order.asc("cents"));
    private final SaasSubscriptionRepository repository;

    SaasSubscriptionGetListController(SaasSubscriptionRepository repository) {
        this.repository = repository;
    }

    @Get
    Iterable<SaasSubscription> findAll(Pageable pageable) {
        Page<SaasSubscription> page = repository.findAll(pageable.getSort().isSorted()
                ? pageable
                : Pageable.from(pageable.getNumber(), pageable.getSize(), CENTS)
        );
        return page.getContent();
    }
}
```

1. **@Controller** - anota a classe como um controlador REST para o path `/subscriptions`
2. **SaasSubscriptionRepository** - injeta o repositório no construtor
3. **@Get** - anota o método como um endpoint GET
4. **Pageable** - inclui a capacidade de especificar requisitos de paginação. Você pode vinculá-lo como um parâmetro de método do controlador.
5. **PageableRepository** provem a findAll meto que tem um pageble como parametro.


## Testes

``` java

package example.micronaut;

import com.jayway.jsonpath.DocumentContext;
import com.jayway.jsonpath.JsonPath;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.uri.UriBuilder;
import io.micronaut.test.annotation.Sql;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import net.minidev.json.JSONArray;
import org.junit.jupiter.api.Test;

import java.net.URI;

import static org.assertj.core.api.Assertions.assertThat;

@Sql(value = {"classpath:schema.sql", "classpath:data.sql"})
@MicronautTest
class SaasSubscriptionGetListControllerTest {

    @Inject
    @Client("/")
    HttpClient httpClient;

    @Test
    void shouldReturnASortedPageOfSaasSubscriptions() {
        BlockingHttpClient client = httpClient.toBlocking();
        URI uri = UriBuilder.of("/subscriptions")
                .queryParam("page", 0)
                .queryParam("size", 1)
                .queryParam("sort", "cents,desc")
                .build();
        HttpResponse<String> response = client.exchange(HttpRequest.GET(uri), String.class);
        assertThat(response.status().getCode()).isEqualTo(HttpStatus.OK.getCode());

        DocumentContext documentContext = JsonPath.parse(response.body());
        JSONArray page = documentContext.read("$[*]");
        assertThat(page).hasSize(1);

        Integer cents = documentContext.read("$[0].cents");
        assertThat(cents).isEqualTo(4900);
    }

    @Test
    void shouldReturnAPageOfSaasSubscriptions() {
        BlockingHttpClient client = httpClient.toBlocking();
        URI uri = UriBuilder.of("/subscriptions")
                .queryParam("page", 0)
                .queryParam("size", 1)
                .build();
        HttpResponse<String> response = client.exchange(HttpRequest.GET(uri), String.class);
        assertThat(response.status().getCode()).isEqualTo(HttpStatus.OK.getCode());

        DocumentContext documentContext = JsonPath.parse(response.body());
        JSONArray page = documentContext.read("$[*]");
        assertThat(page.size()).isEqualTo(1);
    }

    @Test
    void shouldReturnAllSaasSubscriptionsWhenListIsRequested() {
        BlockingHttpClient client = httpClient.toBlocking();
        HttpResponse<String> response = client.exchange("/subscriptions", String.class);
        assertThat(response.status().getCode()).isEqualTo(HttpStatus.OK.getCode());

        DocumentContext documentContext = JsonPath.parse(response.body());
        int saasSubscriptionCount = documentContext.read("$.length()");
        assertThat(saasSubscriptionCount).isEqualTo(3);

        JSONArray ids = documentContext.read("$..id");
        assertThat(ids).containsExactlyInAnyOrder(99, 100, 101);

        JSONArray cents = documentContext.read("$..cents");
        assertThat(cents).containsExactlyInAnyOrder(1400, 2900, 4900);
    }
}
```

## conclusão

e bem parecido com o spring boot.
