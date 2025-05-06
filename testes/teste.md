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


# [Testando integrações de API REST usando Testcontainers com WireMock ou MockServer](https://guides.micronaut.io/latest/testing-rest-api-integrations-using-mockserver-maven-java.html)

aplicativo micronaut que se comunica com uma api Rest externa. testaremos a integração com wireMock do TestContainer e o mockServer

Implementaremos um endpoint de API REST para buscar um álbum para o album Id fornecido . Essa API se comunica internamente com o serviço de fotos para buscar as fotos daquele álbum.

Usaremos o WireMock , uma ferramenta para criar APIs simuladas, para simular as interações de serviços externos e testar nossos endpoints de API. O Testcontainers fornece o módulo WireMock do Testcontainers para que possamos executar o WireMock como um contêiner Docker.


## Criar o modelo

``` java
package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

@Serdeable
public record Photo(Long id, String title, String url, String thumbnailUrl) {
}
```


``` java
package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

import java.util.List;

@Serdeable
public record Album(Long albumId, List<Photo> photos) {
}
```

## Criar photServiceClient

``` java

package example.micronaut;

import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.PathVariable;
import io.micronaut.http.client.annotation.Client;

import java.util.List;

@Client(id = "photosapi")
interface PhotoServiceClient {

    @Get("/albums/{albumId}/photos")
    List<Photo> getPhotos(@PathVariable Long albumId);
}
```

1. Use @Client para usar clientes HTTP declarativos . Você pode anotar interfaces ou classes abstratas. Você pode usar o id membro para fornecer um identificador de serviço ou especificar a URL diretamente como o valor da anotação.

```properties
micronaut.http.services.photosapi.url=https://jsonplaceholder.typicode.com
```


## Implementado endpoint de API para obeter m álbum por id

``` java
package example.micronaut;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.PathVariable;
import io.micronaut.scheduling.annotation.ExecuteOn;

import static io.micronaut.scheduling.TaskExecutors.BLOCKING;

@Controller("/api")
class AlbumController {

    private final PhotoServiceClient photoServiceClient;

    AlbumController(PhotoServiceClient photoServiceClient) {
        this.photoServiceClient = photoServiceClient;
    }

    @ExecuteOn(BLOCKING) // <1>
    @Get("/albums/{albumId}")
    public Album getAlbumById(@PathVariable Long albumId) {
        return new Album(albumId, photoServiceClient.getPhotos(albumId));
    }
}
```

1. 	É essencial que quaisquer operações de E/S de bloqueio (como buscar dados do banco de dados) sejam descarregadas para um pool de threads separado que não bloqueie o loop de eventos.


Resposta da api

``` json

{
   "albumId": 1,
   "photos": [
       {
           "id": 51,
           "title": "non sunt voluptatem placeat consequuntur rem incidunt",
           "url": "https://via.placeholder.com/600/8e973b",
           "thumbnailUrl": "https://via.placeholder.com/150/8e973b"
       },
       {
           "id": 52,
           "title": "eveniet pariatur quia nobis reiciendis laboriosam ea",
           "url": "https://via.placeholder.com/600/121fa4",
           "thumbnailUrl": "https://via.placeholder.com/150/121fa4"
       },
       ...
       ...
   ]
}
```

## Teste

``` xml

<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <version>3.0.4</version>
    <scope>test</scope>
</dependency>
```


