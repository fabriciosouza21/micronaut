**08-05-25**

# [Micronaut Data and Java Records](https://guides.micronaut.io/latest/micronaut-java-records-maven-java.html)

## Immutable Configuration

```java

package example.micronaut;

import io.micronaut.context.annotation.ConfigurationProperties;
import io.micronaut.core.annotation.NonNull;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

@ConfigurationProperties("vat")
public record ValueAddedTaxConfiguration(
    @NonNull @NotNull BigDecimal percentage) {
}
```

### Teste

```java

package example.micronaut;

import io.micronaut.context.BeanContext;
import io.micronaut.context.annotation.Property;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.junit.jupiter.api.Assertions.assertEquals;

@Property(name = "vat.percentage", value = "21.0") //<1>
@MicronautTest(startApplication = false) // <2>
class ValueAddedTaxConfigurationTest {

    @Inject
    BeanContext beanContext;

    @Test
    void immutableConfigurationViaJavaRecords() {
        assertTrue(beanContext.containsBean(ValueAddedTaxConfiguration.class));
        assertEquals(new BigDecimal("21.0"),
                beanContext.getBean(ValueAddedTaxConfiguration.class).percentage());
    }
}
```
1. Define the property in the test class.
2. Start the application with the test class as the main class.

### Migration

``` xml
<dependency>
    <groupId>io.micronaut.liquibase</groupId>
    <artifactId>micronaut-liquibase</artifactId>
    <scope>compile</scope>
</dependency>
```

```yml

liquibase:
  enabled: true
  datasources:
    default:
      change-log: 'classpath:db/liquibase-changelog.xml'
```

`src/main/resources/db/liquibase-changelog.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
         http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd">
    <include file="changelog/01-create-books-schema.xml" relativeToChangelogFile="true"/>
    <include file="changelog/02-insert-book.xml" relativeToChangelogFile="true"/>
</databaseChangeLog>
```

`src/main/resources/db/changelog/01-create-books-schema.xml`

``` xml
<?xml version="1.0" encoding="UTF-8"?>

<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
         http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd">
    <changeSet id="01" author="sdelamo">
        <createTable tableName="book" remarks="A table to contain all books">
            <column name="isbn" type="varchar(255)">
                <constraints nullable="false" unique="true" primaryKey="true"/>
            </column>
            <column name="title" type="varchar(255)">
                <constraints nullable="false"/>
            </column>
            <column name="price" type="NUMERIC">
                <constraints nullable="false"/>
            </column>
            <column name="about" type="LONGVARCHAR">
                <constraints nullable="true"/>
            </column>
        </createTable>
    </changeSet>
</databaseChangeLog>
```

`src/main/resources/db/changelog/02-insert-book.xml`
``` xml
<?xml version="1.0" encoding="UTF-8"?>

<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
         http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd">
    <changeSet id="02" author="sdelamo">
        <insert tableName="book">
            <column name="isbn">0321601912</column>
            <column name="title">Continuous Delivery</column>
            <column name="price">39.99</column>
            <column name="about">Winner of the 2011 Jolt Excellence Award! Getting software released to users is often a painful, risky, and time-consuming process. This groundbreaking new book sets out the principles and technical practices that enable rapid, incremental delivery of high quality, valuable new functionality to users.</column>
        </insert>
    </changeSet>
</databaseChangeLog>
```

### Entidades

```java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.annotation.Nullable;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.annotation.MappedEntity;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import java.math.BigDecimal;

@MappedEntity
public record Book(@Id @NonNull @NotBlank @Size(max = 255) String isbn,
                   @NonNull @NotBlank @Size(max = 255) String title,
                   @NonNull @NotNull BigDecimal price,
                   @Nullable String about) {
}
```


### Projeções

``` java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.annotation.Nullable;
import io.micronaut.data.annotation.GeneratedValue;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.annotation.MappedEntity;
import io.micronaut.serde.annotation.Serdeable;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import java.math.BigDecimal;

@Serdeable
public record BookCard(@NonNull @NotBlank @Size(max = 255) String isbn,
                       @NonNull @NotBlank @Size(max = 255) String title,
                       @NonNull @NotNull BigDecimal price) {
}
```

```java
package example.micronaut;

import io.micronaut.data.jdbc.annotation.JdbcRepository;
import io.micronaut.data.model.query.builder.sql.Dialect;
import io.micronaut.data.repository.CrudRepository;
import java.util.List;

@JdbcRepository(dialect = Dialect.POSTGRES)
public interface BookRepository extends CrudRepository<Book, String> {
    List<BookCard> find();
}

```

### Serialização json


