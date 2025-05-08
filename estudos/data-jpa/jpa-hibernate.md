**08-05-25**

# [Access a database with JPA and Hibernate](https://guides.micronaut.io/latest/micronaut-jpa-hibernate-maven-java.html)

## Dependencies

```xml
<dependency>
    <groupId>io.micronaut.sql</groupId>
    <artifactId>micronaut-hibernate-jpa</artifactId>
    <scope>compile</scope>
</dependency>
<dependency>
    <groupId>io.micronaut.data</groupId>
    <artifactId>micronaut-data-tx-hibernate</artifactId>
    <scope>compile</scope>
</dependency>
<dependency>
    <groupId>io.micronaut.sql</groupId>
    <artifactId>micronaut-jdbc-hikari</artifactId>
    <scope>compile</scope>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

## Configuration

```properties
datasources.default.password=${JDBC_PASSWORD:""}
datasources.default.url=${JDBC_URL:`jdbc:h2:mem:default;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE`}
datasources.default.username=${JDBC_USER:sa}
datasources.default.driver-class-name=${JDBC_DRIVER:org.h2.Driver}
```

Jpa Configuration

``` properties
jpa.default.properties.hibernate.hbm2ddl.auto=update
jpa.default.properties.hibernate.show_sql=true
```

## Domain

``` java
package example.micronaut.domain;

import com.fasterxml.jackson.annotation.JsonIgnore;

import io.micronaut.serde.annotation.Serdeable;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.Id;
import jakarta.persistence.OneToMany;
import jakarta.persistence.Table;
import jakarta.validation.constraints.NotNull;
import java.util.HashSet;
import java.util.Set;

import static jakarta.persistence.GenerationType.AUTO;

@Serdeable
@Entity
@Table(name = "genre")
public class Genre {

    @Id
    @GeneratedValue(strategy = AUTO)
    private Long id;

    @NotNull
    @Column(name = "name", nullable = false, unique = true)
    private String name;

    @JsonIgnore
    @OneToMany(mappedBy = "genre")
    private Set<Book> books = new HashSet<>();

    public Genre() {}

    public Genre(@NotNull String name) {
        this.name = name;
    }

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

    public Set<Book> getBooks() {
        return books;
    }

