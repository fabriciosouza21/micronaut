# [Access a database with Micronaut Data JDBC](https://guides.micronaut.io/latest/micronaut-data-jdbc-repository-maven-java.html)


## Datasource configuration

``` properties
datasources.default.dialect=MYSQL
datasources.default.driver-class-name=com.mysql.cj.jdbc.Driver
```

## Migrations

``` xml

<dependency>
    <groupId>io.micronaut.flyway</groupId>
    <artifactId>micronaut-flyway</artifactId>
    <scope>compile</scope>
</dependency>
```



`src/main/resources/application.properties`
``` properties

flyway.datasources.default.enabled=true

```

A migração do Flyway será acionada automaticamente antes do início do seu aplicativo Micronaut. O Flyway lerá os comandos de migração no resources/db/migration/diretório, os executará se necessário e verificará se a fonte de dados configurada é consistente com eles.

`src/main/resources/db/migration/V1__schema.sql`

``` sql
DROP TABLE IF EXISTS genre;

CREATE TABLE genre (
    id   BIGINT NOT NULL AUTO_INCREMENT UNIQUE PRIMARY KEY,
   name  VARCHAR(255) NOT NULL UNIQUE
);
```

## Domain

``` java
package example.micronaut.domain;

import io.micronaut.data.annotation.GeneratedValue;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.annotation.MappedEntity;
import io.micronaut.serde.annotation.Serdeable;

import jakarta.validation.constraints.NotNull;

@Serdeable
@MappedEntity
public class Genre {

    @Id
    @GeneratedValue(GeneratedValue.Type.AUTO)
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

## Repository

1. Quando você define um método como save(String name), o Micronaut Data analisa o nome do parâmetro (name) durante a compilação.
2. Ele então procura na entidade associada ao repositório (neste caso, Genre) por um atributo com o mesmo nome e tenta fazer a correspondência.
3. Quando encontra uma correspondência, ele gera código que vai:

   - Criar uma nova instância da entidade
   - Chamar o setter apropriado (neste caso, setName(name))
   - Persistir a entidade no banco de dados

Se você tivesse múltiplos atributos, como name e lastname, o Micronaut Data seguiria o mesmo princípio para cada parâmetro em um método como save(String name, String lastname) - ele associaria cada parâmetro ao atributo correspondente na entidade com base nos nomes.

``` java
package example.micronaut;

import example.micronaut.domain.Genre;
import io.micronaut.core.annotation.NonNull;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.exceptions.DataAccessException;
import io.micronaut.data.jdbc.annotation.JdbcRepository;
import io.micronaut.data.model.query.builder.sql.Dialect;
import io.micronaut.data.repository.PageableRepository;

import jakarta.transaction.Transactional;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

@JdbcRepository(dialect = Dialect.MYSQL) // <1>
public interface GenreRepository extends PageableRepository<Genre, Long> { // <2>

    Genre save(@NonNull @NotBlank String name);

    @Transactional
    default Genre saveWithException(@NonNull @NotBlank String name) {
        save(name);
        throw new DataAccessException("test exception");
    }

    long update(@NonNull @NotNull @Id Long id, @NonNull @NotBlank String name);
}
```

1. @JdbcRepositorycom um dialeto específico.
2. Genre, a entidade a ser tratada como entidade raiz para fins de consulta, é estabelecida a partir da assinatura do método ou do parâmetro de tipo genérico especificado para a GenericRepositoryinterface.



| Repositório | Descrição |
|-------------|-----------|
| `PageableRepository` | Um repositório que suporta paginação. Ele fornece `findAll(Pageable)` e `findAll(Sort)`. |
| `CrudRepository` | Uma interface de repositório para executar CRUD (Criar, Ler, Atualizar, Excluir). Ela fornece métodos como `findAll()`, `save(Genre)`, `deleteById(Long)` e `findById(Long)`. |
| `GenericRepository` | Uma interface raiz que não apresenta métodos, mas define o tipo de entidade e o tipo de ID como argumentos genéricos. |


## Controlador

``` xml
<!-- Add the following to your annotationProcessorPaths element -->
<path>
    <groupId>io.micronaut.validation</groupId>
    <artifactId>micronaut-validation-processor</artifactId>
</path>
<dependency>
    <groupId>io.micronaut.validation</groupId>
    <artifactId>micronaut-validation</artifactId>
    <scope>compile</scope>
</dependency>
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

### Controlador validator