``` java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.annotation.ReflectiveAccess;
import io.micronaut.serde.annotation.Serdeable;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

@Serdeable // <1>
@ReflectiveAccess // <2>
public record BookForSale(@NonNull @NotBlank String isbn,// <3>
                          @NonNull @NotBlank String title,
                          @NonNull @NotNull BigDecimal price) { }
```
1. **@Serdeable** é usado para serializar e desserializar o objeto.
2. **@ReflectiveAccess** é usado para permitir o acesso reflexivo ao objeto. Por conta que jackson precisar acessar reflexivamente o objeto. Utilizando o GraalVM, precisa-se adicionar o seguinte no `src/main/resources/META-INF/native-image/reflect-config.json`
3. Contraints são usados para validar o objeto. O `@NotBlank` é usado para garantir que o valor não seja nulo ou vazio. O `@NotNull` é usado para garantir que o valor não seja nulo.


``` java

package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Collections;
import java.util.List;
import java.util.stream.Collectors;
import io.micronaut.scheduling.TaskExecutors;
import io.micronaut.scheduling.annotation.ExecuteOn;

@Controller("/books") // <1>
class BookController {

    private final BookRepository bookRepository;
    private final ValueAddedTaxConfiguration valueAddedTaxConfiguration;

    BookController(BookRepository bookRepository, // <2>
                   ValueAddedTaxConfiguration valueAddedTaxConfiguration) {
        this.bookRepository = bookRepository;
        this.valueAddedTaxConfiguration = valueAddedTaxConfiguration;
    }

    @ExecuteOn(TaskExecutors.BLOCKING) // <3>
    @Get // <4>
    List<BookForSale> index() { // <5>
        return bookRepository.find()
                .stream()
                .map(bookCard -> new BookForSale(bookCard.isbn(),
                    bookCard.title(),
                    salePrice(bookCard)))
                .collect(Collectors.toList());
    }

    @NonNull
    private BigDecimal salePrice(@NonNull BookCard bookCard) {
        return bookCard.price()
                .add(bookCard.price()
                    .multiply(valueAddedTaxConfiguration.percentage()
                        .divide(new BigDecimal("100.0"), 2, RoundingMode.HALF_DOWN)))
                .setScale(2, RoundingMode.HALF_DOWN);
    }
}
```

1. A classe é definida como um controlador com a anotação @Controller mapeada para o caminho /books.
2. Use injeção de construtor para injetar um bean do tipo BookRepository.
3. É essencial que quaisquer operações de E/S de bloqueio (como buscar dados do banco de dados) sejam descarregadas para um pool de threads separado que não bloqueie o loop de eventos.
4. A anotação @Get mapeia o indexmétodo para uma solicitação HTTP GET em '/'.
Responda a um registro Java e o aplicativo responderá a representação JSON do objeto

### Teste

``` java
package example.micronaut;

import io.micronaut.context.annotation.Property;
import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertEquals;

@Property(name = "vat.percentage", value = "21.0")
@MicronautTest(transactional = false)
class BookControllerTest {

    int booksInsertedInDbByMigration = 1;

    @Inject
    @Client("/")
    HttpClient httpClient;

    @Inject
    BookRepository bookRepository;

    @Test
    void recordsUsedForJsonSerialization() {
        String title = "Building Microservices";
        String isbn = "1491950358";

        String about = """
                        Distributed systems have become more fine-grained in the past 10 years, shifting from code-heavy monolithic applications to smaller, self-contained microservices. But developing these systems brings its own set of headaches. With lots of examples and practical advice, this book takes a holistic view of the topics that system architects and administrators must consider when building, managing, and evolving microservice architectures.

                        Microservice technologies are moving quickly. Author Sam Newman provides you with a firm grounding in the concepts while diving into current solutions for modeling, integrating, testing, deploying, and monitoring your own autonomous services. You’ll follow a fictional company throughout the book to learn how building a microservice architecture affects a single domain.

                        Discover how microservices allow you to align your system design with your organization’s goals
                        Learn options for integrating a service with the rest of your system
                        Take an incremental approach when splitting monolithic codebases
                        Deploy individual microservices through continuous integration
                        Examine the complexities of testing and monitoring distributed services
                        Manage security with user-to-service and service-to-service models
                        Understand the challenges of scaling microservice architectures
                        """;
        Book b = new Book(isbn,
            title,
            new BigDecimal("38.15"),
            about);
        Book book = bookRepository.save(b);
        assertEquals(booksInsertedInDbByMigration + 1, bookRepository.count());
        BlockingHttpClient client = httpClient.toBlocking();
        List<BookForSale> books = client.retrieve(HttpRequest.GET("/books"),
                Argument.listOf(BookForSale.class));
        assertNotNull(books);
        assertEquals(booksInsertedInDbByMigration + 1, books.size());
        assertEquals("Building Microservices", books.get(1).title());
        assertEquals("1491950358", books.get(1).isbn());
        assertEquals(new BigDecimal("46.16"), books.get(1).price());
        bookRepository.delete(book);
    }
}
```

## Recursos

``` yml
datasources:
  default:
    driverClassName: org.postgresql.Driver
    dialect: POSTGRES
    schema-generate: NONE
```

