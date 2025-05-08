**08-05-25**

# [Access a database with Micronaut Data and Hibernate Reactive](https://guides.micronaut.io/latest/micronaut-data-hibernate-reactive-gradle-java.html)

## Dependências

```groovy

implementation("io.micronaut.data:micronaut-data-hibernate-reactive")
implementation("io.vertx:vertx-mysql-client")

```

## Configuração do banco de dados

```yaml

jpa:
  default:
    entity-scan:
      packages:
        - 'example.micronaut.domain'
    properties:
      hibernate:
        show-sql: true
        hbm2ddl:
          auto: update
        connection:
          db-type: mysql
    reactive: true
```

## Entidade

``` java
package example.micronaut.domain;

import io.micronaut.serde.annotation.Serdeable;
import jakarta.persistence.GenerationType;
import jakarta.validation.constraints.NotNull;
import jakarta.persistence.Column;
import jakarta.persistence.Convert;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.Id;

@Serdeable
@Entity
public class Genre {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    private String name;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Genre{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }
}
```

## Repositório

```java

package example.micronaut;

import example.micronaut.domain.Genre;
import io.micronaut.core.annotation.NonNull;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.annotation.Repository;
import io.micronaut.data.exceptions.DataAccessException;
import io.micronaut.data.repository.reactive.ReactorPageableRepository;
import reactor.core.publisher.Mono;

import jakarta.transaction.Transactional;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

@Repository // <1>
public interface GenreRepository extends ReactorPageableRepository<Genre, Long> { // <2>

    Mono<Genre> save(@NonNull @NotBlank String name);

    @Transactional
    default Mono<Genre> saveWithException(@NonNull @NotBlank String name) {
        return save(name)
                .handle((genre, sink) -> {
                    sink.error(new DataAccessException("test exception"));
                });
    }

    Mono<Long> update(@NonNull @NotNull @Id Long id, @NonNull @NotBlank String name);
}

```

1. 	Anote com @Repositorypara permitir que implementações em tempo de compilação sejam adicionadas.
2. 	Genre, a entidade a ser tratada como entidade raiz para fins de consulta, é estabelecida a partir da assinatura do método ou do parâmetro de tipo genérico especificado para a GenericRepository interface.

O repositório se estende de ReactorPageableRepository. Ele herda a hierarquia ReactorPageableRepository→ ReactorCrudRepository→ GenericRepository.


| Repositório | Descrição |
|-------------|-----------|
| `ReactorPageableRepository` | Um repositório reativo que suporta paginação. Ele fornece `findAll(Pageable)` e `findAll(Sort)`. |
| `ReactorCrudRepository` | Uma interface de repositório para executar CRUD reativo (Criar, Ler, Atualizar, Excluir). Ela fornece métodos como `findAll()`, `save(Genre)`, `deleteById(Long)` e `findById(Long)`. |
| `GenericRepository` | Uma interface raiz que não apresenta métodos, mas define o tipo de entidade e o tipo de ID como argumentos genéricos. |


## Controlador

dependência validação

``` groovy
annotationProcessor("io.micronaut.validation:micronaut-validation-processor")
implementation("io.micronaut.validation:micronaut-validation")
```

``` java
package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

@Serdeable
public class GenreUpdateCommand {

    @NotNull
    private final Long id;

    @NotBlank
    private final String name;

    public GenreUpdateCommand(Long id, String name) {
        this.id = id;
        this.name = name;
    }

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }
}
```

``` java
package example.micronaut;

import example.micronaut.domain.Genre;
import io.micronaut.data.exceptions.DataAccessException;
import io.micronaut.data.model.Page;
import io.micronaut.data.model.Pageable;
import io.micronaut.http.HttpHeaders;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.MutableHttpResponse;
import io.micronaut.http.annotation.Body;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Delete;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Post;
import io.micronaut.http.annotation.Put;
import reactor.core.publisher.Mono;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import java.net.URI;
import java.util.List;

@Controller("/genres")
public class GenreController {

    protected final GenreRepository genreRepository;

    public GenreController(GenreRepository genreRepository) {
        this.genreRepository = genreRepository;
    }

    @Get("/{id}")
    public Mono<Genre> show(Long id) {
        return genreRepository
                .findById(id);
    }

    @Put
    public Mono<HttpResponse<Genre>> update(@Body @Valid GenreUpdateCommand command) {
        return genreRepository.update(command.getId(), command.getName())
                .map(e -> HttpResponse
                        .<Genre>noContent()
                        .header(HttpHeaders.LOCATION, location(command.getId()).getPath()));

    }

    @Get("/list")

    public Mono<List<Genre>> list(@Valid Pageable pageable) {// <1>
        return genreRepository.findAll(pageable)
                .map(Page::getContent);
    }

    @Post
    public Mono<HttpResponse<Genre>> save(@Body("name") @NotBlank String name) {
        return genreRepository.save(name)
                .map(genre -> HttpResponse.created(genre)
                        .headers(headers -> headers.location(location(genre.getId()))));
    }

    @Post("/ex")
    public Mono<MutableHttpResponse<Genre>> saveExceptions(@Body @NotBlank String name) {
        return genreRepository
                .saveWithException(name)
                .map(genre -> HttpResponse
                        .created(genre)
                        .headers(headers -> headers.location(location(genre.getId())))
                )
                .onErrorReturn(DataAccessException.class, HttpResponse.noContent());
    }

    @Delete("/{id}")
    public Mono<HttpResponse<?>> delete(Long id) {
        return genreRepository.deleteById(id)
                .map(deleteId -> HttpResponse.noContent());
    }

    protected URI location(Long id) {
        return URI.create("/genres/" + id);
    }

    protected URI location(Genre genre) {
        return location(genre.getId());
    }
}
```

