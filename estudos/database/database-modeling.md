**07-05-25**

# [Database Modeling](https://guides.micronaut.io/latest/micronaut-database-modeling-maven-java.html)


Neste tutorial, você desenvolverá um relacionamento um-para-muitos, conforme ilustrado nas tabelas a seguir.



**Tabela 1. Tabela: Contato**

| id | primeiro nome | sobrenome |
|---|---|---|
| 1 | Sérgio | do Amo |

**Tabela 2. Tabela: Telefone**

| id | telefone | ID do contato |
|---|---|---|
| 1 | +14155552671 | 1 |
| 2 | +442071838750 | 1 |

Neste exemplo, temos um relacionamento um-para-muitos entre um contato (Sérgio do Amo) e seus múltiplos números de telefone.

## configuração do banco de dados

``` properties
datasources.default.dialect=H2
datasources.default.driver-class-name=org.h2.Driver
datasources.default.url=jdbc\:h2\:mem\:devDb;LOCK_TIMEOUT\=10000;DB_CLOSE_ON_EXIT\=FALSE
datasources.default.username=sa
datasources.default.password=
```

## Migration Liquibase

``` xml
<dependency>
    <groupId>io.micronaut.liquibase</groupId>
    <artifactId>micronaut-liquibase</artifactId>
    <scope>compile</scope>
</dependency>
```

``` properties

liquibase.datasources.default.change-log=classpath\:db/liquibase-changelog.xml

```

`src/main/resources/db/liquibase-changelog.xml`

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
  xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
         http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd">
  <include file="changelog/01-schema.xml" relativeToChangelogFile="true"/>
</databaseChangeLog>
```


`src/main/resources/db/changelog/01-schema.xml`

``` xml
<?xml version="1.0" encoding="UTF-8"?>

<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
         http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd">
  <changeSet id="01" author="username">
    <createTable tableName="contact">
      <column name="id" type="BIGINT" autoIncrement="true">
        <constraints nullable="false"
                     unique="true"
                     primaryKey="true"
                     primaryKeyName="pk_contact"/>
      </column>

      <column name="first_name" type="VARCHAR(255)">
        <constraints nullable="true"/>
      </column>

      <column name="last_name" type="VARCHAR(255)">
        <constraints nullable="true"/>
      </column>
    </createTable>

    <createTable tableName="phone">
      <column name="id" type="BIGINT" autoIncrement="true">
        <constraints nullable="false"
                     unique="true"
                     primaryKey="true"
                     primaryKeyName="pk_phone"/>
      </column>
      <column name="phone" type="VARCHAR(20)">
        <constraints nullable="false"/>
      </column>

      <column name="contact_id" type="BIGINT">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <addForeignKeyConstraint baseTableName="phone"
                             baseColumnNames="contact_id"
                             constraintName="fk_phone_contact"
                             referencedTableName="contact"
                             referencedColumnNames="id"/>
    <rollback>
      <dropTable tableName="phone"/>
      <dropTable tableName="contact"/>
    </rollback>
  </changeSet>
</databaseChangeLog>
```
## Entidade de contato

``` java

package example.micronaut;

import io.micronaut.core.annotation.Nullable;
import io.micronaut.core.util.StringUtils;
import io.micronaut.data.annotation.GeneratedValue;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.annotation.MappedEntity;
import io.micronaut.data.annotation.Relation;

import java.util.List;