``` java

package example.micronaut;

import com.github.tomakehurst.wiremock.client.WireMock;
import com.github.tomakehurst.wiremock.junit5.WireMockExtension;
import io.micronaut.context.ApplicationContext;
import io.micronaut.http.MediaType;
import io.micronaut.runtime.server.EmbeddedServer;
import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.util.Collections;
import java.util.Map;

import static com.github.tomakehurst.wiremock.client.WireMock.aResponse;
import static com.github.tomakehurst.wiremock.client.WireMock.urlMatching;
import static com.github.tomakehurst.wiremock.core.WireMockConfiguration.wireMockConfig;
import static io.restassured.RestAssured.given;
import static org.hamcrest.CoreMatchers.is;
import static org.hamcrest.Matchers.hasSize;

@Testcontainers(disabledWithoutDocker = true) // <1>
class AlbumControllerTest {

    @RegisterExtension // <2>
    static WireMockExtension wireMock = WireMockExtension.newInstance()
            .options(wireMockConfig().dynamicPort())
            .build();

    private Map<String, Object> getProperties() {
        return Collections.singletonMap("micronaut.http.services.photosapi.url", // <3>
                wireMock.baseUrl());
    }

    @Test
    void shouldGetAlbumById() { // <4>
        try (EmbeddedServer server = ApplicationContext.run(EmbeddedServer.class, getProperties())) {
            RestAssured.port = server.getPort(); // <5>
            Long albumId = 1L;
            wireMock.stubFor( // <6>
                    WireMock.get(urlMatching("/albums/" + albumId + "/photos"))
                            .willReturn(
                                    aResponse()
                                            .withHeader("Content-Type", MediaType.APPLICATION_JSON)
                                            .withBody(
                                                    """
                                                            [
                                                                 {
                                                                     "id": 1,
                                                                     "title": "accusamus beatae ad facilis cum similique qui sunt",
                                                                     "url": "https://via.placeholder.com/600/92c952",
                                                                     "thumbnailUrl": "https://via.placeholder.com/150/92c952"
                                                                 },
                                                                 {
                                                                     "id": 2,
                                                                     "title": "reprehenderit est deserunt velit ipsam",
                                                                     "url": "https://via.placeholder.com/600/771796",
                                                                     "thumbnailUrl": "https://via.placeholder.com/150/771796"
                                                                 }
                                                             ]
                                                            """)));

            given().contentType(ContentType.JSON)
                    .when()
                    .get("/api/albums/{albumId}", albumId)
                    .then()
                    .statusCode(200)
                    .body("albumId", is(albumId.intValue()))
                    .body("photos", hasSize(2));
        }
    }

    @Test// <7>
    void shouldReturnServerErrorWhenPhotoServiceCallFailed() {
        try (EmbeddedServer server = ApplicationContext.run(EmbeddedServer.class, getProperties())) {
            RestAssured.port = server.getPort();// <8>
            Long albumId = 2L;
            wireMock.stubFor(WireMock.get(urlMatching("/albums/" + albumId + "/photos"))
                    .willReturn(aResponse().withStatus(500)));

            given().contentType(ContentType.JSON)
                    .when()
                    .get("/api/albums/{albumId}", albumId)
                    .then()
                    .statusCode(500);
        }
    }
}
```

1. Desabilite o teste se o Docker não estiver presente.
2. Podemos criar uma instância do servidor WireMock usando WireMockExtension .
3. 	Registramos a propriedade "micronaut.http.services.photosapi.url" apontando para a URL do ponto de extremidade do WireMock.
4. No teste shouldGetAlbumById() , definimos a resposta simulada esperada para /albums/{albumId}/photosa chamada da API, fizemos uma solicitação ao ponto de extremidade do nosso aplicativo /api/albums/{albumId}e verificamos a resposta.
5. 	Estamos usando a biblioteca RestAssured para testar nosso endpoint de API, então capturamos a porta aleatória na qual o aplicativo iniciou e inicializamos a porta RestAssured .
6. 	Defina as expectativas para uma chamada de API.
7. No teste shouldReturnServerErrorWhenPhotoServiceCallFailed() , definimos a resposta simulada esperada para /albums/{albumId}/photosa chamada de API para retornar o código de status InternalServerError 500 e fazer uma solicitação ao ponto de extremidade do nosso aplicativo /api/albums/{albumId}e verificar a resposta.

## Stubbing usando arquivos de mapeamento Json

No teste anterior, vimos como criar um stub de uma API usando wireMock.stubFor(…​) . Em vez de criar um stub usando a API Java WireMock, podemos usar uma configuração baseada em mapeamento JSON.

```xml
<dependency>
    <groupId>com.github.wiremock</groupId>
    <artifactId>wiremock-testcontainers-java</artifactId>
    <version>1.0-alpha-6</version>
    <scope>test</scope>
</dependency>
```

``` json
//src/teste/recursos/wiremock/mapeamentos/obter-fotos-do-álbum.json
{
  "mappings": [
    {
      "request": {
        "method": "GET",
        "urlPattern": "/albums/([0-9]+)/photos"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "application/json"
        },
        "bodyFileName": "album-photos-resp-200.json"
      }
    },
    {
      "request": {
        "method": "GET",
        "urlPattern": "/albums/2/photos"
      },
      "response": {
        "status": 500,
        "headers": {
          "Content-Type": "application/json"
        }
      }
    },
    {
      "request": {
        "method": "GET",
        "urlPattern": "/albums/3/photos"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "application/json"
        },
        "jsonBody": []
      }
    }
  ]
}
```

