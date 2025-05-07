**07-05-25**

# [Replace H2 with a real database for testing](https://guides.micronaut.io/latest/replace-h2-with-real-database-for-testing-maven-java.html)


## Começando

- substitua o H2 por um banco de dados real para testes
- usar URL JDBC especial  do Testcontainers
- Testcontainers Junit 5
- Teste de repositorio com micronaut data
- simplify teste com micronaut Test Resoruces


## Entity

``` java
package example.micronaut;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

@Entity
@Table(name = "products")
public class Product {

    @Id
    private Long id;

    @Column(nullable = false, unique = true)
    private String code;

    @Column(nullable = false)
    private String name;

    public Product() {}

    public Product(Long id, String code, String name) {
        this.id = id;
        this.code = code;
        this.name = name;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getCode() {
        return code;
    }

    public void setCode(String code) {
        this.code = code;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

## Micronaut Data Repository

``` java

import io.micronaut.data.annotation.Query;
import io.micronaut.data.annotation.Repository;
import io.micronaut.data.jpa.repository.JpaRepository;

@Repository
interface ProductRepository extends JpaRepository<Product, Long> {

}

```

## Testando com banco de dados h2 na memoria

- O banco de dados de teste pode não suportar todos os recurso do seu banco de dados de produção
- A sintaxe da consulta SQL pode não ser compatível com o banco de dados na memória e com o banco de dados de produção
- Testar com o banco de dados diferente dquele que você usa para produção não lhe dará total confiança no seu conjunto de testes.

``` properties
jpa.default.properties.hibernate.hbm2ddl.auto=update
datasources.default.dialect=H2
datasources.default.driver-class-name=org.h2.Driver
datasources.default.url=jdbc\:h2\:mem\:devDb;LOCK_TIMEOUT\=10000;DB_CLOSE_ON_EXIT\=FALSE
datasources.default.username=sa
datasources.default.password=
```


```
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

## seed

``` java
insert into products(id, code, name) values(1, 'p101', 'Apple MacBook Pro');
insert into products(id, code, name) values(2, 'p102', 'Sony TV');
```

## Teste do ProductRepository

``` java

package example.micronaut;

import io.micronaut.core.io.ResourceLoader;
import io.micronaut.test.annotation.Sql;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.sql.Connection;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
@Sql(scripts = "classpath:sql/seed-data.sql", phase = Sql.Phase.BEFORE_EACH) //<1>
class ProductRepositoryTest {

    @Inject
    Connection connection;

    @Inject
    ResourceLoader resourceLoader;

    @Test
    void shouldGetAllProducts(ProductRepository productRepository) {
        List<Product> products = productRepository.findAll();
        assertEquals(2, products.size());
    }
}
```

1. Semeie o banco de dados usando a @Sqlanotação do micronaut-test.

por exemplo o sql abaixo não funciona no h2

``` sql
INSERT INTO products(id, code, name) VALUES(?,?,?) ON CONFLICT DO NOTHING;
```


## Repositorio de produtos

``` java
import io.micronaut.data.annotation.Query;
import io.micronaut.data.annotation.Repository;
import io.micronaut.data.jpa.repository.JpaRepository;

@Repository
interface ProductRepository extends JpaRepository<Product, Long> {

    default void createProductIfNotExists(Product product) {
        createProductIfNotExists(product.getId(), product.getCode(), product.getName());
    }

    @Query(
            value = "insert into products(id, code, name) values(:id, :code, :name) ON CONFLICT DO NOTHING",
            nativeQuery = true
    )
    void createProductIfNotExists(Long id, String code, String name);

}
```

## Testando usando TestContainers

### Configurando o pstgresSql

``` properties
jpa.default.properties.hibernate.hbm2ddl.auto=none
datasources.default.db-type=postgres
datasources.default.dialect=POSTGRES
datasources.default.driver-class-name=org.postgresql.Driver
```

initializer

`postgresql/src/teste/recursos/sql/init-db.sql`

``` sql
CREATE TABLE IF NOT EXISTS products
(
    id   int          not null,
    code varchar(255) not null,
    name varchar(255) not null,
    primary key (id),
    unique (code)
    );
```

#### Dependências do Testcontainers
``` xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>

```

``` xml

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>

```

``` xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```


#### JDBC dos contênineres de testes

``` java
package example.micronaut;

import io.micronaut.context.annotation.Property;
import io.micronaut.core.io.ResourceLoader;
import io.micronaut.test.annotation.Sql;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.sql.Connection;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest(startApplication = false) //<1>
@Property(name = "datasources.default.driver-class-name",
        value = "org.testcontainers.jdbc.ContainerDatabaseDriver") //<2>
@Property(name = "datasources.default.url",
        value = "jdbc:tc:postgresql:15.2-alpine:///db?TC_INITSCRIPT=sql/init-db.sql") //<3>
@Sql(scripts = "classpath:sql/seed-data.sql", phase = Sql.Phase.BEFORE_EACH) //<4>
class ProductRepositoryWithJdbcUrlTest {

    @Inject
    Connection connection;

    @Inject
    ResourceLoader resourceLoader;

    @Test
    void shouldGetAllProducts(ProductRepository productRepository) {
        List<Product> products = productRepository.findAll();
        assertEquals(2, products.size());
    }
}
```