@MappedEntity("contact") // <1>
public record ContactEntity(
        @Id // <2>
        @GeneratedValue // <3>
        @Nullable // <4>
        Long id,

        @Nullable
        String firstName,

        @Nullable
        String lastName,

        @Nullable
        @Relation(value = Relation.Kind.ONE_TO_MANY, mappedBy = "contact") // <5>
        List<PhoneEntity>phones
) {
}
```

1. `@MappedEntity` - Mapeia a classe como uma entidade de banco de dados.
2. `@Id` - Indica que o campo é a chave primária da entidade.
3. Especifica que o valor da propriedade é gerado pelo banco de dados e não incluído nas inserções
4. 	Como [Registros](https://docs.oracle.com/en/java/javase/14/language/records.html) têm argumentos de construtor imutáveis, esses argumentos precisam ser marcados como @Nullable, e você deve passar nulo para esses argumentos.
5. Você pode especificar um relacionamento (um para um, um para muitos, etc.) com a @Relation anotação.

## Entidade de telefone

``` java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.annotation.Nullable;
import io.micronaut.data.annotation.GeneratedValue;
import io.micronaut.data.annotation.Id;
import io.micronaut.data.annotation.MappedEntity;
import io.micronaut.data.annotation.Relation;
@MappedEntity("phone")
public record PhoneEntity(
        @Id
        @GeneratedValue
        @Nullable
        Long id,

        @NonNull
        String phone,

        @Nullable
        @Relation(value = Relation.Kind.MANY_TO_ONE)
        ContactEntity contact
) {
}
```

1. `@MappedEntity` - Mapeia a classe como uma entidade de banco de dados.
2. @Id - Indica que o campo é a chave primária da entidade.
3. @GeneratedValue - Especifica que o valor da propriedade é gerado pelo banco de dados e não incluído nas inserções
4. @Nullable - Indica que o campo pode ser nulo.
5. @NonNull - Indica que o campo não pode ser nulo.
6. @Relation - Especifica o relacionamento entre as entidades. Neste caso, é um relacionamento muitos-para-um com a entidade ContactEntity.

## Projeções

``` java
package example.micronaut;

import io.micronaut.core.annotation.Introspected;
import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.annotation.Nullable;

import java.util.Set;

@Introspected
public record ContactComplete(
        @NonNull Long id,
        @Nullable String firstName,
        @Nullable String lastName,
        @Nullable Set<String> phones) {
}
```
1. 	Anote a classe com @Introspectedpara gerar BeanIntrospectionmetadados em tempo de compilação. Essas informações podem ser usadas, por exemplo, para renderizar o POJO como JSON usando Jackson sem usar reflexão.


registro Java para visualizar um contato, sem telefones:


``` java
package example.micronaut;

import io.micronaut.core.annotation.Introspected;
import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.annotation.Nullable;

@Introspected
public record ContactPreview(
        @NonNull Long id,
        @Nullable String firstName,
        @Nullable String lastName
) {
}
```

> Anote a classe com @Introspected para gerar BeanIntrospection metadados em tempo de compilação. Essas informações podem ser usadas, por exemplo, para renderizar o POJO como JSON usando Jackson sem usar reflexão.

## Repositorios

``` java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.data.jdbc.annotation.JdbcRepository;
import io.micronaut.data.model.query.builder.sql.Dialect;
import io.micronaut.data.repository.CrudRepository;

@JdbcRepository(dialect = Dialect.H2)
public interface PhoneRepository extends CrudRepository<PhoneEntity, Long> {
    void deleteByContact(@NonNull ContactEntity contact);
}
```

``` java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.data.annotation.Join;
import io.micronaut.data.annotation.Query;
import io.micronaut.data.jdbc.annotation.JdbcRepository;
import io.micronaut.data.model.query.builder.sql.Dialect;
import io.micronaut.data.repository.CrudRepository;

import java.util.Optional;

@JdbcRepository(dialect = Dialect.H2) // <1>
public interface ContactRepository extends CrudRepository<ContactEntity, Long> { // <2>
    @Join(value = "phones", type = Join.Type.LEFT_FETCH) // <3>
    Optional<ContactEntity> getById(@NonNull Long id);

    @Query("select id, first_name, last_name from contact where id = :id") // <4>
    Optional<ContactPreview> findPreviewById(@NonNull Long id);