    public void setBooks(Set<Book> books) {
        this.books = books;
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

``` java
package example.micronaut.domain;

import io.micronaut.serde.annotation.Serdeable;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.Id;
import jakarta.persistence.ManyToOne;
import jakarta.persistence.Table;
import jakarta.validation.constraints.NotNull;

import static jakarta.persistence.GenerationType.AUTO;

@Serdeable
@Entity
@Table(name = "book")
public class Book {

    @Id
    @GeneratedValue(strategy = AUTO)
    private Long id;

    @NotNull
    @Column(name = "name", nullable = false)
    private String name;

    @NotNull
    @Column(name = "isbn", nullable = false)
    private String isbn;

    @ManyToOne
    private Genre genre;

    public Book() {}

    public Book(@NotNull String isbn,
                @NotNull String name,
                Genre genre) {
        this.isbn = isbn;
        this.name = name;
        this.genre = genre;
    }

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

    public String getIsbn() {
        return isbn;
    }

    public void setIsbn(String isbn) {
        this.isbn = isbn;
    }

    public Genre getGenre() {
        return genre;
    }

    public void setGenre(Genre genre) {
        this.genre = genre;
    }

    @Override
    public String toString() {
        return "Book{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", isbn='" + isbn + '\'' +
                ", genre=" + genre +
                '}';
    }
}
```

## Aplication configuration

``` java


package example.micronaut;

public interface ApplicationConfiguration {
    int getMax();
}

```

``` java
package example.micronaut;

import io.micronaut.context.annotation.ConfigurationProperties;

@ConfigurationProperties("application")
public class ApplicationConfigurationProperties implements ApplicationConfiguration {

    private final int DEFAULT_MAX = 10;

    private int max = DEFAULT_MAX;

    @Override
    public int getMax() {
        return max;
    }

    public void setMax(int max) {
        this.max = max;
    }
}
```

1. @ConfigurationProperties - obtem a configuração apartir do propriedades

``` properties
application.max=50
```


## Validation

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

## Repository acess

``` java
package example.micronaut;

import example.micronaut.domain.Genre;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.util.List;
import java.util.Optional;

public interface GenreRepository {

    Optional<Genre> findById(long id);

    Genre save(@NotBlank String name);

    Genre saveWithException(@NotBlank String name);

    void deleteById(long id);

    List<Genre> findAll(@NotNull SortingAndOrderArguments args);

    int update(long id, @NotBlank String name);
}

```


``` java
package example.micronaut;

import example.micronaut.domain.Genre;
import io.micronaut.transaction.annotation.ReadOnly;
import io.micronaut.transaction.annotation.Transactional;
import jakarta.inject.Singleton;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceException;
import jakarta.persistence.TypedQuery;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.util.Arrays;
import java.util.List;
import java.util.Optional;

@Singleton
public class GenreRepositoryImpl implements GenreRepository {

    private static final List<String> VALID_PROPERTY_NAMES = Arrays.asList("id", "name");

    private final EntityManager entityManager;
    private final ApplicationConfiguration applicationConfiguration;

    public GenreRepositoryImpl(EntityManager entityManager,
                               ApplicationConfiguration applicationConfiguration) {
        this.entityManager = entityManager;
        this.applicationConfiguration = applicationConfiguration;
    }

    @Override
    @ReadOnly
    public Optional<Genre> findById(long id) {
        return Optional.ofNullable(entityManager.find(Genre.class, id));
    }

    @Override
    @Transactional
    public Genre save(@NotBlank String name) {
        Genre genre = new Genre(name);
        entityManager.persist(genre);
        return genre;
    }

    @Override
    @Transactional
    public void deleteById(long id) {
        findById(id).ifPresent(entityManager::remove);
    }

    @ReadOnly
    public List<Genre> findAll(@NotNull SortingAndOrderArguments args) {
        String qlString = "SELECT g FROM Genre as g";
        if (args.order() != null && args.sort() != null && VALID_PROPERTY_NAMES.contains(args.sort())) {
            qlString += " ORDER BY g." + args.sort() + ' ' + args.order().toLowerCase();
        }
        TypedQuery<Genre> query = entityManager.createQuery(qlString, Genre.class);
        query.setMaxResults(args.max() != null ? args.max() : applicationConfiguration.getMax());
        if (args.offset() != null) {
            query.setFirstResult(args.offset());
        }
        return query.getResultList();
    }

    @Override
    @Transactional
    public int update(long id, @NotBlank String name) {
        return entityManager.createQuery("UPDATE Genre g SET name = :name where id = :id")
                .setParameter("name", name)
                .setParameter("id", id)
                .executeUpdate();
    }

    @Override
    @Transactional
    public Genre saveWithException(@NotBlank String name) {
        save(name);
        throw new PersistenceException();
    }
}
```

``` java
package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

import jakarta.validation.constraints.NotBlank;

@Serdeable
public class GenreSaveCommand {

    @NotBlank
    private String name;

    public GenreSaveCommand(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```


``` java
package example.micronaut;

import io.micronaut.core.annotation.Nullable;
import io.micronaut.serde.annotation.Serdeable;

import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Positive;
import jakarta.validation.constraints.PositiveOrZero;

@Serdeable
public record SortingAndOrderArguments(
        @Nullable
        @PositiveOrZero
        Integer offset,

        @Nullable
        @Positive
        Integer max,

         @Nullable
         @Pattern(regexp = "id|name")
         String sort,

         @Nullable
        @Pattern(regexp = "asc|ASC|desc|DESC")
        String order
) {

}
```

``` java
package example.micronaut;

import example.micronaut.domain.Genre;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.annotation.Body;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Delete;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Post;
import io.micronaut.http.annotation.Put;
import io.micronaut.scheduling.TaskExecutors;
import io.micronaut.scheduling.annotation.ExecuteOn;

import jakarta.persistence.PersistenceException;
import jakarta.validation.Valid;
import java.net.URI;
import java.util.List;

import static io.micronaut.http.HttpHeaders.LOCATION;

@ExecuteOn(TaskExecutors.BLOCKING)
@Controller("/genres")
class GenreController {

    private final GenreRepository genreRepository;

    GenreController(GenreRepository genreRepository) {
        this.genreRepository = genreRepository;
    }

    @Get("/{id}")
    Genre show(Long id) {
        return genreRepository
                .findById(id)
                .orElse(null);
    }

    @Put
    HttpResponse<?> update(@Body @Valid GenreUpdateCommand command) {
        int numberOfEntitiesUpdated = genreRepository.update(command.getId(), command.getName());

        return HttpResponse
                .noContent()
                .header(LOCATION, location(command.getId()).getPath());
    }

    @Get(value = "/list{?args*}")
    List<Genre> list(@Valid SortingAndOrderArguments args) {
        return genreRepository.findAll(args);
    }

    @Post
    HttpResponse<Genre> save(@Body @Valid GenreSaveCommand cmd) {
        Genre genre = genreRepository.save(cmd.getName());

        return HttpResponse
                .created(genre)
                .headers(headers -> headers.location(location(genre.getId())));
    }

    @Post("/ex")
    HttpResponse<Genre> saveExceptions(@Body @Valid GenreSaveCommand cmd) {
        try {
            Genre genre = genreRepository.saveWithException(cmd.getName());
            return HttpResponse
                    .created(genre)
                    .headers(headers -> headers.location(location(genre.getId())));
        } catch(PersistenceException e) {
            return HttpResponse.noContent();
        }
    }

    @Delete("/{id}")
    HttpResponse<?> delete(Long id) {
        genreRepository.deleteById(id);
        return HttpResponse.noContent();
    }

    private URI location(Long id) {
        return URI.create("/genres/" + id);
    }
}
```


## Test

``` java
package example.micronaut;

import example.micronaut.domain.Genre;
import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static io.micronaut.http.HttpHeaders.LOCATION;
import static io.micronaut.http.HttpStatus.BAD_REQUEST;
import static io.micronaut.http.HttpStatus.CREATED;
import static io.micronaut.http.HttpStatus.NOT_FOUND;
import static io.micronaut.http.HttpStatus.NO_CONTENT;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertThrows;

@MicronautTest
class GenreControllerTest {

    private BlockingHttpClient blockingClient;

    @Inject
    @Client("/")
    HttpClient client;

    @BeforeEach
    void setup() {
        blockingClient = client.toBlocking();
    }

    @Test
    void supplyAnInvalidOrderTriggersValidationFailure() {
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, () ->
                blockingClient.exchange(HttpRequest.GET("/genres/list?order=foo"))
        );

        assertNotNull(thrown.getResponse());
        assertEquals(BAD_REQUEST, thrown.getStatus());
    }

    @Test
    void testFindNonExistingGenreReturns404() {
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, () ->
            blockingClient.exchange(HttpRequest.GET("/genres/99"))
        );

        assertNotNull(thrown.getResponse());
        assertEquals(NOT_FOUND, thrown.getStatus());
    }

    @Test
    void testGenreCrudOperations() {

        List<Long> genreIds = new ArrayList<>();

        HttpRequest<?> request = HttpRequest.POST("/genres", new GenreSaveCommand("DevOps"));
        HttpResponse<?> response = blockingClient.exchange(request);
        genreIds.add(entityId(response));

        assertEquals(CREATED, response.getStatus());

        request = HttpRequest.POST("/genres", new GenreSaveCommand("Microservices"));
        response = blockingClient.exchange(request);

        assertEquals(CREATED, response.getStatus());

        Long id = entityId(response);
        genreIds.add(id);
        request = HttpRequest.GET("/genres/" + id);

        Genre genre = blockingClient.retrieve(request, Genre.class);

        assertEquals("Microservices", genre.getName());

        request = HttpRequest.PUT("/genres", new GenreUpdateCommand(id, "Micro-services"));
        response = blockingClient.exchange(request);

        assertEquals(NO_CONTENT, response.getStatus());

        request = HttpRequest.GET("/genres/" + id);
        genre = blockingClient.retrieve(request, Genre.class);
        assertEquals("Micro-services", genre.getName());

        request = HttpRequest.GET("/genres/list");
        List<Genre> genres = blockingClient.retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(2, genres.size());

        request = HttpRequest.POST("/genres/ex", new GenreSaveCommand("Microservices"));
        response = blockingClient.exchange(request);

        assertEquals(NO_CONTENT, response.getStatus());

        request = HttpRequest.GET("/genres/list");
        genres = blockingClient.retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(2, genres.size());

        request = HttpRequest.GET("/genres/list?max=1");
        genres = blockingClient.retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(1, genres.size());
        assertEquals("DevOps", genres.get(0).getName());

        request = HttpRequest.GET("/genres/list?max=1&order=desc&sort=name");
        genres = blockingClient.retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(1, genres.size());
        assertEquals("Micro-services", genres.get(0).getName());

        request = HttpRequest.GET("/genres/list?max=1&offset=10");
        genres = blockingClient.retrieve(request, Argument.of(List.class, Genre.class));

        assertEquals(0, genres.size());

        // cleanup:
        for (Long genreId : genreIds) {
            request = HttpRequest.DELETE("/genres/" + genreId);
            response = blockingClient.exchange(request);
            assertEquals(NO_CONTENT, response.getStatus());
        }
    }

    private Long entityId(HttpResponse response) {
        String path = "/genres/";
        String value = response.header(LOCATION);
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