``` xml
<!-- Add the following to your annotationProcessorPaths element -->
<path>
    <groupId>io.micronaut.validation</groupId>
    <artifactId>micronaut-validation-processor</artifactId>
</path>
<dependency>
    <groupId>io.micronaut.validation</groupId>
    <artifactId>micronaut-validation</artifactId>
    <scope>compile</scope>
</dependency>
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
import io.micronaut.data.model.Pageable;
import io.micronaut.http.HttpHeaders;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.annotation.Body;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Delete;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Post;
import io.micronaut.http.annotation.Put;
import io.micronaut.http.annotation.Status;
import io.micronaut.scheduling.TaskExecutors;
import io.micronaut.scheduling.annotation.ExecuteOn;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import java.net.URI;
import java.util.List;
import java.util.Optional;

@ExecuteOn(TaskExecutors.BLOCKING)
@Controller("/genres")
public class GenreController {

    protected final GenreRepository genreRepository;

    public GenreController(GenreRepository genreRepository) {
        this.genreRepository = genreRepository;
    }

    @Get("/{id}")
    public Optional<Genre> show(Long id) {
        return genreRepository
                .findById(id);
    }

    @Put
    public HttpResponse update(@Body @Valid GenreUpdateCommand command) {
        genreRepository.update(command.getId(), command.getName());
        return HttpResponse
                .noContent()
                .header(HttpHeaders.LOCATION, location(command.getId()).getPath());
    }

    @Get("/list")
    public List<Genre> list(@Valid Pageable pageable) {
        return genreRepository.findAll(pageable).getContent();
    }

    @Post
    public HttpResponse<Genre> save(@Body("name") @NotBlank String name) {
        Genre genre = genreRepository.save(name);

        return HttpResponse
                .created(genre)
                .headers(headers -> headers.location(location(genre.getId())));
    }

    @Post("/ex")
    public HttpResponse<Genre> saveExceptions(@Body @NotBlank String name) {
        try {
            Genre genre = genreRepository.saveWithException(name);
            return HttpResponse
                    .created(genre)
                    .headers(headers -> headers.location(location(genre.getId())));
        } catch(DataAccessException e) {
            return HttpResponse.noContent();
        }
    }

    @Delete("/{id}")
    @Status(HttpStatus.NO_CONTENT)
    public void delete(Long id) {
        genreRepository.deleteById(id);
    }

    protected URI location(Long id) {
        return URI.create("/genres/" + id);
    }

    protected URI location(Genre genre) {
        return location(genre.getId());
    }
}

``` java
package example.micronaut;

import example.micronaut.domain.Genre;
import io.micronaut.data.exceptions.DataAccessException;
import io.micronaut.data.model.Pageable;
import io.micronaut.http.HttpHeaders;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.annotation.Body;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Delete;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Post;
import io.micronaut.http.annotation.Put;
import io.micronaut.http.annotation.Status;
import io.micronaut.scheduling.TaskExecutors;
import io.micronaut.scheduling.annotation.ExecuteOn;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import java.net.URI;
import java.util.List;
import java.util.Optional;

@ExecuteOn(TaskExecutors.BLOCKING)  // <1>
@Controller("/genres")  // <2>
public class GenreController {

    protected final GenreRepository genreRepository;

    public GenreController(GenreRepository genreRepository) { // <3>
        this.genreRepository = genreRepository;
    }

    @Get("/{id}")  // <4>
    public Optional<Genre> show(Long id) {
        return genreRepository
                .findById(id);  // <5>
    }

    @Put // <6>
    public HttpResponse update(@Body @Valid GenreUpdateCommand command) { // <7>
        genreRepository.update(command.getId(), command.getName());
        return HttpResponse
                .noContent()
                .header(HttpHeaders.LOCATION, location(command.getId()).getPath()); // <8>
    }

    @Get("/list") // <9>
    public List<Genre> list(@Valid Pageable pageable) { // <10>
        return genreRepository.findAll(pageable).getContent();
    }

    @Post // <11>
    public HttpResponse<Genre> save(@Body("name") @NotBlank String name) {
        Genre genre = genreRepository.save(name);

        return HttpResponse
                .created(genre)
                .headers(headers -> headers.location(location(genre.getId())));
    }

    @Post("/ex") // <12>
    public HttpResponse<Genre> saveExceptions(@Body @NotBlank String name) {
        try {
            Genre genre = genreRepository.saveWithException(name);
            return HttpResponse
                    .created(genre)
                    .headers(headers -> headers.location(location(genre.getId())));
        } catch(DataAccessException e) {
            return HttpResponse.noContent();
        }
    }

    @Delete("/{id}")  // <13>
    @Status(HttpStatus.NO_CONTENT)
    public void delete(Long id) {
        genreRepository.deleteById(id);
    }

    protected URI location(Long id) {
        return URI.create("/genres/" + id);
    }