1. Anote a classe com @MicronautTest para que o framework Micronaut inicialize o contexto da aplicação e o servidor embarcado. Por padrão, cada @Test método será encapsulado em uma transação que será revertida quando o teste for concluído. Esse comportamento pode ser alterado configurando transaction-o como false.
2. Anote a classe com @Propertypara fornecer a configuração do nome da classe do driver para o teste.
3. 	Anote a classe com @Propertypara fornecer a URL da fonte de dados que fornecemos por meio do URL JDBC especial do Testcontainers .
4. 	Semeie o banco de dados usando a @Sqlanotação do micronaut-test.


Se tivermos TestContainers e driver JDBC no classPATH,podemos simplesmente usar Os URLs de conexão JDBC especiais para obter uma nova instância em contêniner do banco de dados sempre que oo aplicativo for iniciado.

Para obter a URL especial do JDBC, insira tc: após jdbc:, conforme a seguir. (Observe que o nome do host, a porta e o nome do banco de dados serão ignorados; portanto, você pode deixá-los como estão ou defini-los com qualquer valor.)

`jdbc:tc:postgresql:///db`

Também podemos indicar qual versão do banco de dados PostgreSQL usar especificando a tag de imagem do Docker após postgresql da seguinte maneira:

`jdbc:tc:postgresql:15.2-alpine:///db`


Aqui, acrescentamos a tag 15.2-alpine ao postgresql para que nosso teste use um contêiner PostgreSQL criado a partir da imagem postgres:15.2-alpine .

Você também pode inicializar o banco de dados usando um script SQL passando o parâmetro TC_INITSCRIPT da seguinte maneira:

`jdbc:tc:postgresql:15.2-alpine:///db?TC_INITSCRIPT=sql/init-db.sql`

Os Testcontainers executarão automaticamente o script SQL especificado usando o parâmetro TC_INITSCRIPT . No entanto, o ideal é usar uma ferramenta de migração de banco de dados adequada, como Flyway ou Liquibase .

#### banco de dados usando Testcontainers e JUnit

``` java

package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.io.ResourceLoader;
import io.micronaut.test.annotation.Sql;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import io.micronaut.test.support.TestPropertyProvider;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestInstance;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.MountableFile;

import java.sql.Connection;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest(startApplication = false) //<1>
@Testcontainers(disabledWithoutDocker = true) //<2>
@TestInstance(TestInstance.Lifecycle.PER_CLASS) //<3>
@Sql(scripts = "classpath:sql/seed-data.sql", phase = Sql.Phase.BEFORE_EACH) //<4>
class ProductRepositoryTest implements TestPropertyProvider { //<5>

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>(
            "postgres:15.2-alpine"
    ).withCopyFileToContainer(MountableFile.forClasspathResource("sql/init-db.sql"), "/docker-entrypoint-initdb.d/init-db.sql");

    @Override
    public @NonNull Map<String, String> getProperties() { //<6>
        if (!postgres.isRunning()) {
            postgres.start();
        }
        return Map.of("datasources.default.driver-class-name", "org.postgresql.Driver",
                "datasources.default.url", postgres.getJdbcUrl(),
                "datasources.default.username", postgres.getUsername(),
                "datasources.default.password", postgres.getPassword());
    }

    @Inject
    Connection connection;

    @Inject
    ResourceLoader resourceLoader;

    @Inject
    ProductRepository productRepository;

    @Test
    void shouldGetAllProducts() {
        List<Product> products = productRepository.findAll();
        assertEquals(2, products.size());
    }

    @Test
    void shouldNotCreateAProductWithDuplicateCode() {
        Product product = new Product(3L, "p101", "Test Product");
        productRepository.createProductIfNotExists(product);
        Optional<Product> optionalProduct = productRepository.findById(product.getId());
        assertTrue(optionalProduct.isEmpty());
    }
}
```

1. Anote a classe com @MicronautTest para que o framework Micronaut inicialize o contexto da aplicação e o servidor embarcado. Por padrão, cada @Test método será encapsulado em uma transação que será revertida quando o teste for concluído. Esse comportamento pode ser alterado configurando transaction-o como false.
2. Desabilite o teste se o Docker não estiver presente.
3. As classes que implementam TestPropertyProviderdevem usar esta anotação para criar uma única instância de classe para todos os testes (não necessário em testes Spock).
4. Semeie o banco de dados usando a @Sqlanotação do micronaut-test.
5. Quando precisar definir propriedades dinâmicas, implemente a TestPropertyProviderinterface. Sobrescreva o método .getProperties()e retorne as propriedades que deseja expor à aplicação.


Usamos as anotações de extensão Testcontainers JUnit 5 @Testcontainers e @Container para iniciar o PostgreSQLContainer e registrar as propriedades da fonte de dados para o teste usando o registro de propriedade dinâmica por meio da API TestPropertyProvider .


## Testando com recursos de teste do Micronaut