``` java

    @RegisterExtension
    static WireMockExtension wireMockServer = WireMockExtension.newInstance()
            .options(wireMockConfig().dynamicPort().usingFilesUnderClasspath("wiremock"))
            .build();
```

``` java

    @Test
    void shouldGetAlbumById() {
        Long albumId = 1L;
        try (EmbeddedServer server = ApplicationContext.run(EmbeddedServer.class, getProperties())) {
            RestAssured.port = server.getPort();

            given().contentType(ContentType.JSON)
                    .when()
                    .get("/api/albums/{albumId}", albumId)
                    .then()
                    .statusCode(200)
                    .body("albumId", is(albumId.intValue()))
                    .body("photos", hasSize(2));

        }
    }
```
Fica muito mais claro usar o json para definir o stub da API, e podemos usar o mesmo arquivo de mapeamento em diferentes testes.

## Usando o módulo wireMOck do Testcontainers

O módulo WireMock do Testcontainers permite provisionar o servidor WireMock como um contêiner autônomo dentro dos seus testes, com base no WireMock Docker .

``` json
///AlbumControllerTestcontainersTests/mocks-config.json
{
  "mappings": [
    {
      "request": {
        "method": "GET",
        "urlPattern": "/albums/([0-9]+)/photos"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "application/json"
        },
        "bodyFileName": "album-photos-response.json"
      }
    },
    {
      "request": {
        "method": "GET",
        "urlPattern": "/albums/2/photos"
      },
      "response": {
        "status": 500,
        "headers": {
          "Content-Type": "application/json"
        }
      }
    },
    {
      "request": {
        "method": "GET",
        "urlPattern": "/albums/3/photos"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "application/json"
        },
        "jsonBody": []
      }
    }
  ]
}

```

``` java
package example.micronaut;

import io.micronaut.context.ApplicationContext;
import io.micronaut.core.annotation.NonNull;
import io.micronaut.runtime.server.EmbeddedServer;
import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.wiremock.integrations.testcontainers.WireMockContainer;

import java.util.Collections;
import java.util.Map;

import static io.restassured.RestAssured.given;
import static org.hamcrest.CoreMatchers.is;
import static org.hamcrest.Matchers.hasSize;
import static org.hamcrest.Matchers.nullValue;

@Testcontainers(disabledWithoutDocker = true)
class AlbumControllerTestcontainersTests {

    @Container
    static WireMockContainer wiremockServer = new WireMockContainer("wiremock/wiremock:2.35.0")
            .withMapping("photos-by-album", AlbumControllerTestcontainersTests.class, "mocks-config.json")
            .withFileFromResource(
                    "album-photos-response.json",
                    AlbumControllerTestcontainersTests.class,
                    "album-photos-response.json");

    @NonNull
    public Map<String, Object> getProperties() {
        return Collections.singletonMap("micronaut.http.services.photosapi.url",
                wiremockServer.getBaseUrl());
    }

    @Test
    void shouldGetAlbumById() {
        Long albumId = 1L;
        try (EmbeddedServer server = ApplicationContext.run(EmbeddedServer.class, getProperties())) {
            RestAssured.port = server.getPort();

            given().contentType(ContentType.JSON)
                    .when()
                    .get("/api/albums/{albumId}", albumId)
                    .then()
                    .statusCode(200)
                    .body("albumId", is(albumId.intValue()))
                    .body("photos", hasSize(2));
        }
    }

    @Test
    void shouldReturnServerErrorWhenPhotoServiceCallFailed() {
        Long albumId = 2L;
        try (EmbeddedServer server = ApplicationContext.run(EmbeddedServer.class, getProperties())) {
            RestAssured.port = server.getPort();
            given().contentType(ContentType.JSON)
                    .when()
                    .get("/api/albums/{albumId}", albumId)
                    .then()
                    .statusCode(500);
        }
    }

    @Test
    void shouldReturnEmptyPhotos() {
        Long albumId = 3L;
        try (EmbeddedServer server = ApplicationContext.run(EmbeddedServer.class, getProperties())) {
            RestAssured.port = server.getPort();
            given().contentType(ContentType.JSON)
                    .when()
                    .get("/api/albums/{albumId}", albumId)
                    .then()
                    .statusCode(200)
                    .body("albumId", is(albumId.intValue()))
                    .body("photos", nullValue());
        }
    }
}
```

## Testando com MockServe