    protected URI location(Genre genre) {
        return location(genre.getId());
    }
}
```
1. É essencial que quaisquer operações de E/S de bloqueio (como buscar dados do banco de dados) sejam descarregadas para um pool de threads separado que não bloqueie o loop de eventos.
2. classe é definida como um controlador com a anotação @Controller mapeada para o caminho /genres.
3. Use injeção de construtor para injetar um bean do tipo GenreRepository.
Mapeia uma GETsolicitação para /genres/{id}, que tenta mostrar um gênero.
4. Isso ilustra o uso de uma variável de caminho de URL.
5. Retornar um opcional vazio quando o gênero não existe faz com que o framework Micronaut responda com 404 (não encontrado).
6. Mapeia uma PUTsolicitação para /genres, que tenta atualizar um gênero.
7. Adiciona @Valida qualquer parâmetro do método que requeira validação. Use um POJO fornecido como carga JSON na solicitação para preencher o comando.
8. É fácil adicionar cabeçalhos personalizados à resposta.
9. Mapeia uma GETsolicitação para /genres/list, que retorna uma lista de gêneros. Este mapeamento ilustra parâmetros de URL sendo mapeados para um único POJO.
10. Você pode vincular Pageablecomo um argumento de método do controlador. Confira os exemplos na seção de testes a seguir e leia as opções de configuração do Pageable . Por exemplo, você pode configurar o tamanho padrão da página com a propriedade de configuração micronaut.data.pageable.default-page-size.
11. Mapeia uma POSTsolicitação para /genres, que tenta salvar um gênero.
12. Mapeia uma POSTsolicitação para /ex, o que gera uma exceção.
13. Mapeia uma DELETEsolicitação para /genres/{id}, que tenta remover um gênero. Isso ilustra o uso de uma variável de caminho de URL.

## Teste

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

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertThrows;

@MicronautTest
public class GenreControllerTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    public void testFindNonExistingGenreReturns404() {
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, () -> {
            client.toBlocking().exchange(HttpRequest.GET("/genres/99"));
        });

        assertNotNull(thrown.getResponse());
        assertEquals(HttpStatus.NOT_FOUND, thrown.getStatus());
    }

    @Test
    public void testGenreCrudOperations() {

        List<Long> genreIds = new ArrayList<>();

        HttpRequest<?> request = HttpRequest.POST("/genres", Collections.singletonMap("name", "DevOps"));
        HttpResponse<?> response = client.toBlocking().exchange(request);
        genreIds.add(entityId(response));

        assertEquals(HttpStatus.CREATED, response.getStatus());

        request = HttpRequest.POST("/genres", Collections.singletonMap("name", "Microservices"));
        response = client.toBlocking().exchange(request);

        assertEquals(HttpStatus.CREATED, response.getStatus());

        Long id = entityId(response);
        genreIds.add(id);
        request = HttpRequest.GET("/genres/" + id);

        Genre genre = client.toBlocking().retrieve(request, Genre.class);

        assertEquals("Microservices", genre.getName());

        request = HttpRequest.PUT("/genres", new GenreUpdateCommand(id, "Micro-services"));
        response = client.toBlocking().exchange(request);

        assertEquals(HttpStatus.NO_CONTENT, response.getStatus());

        request = HttpRequest.GET("/genres/" + id);
        genre = client.toBlocking().retrieve(request, Genre.class);
        assertEquals("Micro-services", genre.getName());

        request = HttpRequest.GET("/genres/list");
        List<Genre> genres = client.toBlocking().retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(2, genres.size());

        request = HttpRequest.POST("/genres/ex", Collections.singletonMap("name", "Microservices"));
        response = client.toBlocking().exchange(request);

        assertEquals(HttpStatus.NO_CONTENT, response.getStatus());

        request = HttpRequest.GET("/genres/list");
        genres = client.toBlocking().retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(2, genres.size());

        request = HttpRequest.GET("/genres/list?size=1");
        genres = client.toBlocking().retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(1, genres.size());
        assertEquals("DevOps", genres.get(0).getName());

        request = HttpRequest.GET("/genres/list?size=1&sort=name,desc");
        genres = client.toBlocking().retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(1, genres.size());
        assertEquals("Micro-services", genres.get(0).getName());

        request = HttpRequest.GET("/genres/list?size=1&page=2");
        genres = client.toBlocking().retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(0, genres.size());

        // cleanup:
        for (Long genreId : genreIds) {
            request = HttpRequest.DELETE("/genres/" + genreId);
            response = client.toBlocking().exchange(request);
            assertEquals(HttpStatus.NO_CONTENT, response.getStatus());
        }
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