    @Query("""
select c.id, c.first_name, c.last_name, group_concat(p.phone) as phones
 from contact c
 left outer join phone p on c.id = p.contact_id
 where c.id = :id
 group by c.id""") // <4>
    Optional<ContactComplete> findCompleteById(@NonNull Long id);
}
```

1. `@JdbcRepository` - Anotação que marca a interface como um repositório JDBC.
2. 	Ao estender, CrudRepositoryvocê habilita a geração automática de operações CRUD (Criar, Ler, Atualizar, Excluir).
3. Você pode usar a @Joinanotação na interface do seu repositório para especificar que a JOIN LEFT FETCH deve ser executado para recuperar o phones.
4.Você pode usar a @Queryanotação para especificar uma consulta explícita.


## Teste

``` java
package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

@MicronautTest(startApplication = false, transactional = false)
class ContactRepositoryTest {

    @Inject
    ContactRepository contactRepository;

    @Inject
    PhoneRepository phoneRepository;

    @Test
    void testAssociationsQuerying() {
        String firstName = "Sergio";
        String lastName = "Sergio";
        long contactCount = contactRepository.count();
        ContactEntity e = contactRepository.save(new ContactEntity(null, firstName, lastName, null));
        assertEquals(1 + contactCount, contactRepository.count());

        Optional<ContactPreview> preview = contactRepository.findPreviewById(e.id());
        assertTrue(preview.isPresent());
        assertEquals(new ContactPreview(e.id(), firstName, lastName), preview.get());

        // Query with @Join
        Optional<ContactEntity> contactEntity = contactRepository.getById(e.id());
        assertTrue(contactEntity.isPresent());
        ContactEntity expected = new ContactEntity(contactEntity.get().id(),
                firstName,
                lastName,
                Collections.emptyList());
        assertEquals(expected, contactEntity.get());

        Optional<ContactComplete> complete = contactRepository.findCompleteById(e.id());
        assertTrue(complete.isPresent());
        assertEquals(new ContactComplete(e.id(), firstName, lastName, null), complete.get());

        String americanPhone = "+14155552671";
        String ukPhone = "+442071838750";
        long phoneCount = phoneRepository.count();
        ContactEntity contactReference = new ContactEntity(e.id(), null, null, null);
        PhoneEntity usPhoneEntity = phoneRepository.save(new PhoneEntity(null, americanPhone, contactReference));
        PhoneEntity ukPhoneEntity =phoneRepository.save(new PhoneEntity(null, ukPhone, contactReference));
        assertEquals(2 + phoneCount, phoneRepository.count());

        // Projection without join with @Query
        preview = contactRepository.findPreviewById(e.id());
        assertTrue(preview.isPresent());
        assertEquals(new ContactPreview(e.id(), firstName, lastName), preview.get());

        // findById without @Join
        contactEntity = contactRepository.findById(e.id());
        assertTrue(contactEntity.isPresent());
        assertEquals(new ContactEntity(contactEntity.get().id(), firstName, lastName, Collections.emptyList()), contactEntity.get());

        // Query with @Join
        contactEntity = contactRepository.getById(e.id());
        assertTrue(contactEntity.isPresent());
        expected = new ContactEntity(contactEntity.get().id(),
                firstName,
                lastName,
                List.of(
                        new PhoneEntity(usPhoneEntity.id(), usPhoneEntity.phone(), new ContactEntity(e.id(), e.firstName(), e.lastName(), Collections.emptyList())),
                        new PhoneEntity(ukPhoneEntity.id(), ukPhoneEntity.phone(), new ContactEntity(e.id(), e.firstName(), e.lastName(), Collections.emptyList()))));
        assertEquals(expected, contactEntity.get());

        // Projection with join with @Query
        complete = contactRepository.findCompleteById(e.id());
        assertTrue(complete.isPresent());
        assertEquals(new ContactComplete(e.id(), firstName, lastName, Set.of(americanPhone, ukPhone)), complete.get());

        //cleanup
        phoneRepository.deleteByContact(contactReference);
        contactRepository.deleteById(e.id());
        assertEquals(phoneCount, phoneRepository.count());
        assertEquals(contactCount, contactRepository.count());
    }

}
```