> Para qualquer sistema que você integrar via HTTP ou HTTPS, o MockServer pode ser usado como um mock configurado para retornar respostas específicas para diferentes solicitações, um proxy gravando e, opcionalmente, modificando solicitações e respostas, tanto um proxy para algumas solicitações quanto um mock para outras solicitações ao mesmo tempo.

``` xml

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mockserver</artifactId>
    <scope>test</scope>
</dependency>
```


``` xml

<dependency>
    <groupId>org.mock-server</groupId>
    <artifactId>mockserver-client-java</artifactId>
    <version>5.15.0</version>
    <scope>test</scope>
</dependency>
```

``` java

package example.micronaut;

import static io.restassured.RestAssured.given;
import static org.hamcrest.CoreMatchers.is;
import static org.hamcrest.Matchers.hasSize;
import static org.mockserver.model.HttpRequest.request;
import static org.mockserver.model.HttpResponse.response;
import static org.mockserver.model.JsonBody.json;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import io.micronaut.test.support.TestPropertyProvider;
import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import io.restassured.specification.RequestSpecification;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestInstance;
import org.mockserver.client.MockServerClient;
import org.mockserver.model.Header;
import org.mockserver.verify.VerificationTimes;
import org.testcontainers.containers.MockServerContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.util.Collections;
import java.util.Map;

@MicronautTest
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
@Testcontainers(disabledWithoutDocker = true)
class AlbumControllerMockServerTest implements TestPropertyProvider {
    @Container
    static MockServerContainer mockServerContainer = new MockServerContainer(
            DockerImageName.parse("mockserver/mockserver:5.15.0")
    );

    static MockServerClient mockServerClient;

    @Override
    public @NonNull Map<String, String> getProperties() {
        mockServerContainer.start();
        mockServerClient =
                new MockServerClient(
                        mockServerContainer.getHost(),
                        mockServerContainer.getServerPort()
                );
        return Collections.singletonMap("micronaut.http.services.photosapi.url",
                mockServerContainer.getEndpoint());
    }

    @BeforeEach
    void setUp() {
        mockServerClient.reset();
    }

    @Test
    void shouldGetAlbumById(RequestSpecification spec) {
        Long albumId = 1L;

        mockServerClient
                .when(request().withMethod("GET").withPath("/albums/" + albumId + "/photos"))
                .respond(
                        response()
                                .withStatusCode(200)
                                .withHeaders(new Header("Content-Type", "application/json; charset=utf-8"))
                                .withBody(
                                        json(
                                                """
                                                [
                                                     {
                                                         "id": 1,
                                                         "title": "accusamus beatae ad facilis cum similique qui sunt",
                                                         "url": "https://via.placeholder.com/600/92c952",
                                                         "thumbnailUrl": "https://via.placeholder.com/150/92c952"
                                                     },
                                                     {
                                                         "id": 2,
                                                         "title": "reprehenderit est deserunt velit ipsam",
                                                         "url": "https://via.placeholder.com/600/771796",
                                                         "thumbnailUrl": "https://via.placeholder.com/150/771796"
                                                     }
                                                 ]
                                                """
                                        )
                                )
                );
        spec
                .contentType(ContentType.JSON)
                .when()
                .get("/api/albums/{albumId}", albumId)
                .then()
                .statusCode(200)
                .body("albumId", is(albumId.intValue()))
                .body("photos", hasSize(2));

        verifyMockServerRequest("GET", "/albums/" + albumId + "/photos", 1);
    }

    private void verifyMockServerRequest(String method, String path, int times) {
        mockServerClient.verify(
                request().withMethod(method).withPath(path),
                VerificationTimes.exactly(times)
        );
    }
}

```

1. 	Anote a classe com @MicronautTestpara que o framework Micronaut inicialize o contexto da aplicação e o servidor embarcado.
2. As classes que implementam TestPropertyProviderdevem usar esta anotação para criar uma única instância de classe para todos os testes (não necessário em testes Spock).
3. Desabilite o teste se o Docker não estiver presente.
4. Quando precisar definir propriedades dinâmicas, implemente a TestPropertyProviderinterface. Sobrescreva o método .getProperties()e retorne as propriedades que deseja expor à aplicação.
5. Registramos a propriedade "micronaut.http.services.photosapi.url" apontando para o ponto de extremidade do contêiner MockServer.
6. Injetar uma instância de RequestSpecification.
7. O Micronaut Test define a porta do servidor incorporado em spec, portanto não é necessário injetá-la EmbeddedServere recuperá-la explicitamente.