> O Micronaut Test Resources adiciona suporte para gerenciar recursos externos que são necessários durante o desenvolvimento ou teste.

> Você pode habilitar o suporte a recursos de teste simplesmente definindo a propriedade micronaut.test.resources.enabled (no seu POM ou por meio da linha de comando).

1. Removendo dependências do Testcontainers
2. Configurar recurso de teste

## Recursos de teste

Quando o aplicativo é iniciado localmente — seja em teste ou executando o aplicativo  — a resolução da URL da fonte de dados é detectada e o serviço Test Resources inicia um contêiner Docker PostgreSQL local e injeta as propriedades necessárias para usá-lo como fonte de dados.

`postgresqltestresources/src/teste/java/exemplo/micronaut/ProductRepositoryTest.java`
``` java
package example.micronaut;

import io.micronaut.core.io.ResourceLoader;
import io.micronaut.test.annotation.Sql;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.sql.Connection;
import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest(startApplication = false) //<1>
@Sql(scripts = {"classpath:sql/init-db.sql", "classpath:sql/seed-data.sql"},
        phase = Sql.Phase.BEFORE_EACH) //<2>
class ProductRepositoryTest {

    @Inject
    Connection connection;

    @Inject
    ResourceLoader resourceLoader;

    @Inject
    ProductRepository productRepository;

    @Test
    void shouldGetAllProducts() {
        List<Product> products = productRepository.findAll();
        assertEquals(2, products.size());
    }

    @Test
    void shouldNotCreateAProductWithDuplicateCode() {
        Product product = new Product(3L, "p101", "Test Product");
        assertDoesNotThrow(() -> productRepository.createProductIfNotExists(product));
        Optional<Product> optionalProduct = productRepository.findById(product.getId());
        assertTrue(optionalProduct.isEmpty());
    }

}
```

1. Anote a classe com @MicronautTestpara que o framework Micronaut inicialize o contexto da aplicação e o servidor embarcado. Por padrão, cada @Testmétodo será encapsulado em uma transação que será revertida quando o teste for concluído. Esse comportamento pode ser alterado configurando transaction-o como false.
2. semeie o banco de dados usando a @Sqlanotação do micronaut-test.

Se você executar o teste, verá um contêiner PostgreSQL sendo iniciado pelo Test Resources por meio da integração com o Testcontainers para fornecer contêineres descartáveis ​​para testes.

## Objetivos dos Recursos de Teste do Micronaut

- configuração zero : sem adicionar nenhuma configuração, os recursos de teste devem ser gerados e o aplicativo configurado para utilizá-los. A configuração é necessária apenas para casos de uso avançados.

- isolamento do classpath : o uso de recursos de teste não deve vazar para o classpath do seu aplicativo, nem para o classpath do seu teste

- compatível com GraalVM nativo : se você construir um binário nativo ou executar testes no modo nativo, os recursos de teste devem estar disponíveis

- fácil de usar : os plugins de construção Micronaut para Gradle e Maven devem lidar com a complexidade de descobrir as dependências para você

- extensível : você pode implementar seus próprios recursos de teste, caso os integrados não cubram seu caso de uso

- tecnologia agnóstica : embora muitos recursos de teste usem Testcontainers por baixo dos panos, você pode usar qualquer outra tecnologia para criar recursos.

## Dados JDBc do micronaut

O Micronaut Data JDBC vai um passo além: você precisa especificar o dialeto na JdbcRepositoryanotação. O Micronaut Data JDBC pré-calcula consultas SQL nativas para o dialeto especificado, fornecendo uma implementação de repositório que é um mapeador de dados simples entre um conjunto de resultados nativo e uma entidade.

``` java
package example.micronaut;

import io.micronaut.data.annotation.Query;
import io.micronaut.data.jdbc.annotation.JdbcRepository;
import io.micronaut.data.model.query.builder.sql.Dialect;
import io.micronaut.data.repository.CrudRepository;

@JdbcRepository(dialect = Dialect.POSTGRES)
interface ProductRepository extends CrudRepository<Product, Long> {

    default void createProductIfNotExists(Product product) {
        createProductIfNotExists(product.getId(), product.getCode(), product.getName());
    }

    @Query(
            value = "insert into products(id, code, name) values(:id, :code, :name) ON CONFLICT DO NOTHING",
            nativeQuery = true
    )
    void createProductIfNotExists(Long id, String code, String name);
}
```

Analisamos como testar repositórios JPA do Micronaut Data usando o banco de dados H2 na memória e falamos sobre as desvantagens de usar diferentes bancos de dados (na memória) para testes enquanto usamos um tipo diferente de banco de dados na produção.

Em seguida, aprendemos como é fácil substituir o banco de dados H2 por um banco de dados real para testes usando a URL JDBC especial do Testcontainer. Também vimos como usar as anotações da extensão JUnit 5 do Testcontainer para iniciar o banco de dados para testes, o que proporciona mais controle sobre o ciclo de vida do contêiner do banco de dados.

Aprendemos que o Micronaut Test Resources simplifica os testes com contêineres descartáveis ​​por meio de sua integração com o Testcontainers.
