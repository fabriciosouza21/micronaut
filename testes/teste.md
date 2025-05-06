**06-05-25**

# [Usando o teste Micronaut REST-Assured em um aplicativo Micronaut](https://guides.micronaut.io/latest/micronaut-rest-assured-maven-java.html)

O módulo micronaut Test Rest Assured facilita a integração da biblioteca Rest Assured.
Sua utilização elimina a necessidade de codificar a versão e simplifica os testes, permitindo a injeção de dados RequestSpecification em campos de teste ou parâmetros de método (parâmetros suportados apenas com o JUnit 5):

## Controlador

``` java
package example.micronaut;

import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;

@Controller("/hello")
public class HelloController {
    @Get
    @Produces(MediaType.TEXT_PLAIN)
    public String index() {
        return "Hello World";
    }
}
```

## Rest assured

``` java
<dependency>
    <groupId>io.micronaut.test</groupId>
    <artifactId>micronaut-test-rest-assured</artifactId>
    <scope>test</scope>
</dependency>
```

## Teste

``` java

package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import io.restassured.specification.RequestSpecification;
import org.junit.jupiter.api.Test;

import static org.hamcrest.CoreMatchers.is;

@MicronautTest // <1>
public class HelloControllerTest {

    @Test // <2>
    public void testHelloEndpoint(RequestSpecification spec) {
        spec // <3>
            .when()
                .get("/hello")
            .then()
                .statusCode(200)
                .body(is("Hello World"));
    }
}
```

1. **@MicronautTest**: A anotação ativa o suporte ao teste do Micronaut e fornece a configuração necessária para o teste.
2. Injetar o RequestSpecification como parâmetro do método de teste.
3. **spec**: O objeto RequestSpecification é injetado como parâmetro do método de teste. Ele fornece uma maneira de construir e executar solicitações HTTP.