O interessante é que ele passa o pageable como parâmetro e utilizar o @Valid, podemos adicionar configurações no application.yml para definir o tamanho máximo da página, por exemplo:

```properties
micronaut.data.pageable.default-page-size.
```

## Testes

``` java
package example.micronaut;

import example.micronaut.domain.Genre;
import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpHeaders;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;

import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestInstance;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.Callable;

import static org.awaitility.Awaitility.await;
import static org.hamcrest.Matchers.equalTo;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertThrows;

@MicronautTest
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class GenreControllerTest {

    @Inject
    @Client("/")
    HttpClient httpClient;

    @Test
    void testFindNonExistingGenreReturns404() {
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, () -> {
            httpClient.toBlocking().exchange(HttpRequest.GET("/genres/99"));
        });

        assertNotNull(thrown.getResponse());
        assertEquals(HttpStatus.NOT_FOUND, thrown.getStatus());
    }

    @Test
    void testGenreCrudOperations() {
        List<Long> genreIds = new ArrayList<>();

        HttpRequest<?> request = HttpRequest.POST("/genres", Collections.singletonMap("name", "DevOps"));
        HttpResponse<?> response = httpClient.toBlocking().exchange(request);
        genreIds.add(entityId(response));

        assertEquals(HttpStatus.CREATED, response.getStatus());

        request = HttpRequest.POST("/genres", Collections.singletonMap("name", "Microservices"));
        response = httpClient.toBlocking().exchange(request);

        assertEquals(HttpStatus.CREATED, response.getStatus());

        Long id = entityId(response);
        genreIds.add(id);
        request = HttpRequest.GET("/genres/" + id);

        Genre genre = httpClient.toBlocking().retrieve(request, Genre.class);

        assertEquals("Microservices", genre.getName());

        request = HttpRequest.PUT("/genres", new GenreUpdateCommand(id, "Micro-services"));
        response = httpClient.toBlocking().exchange(request);

        assertEquals(HttpStatus.NO_CONTENT, response.getStatus());

        request = HttpRequest.GET("/genres/" + id);
        genre = httpClient.toBlocking().retrieve(request, Genre.class);
        assertEquals("Micro-services", genre.getName());

        await().until(countGenres(), equalTo(2));

        request = HttpRequest.POST("/genres/ex", Collections.singletonMap("name", "Microservices"));
        response = httpClient.toBlocking().exchange(request);

        assertEquals(HttpStatus.NO_CONTENT, response.getStatus());

        await().until(countGenres(), equalTo(2));

        request = HttpRequest.GET("/genres/list?size=1");
        List<Genre> genres = httpClient.toBlocking().retrieve(request, Argument.listOf(Genre.class));

        assertEquals(1, genres.size(), "Expected 1 genre, received: " + genres);
        assertEquals("DevOps", genres.get(0).getName());

        request = HttpRequest.GET("/genres/list?size=1&sort=name,desc");
        genres = httpClient.toBlocking().retrieve(request, Argument.listOf(Genre.class));

        assertEquals(1, genres.size(), "Expected 1 genre, received: " + genres);
        assertEquals("Micro-services", genres.get(0).getName());

        request = HttpRequest.GET("/genres/list?size=1&page=2");
        genres = httpClient.toBlocking().retrieve(request, Argument.listOf(Genre.class));

        assertEquals(0, genres.size(), "Expected 0 genres, received: " + genres);

        // cleanup:
        for (Long genreId : genreIds) {
            request = HttpRequest.DELETE("/genres/" + genreId);
            response = httpClient.toBlocking().exchange(request);
            assertEquals(HttpStatus.NO_CONTENT, response.getStatus());
        }
    }

    private Callable<Integer> countGenres() {
        return () -> httpClient
                .toBlocking()
                .retrieve(HttpRequest.GET("/genres/list"), Argument.listOf(Genre.class)).size();
    }

    protected Long entityId(HttpResponse<?> response) {
        String path = "/genres/";
        String value = response.header(HttpHeaders.LOCATION);
        if (value == null) {
            return null;
        }
        int index = value.indexOf(path);
        if (index != -1) {
            return Long.valueOf(value.substring(index + path.length()));
        }
        return null;
    }
}
```
