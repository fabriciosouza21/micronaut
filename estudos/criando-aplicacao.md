**05-05-25**

Existe duas forma de criar uma aplicação no micronaut o micronaut-cli e o micronaut-launcher. o micronaut-laucher é uma solução bem parecida com spring-boot-starter.

``` bash
mn create-app example.micronaut.micronautguide --build=maven --lang=java
```

no exemplo de criação é utilizado os parâmetros:

- example.micronaut.micronautguide: nome do pacote
- --build=maven: tipo de build
- --lang=java: linguagem utilizada
- --test argument, adicionar a dependência de teste, no java e kotlin é utilizado o JUnit


## Aplicação

é criada a classe Aplication.java que é usada para executar a aplicação

``` java
package example.micronaut;

import io.micronaut.runtime.Micronaut;

public class Application {

    public static void main(String[] args) {
        Micronaut.run(Application.class, args);
    }
}
```


## Controller

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

1. **@Controller**: A anotação mapeia o path `/hello`
2. **@Get**: a anotação mapeia o método `index` para o verbo HTTP GET
3.**@Produces** Por padrão, o micronaut retornar o tipo application/json como Content-Type,Então podemos alterar utiliza o @Produces para alterar o tipo de retorno


## Teste

``` java
package example.micronaut;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.MediaType;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;

import jakarta.inject.Inject;

@MicronautTest // <1>
public class HelloControllerTest {

    @Inject
    @Client("/") // <2>
    HttpClient client;

    @Test
    public void testHello() {
        HttpRequest<?> request = HttpRequest.GET("/hello").accept(MediaType.TEXT_PLAIN);
        String body = client.toBlocking().retrieve(request); // <3>

        assertNotNull(body);
        assertEquals("Hello World", body);
    }
}
```

1. **@MicronautTest** - inicializa o contexto do micronaut e adicionar o server.
2. **@Client("/")** - injeta o cliente HTTP do micronaut
3. **HttpRequest** - criar um http request ficilmente graças a o micronaut fluid API.

``` java
./mvnw test
```

## Resumo

Fazendo um comparativo com o spring-boot é muito similar, preciso verificar o que seria o ResponseEntity do micronaut.

preciso verificar também como funcionar o  HttpRequest e o http client do micronaut, acredito que seja paracido com o do spring.

## Generate a micronaut aplication GrallVm

utilizando imagem nativa, preciso verificar como funciona o graalvm com o micronaut, o micronaut tem suporte a graalvm.

``` bash
sdk install java 21.0.5-graal
```



``` bash

./mvnw package -Dpackaging=native-image

```

Podemos customizar o processo de build do graalvm, para isso precisamos adicionar o plugin `native-maven-plugin` no pom.xml

``` xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <version>0.10.3</version>
    <configuration>
        <!-- <1> -->
        <imageName>mn-graalvm-application</imageName>
        <buildArgs>
              <!-- <2> -->
          <buildArg>-Ob</buildArg>
        </buildArgs>
    </configuration>
</plugin>
```

1. **mn-graalvm-application**: nome da imagem gerada
2. **-Ob**: é possivel to pass extra build arguments para nativa image. -Ob habilitar the quick com o build mode.
