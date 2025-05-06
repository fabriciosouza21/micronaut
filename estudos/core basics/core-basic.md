**06/05/25**

# [@Configuration e @ConfigurationBuilder](https://guides.micronaut.io/latest/micronaut-configuration-maven-java.html)

Utilizar as anotações @ConfigurationProperties, @ConfigurationBuilder e @EachProperty.


``` java
mn create-app example.micronaut.micronautguide \
    --features=yaml,serialization-jackson,graalvm \
    --build=maven \
    --lang=java \
    --test=junit
```


## Team configuration with @ConfigurationProperties



``` yml
// src/main/resources/application.yml
team:
  name: 'Steelers'
  color: 'Black'
  player-names:
    - 'Mason Rudolph'
    - 'James Connor'
```


``` java
@ConfigurationProperties("team")
public class TeamConfiguration {
    private String name;
    private String color;
    private List<String> playerNames;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
    }

    public List<String> getPlayerNames() {
        return playerNames;
    }

    public void setPlayerNames(List<String> playerNames) {
        this.playerNames = playerNames;
    }
}
```

### Teste @ConfigurationProperties

``` java
    @Test
    void testTeamConfiguration() {
        List<String> names = Arrays.asList("Nirav Assar", "Lionel Messi");
        Map<String, Object> items = new HashMap<>();
        items.put("team.name", "evolution");
        items.put("team.color", "green");
        items.put("team.player-names", names);

        ApplicationContext ctx = ApplicationContext.run(items); // <1>
        TeamConfiguration teamConfiguration = ctx.getBean(TeamConfiguration.class);

        assertEquals("evolution", teamConfiguration.getName());
        assertEquals("green", teamConfiguration.getColor());
        assertEquals(names.size(), teamConfiguration.getPlayerNames().size());
        names.forEach(name -> assertTrue(teamConfiguration.getPlayerNames().contains(name)));

        ctx.close();
    }
```

1. **ApplicationContext.run**: Executa o contexto da aplicação com as propriedades definidas no mapa.

## @ConfigurationBuilder

``` yml
team:
  name: 'Steelers'
  color: 'Black'
  player-names:
    - 'Mason Rudolph'
    - 'James Connor'
  team-admin:
    manager: 'Nirav Assar'
    coach: 'Mike Tomlin'
    president: 'Dan Rooney'
```


``` java

package example.micronaut;

public class TeamAdmin {

    private String manager;
    private String coach;
    private String president;

    // should use the builder pattern to create the object
    private TeamAdmin() {
    }

    public String getManager() {
        return manager;
    }

    public void setManager(String manager) {
        this.manager = manager;
    }

    public String getCoach() {
        return coach;
    }

    public void setCoach(String coach) {
        this.coach = coach;
    }

    public String getPresident() {
        return president;
    }

    public void setPresident(String president) {
        this.president = president;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String manager;
        private String coach;
        private String president;


        public Builder withManager(String manager) {
            this.manager = manager;
            return this;
        }

        public Builder withCoach(String coach) {
            this.coach = coach;
            return this;
        }

        public Builder withPresident(String president) {
            this.president = president;
            return this;
        }

        public TeamAdmin build() {
            TeamAdmin teamAdmin = new TeamAdmin();
            teamAdmin.manager = this.manager;
            teamAdmin.coach = this.coach;
            teamAdmin.president = this.president;
            return teamAdmin;
        }

        public String getManager() {
            return manager;
        }

        public String getCoach() {
            return coach;
        }

        public String getPresident() {
            return president;
        }
    }
}
```

Padrão builder para criar o objeto TeamAdmin.

``` java

package example.micronaut;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import io.micronaut.context.annotation.ConfigurationBuilder;
import io.micronaut.context.annotation.ConfigurationProperties;
import io.micronaut.serde.annotation.Serdeable;

import java.util.List;

@Serdeable //<1>
@JsonIgnoreProperties("builder") //<2>
//tag::teamConfigClassNoBuilder[]
@ConfigurationProperties("team")
public class TeamConfiguration {
    private String name;
    private String color;
    private List<String> playerNames;
//end::teamConfigClassNoBuilder[]

    public TeamConfiguration() {
    }

    @ConfigurationBuilder(prefixes = "with", configurationPrefix = "team-admin")//<3>
    protected TeamAdmin.Builder builder = TeamAdmin.builder();//<4>

    public TeamAdmin.Builder getBuilder() {
        return builder;
    }

    public void setBuilder(TeamAdmin.Builder builder) {
        this.builder = builder;
    }

    //tag::gettersandsetters[]
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
    }

    public List<String> getPlayerNames() {
        return playerNames;
    }

    public void setPlayerNames(List<String> playerNames) {
        this.playerNames = playerNames;
    }
}
//end::gettersandsetters[]

```

1. **@Serdeable**: Anotação do micronaut para serialização e desserialização de objetos.
2. Marca o campo `builder` como ignorado durante a serialização e desserialização.
3. prefixos informa ao framework micronaut para encontra método prefixado with; configurationPrefix permitir que o desenvolvedor personalize o application.yml
4. Instancie o objeto constutor para que posssa ser preenchido com valores.


> O atributo prefixes indica ao Micronaut quais são os prefixos dos métodos que devem ser chamados no builder. No seu exemplo, o valor é "with", o que significa que o Micronaut procurará por métodos que começam com "with" na classe TeamAdmin.Builder.


## Teste

``` java
    @Test
    void testTeamConfigurationBuilder() {
        List<String> names = Arrays.asList("Nirav Assar", "Lionel Messi");
        Map<String, Object> items = new HashMap<>();
        items.put("team.name", "evolution");
        items.put("team.color", "green");
        items.put("team.team-admin.manager", "Jerry Jones"); // <1>
        items.put("team.team-admin.coach", "Tommy O'Neill");
        items.put("team.team-admin.president", "Mark Scanell");
        items.put("team.player-names", names);

        ApplicationContext ctx = ApplicationContext.run(items);
        TeamConfiguration teamConfiguration = ctx.getBean(TeamConfiguration.class);
        TeamAdmin teamAdmin = teamConfiguration.builder.build(); // <2>

        assertEquals("evolution", teamConfiguration.getName());
        assertEquals("green", teamConfiguration.getColor());
        assertEquals("Nirav Assar", teamConfiguration.getPlayerNames().get(0));
        assertEquals("Lionel Messi", teamConfiguration.getPlayerNames().get(1));

        // check the builder has values set
        assertEquals("Jerry Jones", teamConfiguration.builder.getManager());
        assertEquals("Tommy O'Neill", teamConfiguration.builder.getCoach());
        assertEquals("Mark Scanell", teamConfiguration.builder.getPresident());

        // check the object can be built
        assertEquals("Jerry Jones", teamAdmin.getManager()); // <3>
        assertEquals("Tommy O'Neill", teamAdmin.getCoach());
        assertEquals("Mark Scanell", teamAdmin.getPresident());

        ctx.close();
    }
```

1. **team.team-admin.manager**: O Micronaut irá preencher o campo builder com o valor do mapa.
2. **teamConfiguration.builder.build()**: O Micronaut irá preencher o campo builder com o valor do mapa.
3. verifica se o objeto foi preenchido corretamente.

## @EachProperty

O framework também permite ler uma lista de configurações relacionadas

``` yml
stadium:
  coors:
    city: 'Denver'
    size: 50000
  pnc:
    city: 'Pittsburgh'
    size: 35000
```

``` java

package example.micronaut;

import io.micronaut.context.annotation.EachProperty;
import io.micronaut.context.annotation.Parameter;
import io.micronaut.serde.annotation.Serdeable;

@Serdeable
@EachProperty("stadium")
public class StadiumConfiguration {
    private String name;
    private String city;
    private Integer size;

    public StadiumConfiguration(@Parameter String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getCity() {
        return city;
    }

    public void setCity(String city) {
        this.city = city;
    }

    public Integer getSize() {
        return size;
    }

    public void setSize(Integer size) {
        this.size = size;
    }
}
```
1. **Serdeable**: Anotação do micronaut para serialização e desserialização de objetos.
2. **@EachProperty**: Anotação do micronaut para ler uma lista de configurações relacionadas.
3. **@Parameter**: Anotação do micronaut para ler o nome da propriedade.


### Teste @EachProperty

``` java
package example.micronaut;

import io.micronaut.context.ApplicationContext;
import io.micronaut.inject.qualifiers.Qualifiers;
import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.assertEquals;

public class StadiumConfigurationTest {

    @Test
    void testStadiumConfiguration() {
        Map<String, Object> items = new HashMap<>();
        items.put("stadium.fenway.city", "Boston");
        items.put("stadium.fenway.size", 60000);
        items.put("stadium.wrigley.city", "Chicago");
        items.put("stadium.wrigley.size", 45000);

        ApplicationContext ctx = ApplicationContext.run(items);


        StadiumConfiguration fenwayConfiguration = ctx.getBean(StadiumConfiguration.class, Qualifiers.byName("fenway"));
        StadiumConfiguration wrigleyConfiguration = ctx.getBean(StadiumConfiguration.class, Qualifiers.byName("wrigley"));

        assertEquals("fenway", fenwayConfiguration.getName());
        assertEquals(60000, fenwayConfiguration.getSize());
        assertEquals("wrigley", wrigleyConfiguration.getName());
        assertEquals(45000, wrigleyConfiguration.getSize());

        ctx.close();
    }
}
```

1. Varias configurações podem ser declarada para  amesma classe
2. **Qualifiers.byName**: Como há vários beans para recuperar um bean especifico.

## Controlador

``` java
package example.micronaut;

import io.micronaut.core.annotation.Nullable;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

import jakarta.inject.Named;

@Controller("/my")
public class MyController {

    private final TeamConfiguration teamConfiguration;
    private final StadiumConfiguration stadiumConfiguration;

    public MyController(@Nullable TeamConfiguration teamConfiguration,
                        @Nullable @Named("pnc") StadiumConfiguration stadiumConfiguration) {
        this.teamConfiguration = teamConfiguration;
        this.stadiumConfiguration = stadiumConfiguration;
    }

    @Get("/team")
    public TeamConfiguration team() {
        return this.teamConfiguration;
    }

    @Get("/stadium")
    public  StadiumConfiguration stadium() {
        return this.stadiumConfiguration;
    }
}
```

> 	Injeção de beans de configuração; @Named anotação é necessária para escolher qual StadiumConfigurationinstância será recuperada.

### Teste


``` java
package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;

import jakarta.inject.Inject;

import java.util.Arrays;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest
public class MyControllerTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    void testMyTeam() {
        TeamConfiguration teamConfiguration = client.toBlocking()
                .retrieve(HttpRequest.GET("/my/team"), TeamConfiguration.class);
        assertEquals("Steelers", teamConfiguration.getName());
        assertEquals("Black", teamConfiguration.getColor());
        List<String> expectedPlayers = Arrays.asList("Mason Rudolph", "James Connor");
        assertEquals(expectedPlayers.size(), teamConfiguration.getPlayerNames().size());
        expectedPlayers.forEach(name -> assertTrue(teamConfiguration.getPlayerNames().contains(name)));
    }

    @Test
    void testMyStadium() {
        StadiumConfiguration conf = client.toBlocking()
                .retrieve(HttpRequest.GET("/my/stadium"), StadiumConfiguration.class);
        assertEquals("Pittsburgh", conf.getCity());
        assertEquals(35000, conf.getSize());
    }
}
```

# [Injeção de dependência ](https://guides.micronaut.io/latest/micronaut-dependency-injection-types-maven-java.html)

[JSR-330 - Dependency Injection for Java specification.](https://javax-inject.github.io/javax-inject/)

## As primeiras versões do micronaut utilização javax.inject, mas a partir da versão 2.0.0 o micronaut utiliza jakarta.inject.


## Tipos de injeção de dependência


### Constructor Injection

``` java
package example.micronaut.constructor;

import example.micronaut.MessageService;
import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;

@Controller("/constructor")
class MessageController {
    private final MessageService messageService;

    MessageController(MessageService messageService) {
        this.messageService = messageService;
    }

    @Get
    @Produces(MediaType.TEXT_PLAIN)
    String index() {
        return messageService.compose();
    }
}
```

Se você estiver multiplos construtores, o micronaut framework procura por um contrutor anotado com a anotação @Inject.

### Field Injection

``` java
package example.micronaut.field;

import example.micronaut.MessageService;
import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;
import jakarta.inject.Inject;

@Controller("/field")
class MessageController {
    @Inject
    MessageService messageService;

    @Get
    @Produces(MediaType.TEXT_PLAIN)
    String index() {
        return messageService.compose();
    }
}
```

> A injeção de campo dificulta a compreensão dos requisitos de uma classe, facilitando a obtenção de uma resposta NullPointerExceptionao testar uma classe usando injeção de campo.


> O exemplo de código anterior usa um campo com o modificador de acesso default. Existem quatro tipos de modificadores de acesso disponíveis em Java: private, protected, publice o default. A injeção de campo funciona com todos eles. No entanto, a injeção de campo em um campo com private modificador de acesso requer reflexão. Portanto, recomendamos que você não use private.

> Use injeção de campo em classes de teste anotadas com MicronautTest. O Micronaut não instancia a classe de teste. Portanto, você não pode usar injeção de construtor nessas classes.

### Injeção de parâmetros

Para injeção de parâmetros de método, você define m método com oum ou parâmetros e anota o método com @Inject.

``` java
package example.micronaut.methodparameter;

import example.micronaut.MessageService;
import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;
import jakarta.inject.Inject;

@Controller("/setter")
class MessageController {
    private MessageService messageService;

    @Get
    @Produces(MediaType.TEXT_PLAIN)
    String index() {
        return messageService.compose();
    }

    @Inject
    void populateMessageService(MessageService messageService) {
        this.messageService = messageService;
    }
}
```


### Benefícios da injeção de construtor

- contrato claro -> A injeção de construtor expressa claramente os requisitos da classe e não requer nenhuma anotação adicional.
- imutabilidade -> A injeção de construtor permite definir final dependências, criando objetos imutáveis.
- Identificando odores de código -> A injeção de construtor ajuda você a identificar facilmente se seu bean depende de muitos outros objetos.
- Testabilidade -> A injeção de construtor facilita a criação de testes unitários, pois você pode criar instâncias de classes com dependências simuladas.

# [Scope types](https://guides.micronaut.io/latest/micronaut-scope-types-maven-java.html)

### Cenario

exemplo

``` java

package example.micronaut.singleton;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

import java.util.Arrays;
import java.util.List;

@Controller("/singleton") // <1>

public class RobotController {

    private final RobotFather father;
    private final RobotMother mother;

    public RobotController(RobotFather father, // <2>
                           RobotMother mother) {  // <3>
        this.father = father;
        this.mother = mother;
    }

    @Get // <4>
    List<String> children() {
        return Arrays.asList(
                father.child().getSerialNumber(),
                mother.child().getSerialNumber()
        );
    }
}
```

``` java
package example.micronaut.singleton;

import io.micronaut.core.annotation.NonNull;
import jakarta.inject.Singleton;

@Singleton
public class RobotFather {
    private final Robot robot;

    public RobotFather(Robot robot) {
        this.robot = robot;
    }

    @NonNull
    public Robot child() {
        return this.robot;
    }
}
```


``` java
package example.micronaut.singleton;

import io.micronaut.core.annotation.NonNull;
import jakarta.inject.Singleton;

@Singleton
public class RobotMother {
    private final Robot robot;

    public RobotMother(Robot robot) {
        this.robot = robot;
    }

    @NonNull
    public Robot child() {
        return this.robot;
    }
}
```

### Singleton

singleton indica que existira apenas uma instância do bean

``` java

package example.micronaut.singleton;

import io.micronaut.core.annotation.NonNull;
import jakarta.inject.Singleton;

import java.util.UUID;

@Singleton
public class Robot {
    @NonNull
    private final String serialNumber;

    public Robot() {
        serialNumber = UUID.randomUUID().toString();
    }

    @NonNull
    public String getSerialNumber() {
        return serialNumber;
    }
}
```

#### Teste

``` java
import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import java.util.HashSet;
import java.util.List;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
class SingletonScopeTest {
    @Inject
    @Client("/")
    HttpClient httpClient;

    @ParameterizedTest
    @ValueSource(strings = {"/singleton"})

    void onlyOneInstanceOfTheBeanExistsForSingletonBeans(String path) {
        BlockingHttpClient client = httpClient.toBlocking();
        Set<String> responses = new HashSet<>(executeRequest(client, path));
        assertEquals(1, responses.size());
        responses.addAll(executeRequest(client, path));
        assertEquals(1, responses.size());
    }

    List<String> executeRequest(BlockingHttpClient client, String path) {
        return client.retrieve(HttpRequest.GET(path),
          Argument.listOf(String.class));
    }
}
```


No exemplo acima, o Micronaut cria apenas uma instância do bean Robot. Portanto, o número de série retornado é o mesmo para ambos os robôs.


### Prototype

Indicar que uma nova instância do bean será criada sempre que o bean for injetado.

``` java

package example.micronaut.prototype;

import io.micronaut.context.annotation.Prototype;
import io.micronaut.core.annotation.NonNull;

import java.util.UUID;

@Prototype
public class Robot {
    @NonNull
    private final String serialNumber;

    public Robot() {
        serialNumber = UUID.randomUUID().toString();
    }

    @NonNull
    public String getSerialNumber() {
        return serialNumber;
    }
}
```

#### Teste

``` java

import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import java.util.HashSet;
import java.util.List;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
class PrototypeScopeTest {
    @Inject
    @Client("/")
    HttpClient httpClient;

    @ParameterizedTest
    @ValueSource(strings = {"/prototype"})

    void prototypeScopeIndicatesThatANewInstanceOfTheBeanIsCreatedEachTimeItIsInjected(String path) {
      BlockingHttpClient client = httpClient.toBlocking();
      Set<String> responses = new HashSet<>(executeRequest(client, path));
      assertEquals(2, responses.size());
      responses.addAll(executeRequest(client, path));
      assertEquals(2, responses.size());
    }

    private List<String> executeRequest(BlockingHttpClient client, String path) {
      return client.retrieve(HttpRequest.GET(path), Argument.listOf(String.class));
    }
}
```

No exemplo acima, o Micronaut cria uma nova instância do bean Robot sempre que o bean é injetado. Portanto, o número de série retornado é diferente para ambos os robôs.

### Request

@RequestScope, indica que uma nova instância do bean é criada a cada solicitação HTTP.

``` java

package example.micronaut.request;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.http.HttpRequest;
import io.micronaut.runtime.http.scope.RequestAware;
import io.micronaut.runtime.http.scope.RequestScope;
import java.util.Objects;

@RequestScope
public class Robot implements RequestAware {
    @NonNull
    private String serialNumber;

    @NonNull
    public String getSerialNumber() {
        return serialNumber;
    }

    @Override
    public void setRequest(HttpRequest<?> request) {
        this.serialNumber = Objects.requireNonNull(request.getHeaders().get("UUID"));
    }
}
```

#### Teste

``` java
import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
class RequestScopeTest {
    @Inject
    @Client("/")
    HttpClient httpClient;

    @Test
    void requestScopeScopeIsACustomScopeThatIndicatesANewInstanceOfTheBeanIsCreatedAndAssociatedWithEachHTTPRequest() {
        String path = "/request";
        BlockingHttpClient client = httpClient.toBlocking();
        Set<String> responses = new HashSet<>(executeRequest(client, path));
        assertEquals(1, responses.size());
        responses.addAll(executeRequest(client, path));
        assertEquals(2, responses.size());
    }

    private List<String> executeRequest(BlockingHttpClient client,
                                        String path) {
        return client.retrieve(createRequest(path), Argument.listOf(String.class));
    }
    private HttpRequest<?> createRequest(String path) {
        return HttpRequest.GET(path).header("UUID", UUID.randomUUID().toString());
    }
}
```

### @Refreshable

è um escopo que o bean seja atualizado por meior do /refresh endpoint.


``` java
package example.micronaut.refreshable;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.runtime.context.scope.Refreshable;
import java.util.UUID;

@Refreshable
public class Robot {
    @NonNull
    private final String serialNumber;

    public Robot() {
        serialNumber = UUID.randomUUID().toString();
    }

    @NonNull
    public String getSerialNumber() {
        return serialNumber;
    }
}
``` java
import io.micronaut.context.annotation.Property;
import io.micronaut.core.type.Argument;
import io.micronaut.core.util.StringUtils;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Collections;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.assertEquals;

@Property(name = "endpoints.refresh.enabled", value = StringUtils.TRUE)
@Property(name = "endpoints.refresh.sensitive", value = StringUtils.FALSE)
@MicronautTest
class RefreshableScopeTest {
    @Inject
    @Client("/")
    HttpClient httpClient;

    @Test
    void refreshableScopeIsACustomScopeThatAllowsABeansStateToBeRefreshedViaTheRefreshEndpoint() {

        String path = "/refreshable";
        BlockingHttpClient client = httpClient.toBlocking();
        Set<String> responses = new HashSet<>(executeRequest(client, path));
        assertEquals(1, responses.size());
        responses.addAll(executeRequest(client, path));
        assertEquals(1, responses.size());
        refresh(client);
        responses.addAll(executeRequest(client, path));
        assertEquals(2, responses.size());
    }

    private void refresh(BlockingHttpClient client) {
        client.exchange(HttpRequest.POST("/refresh",
                        Collections.singletonMap("force", true)));
    }

    private List<String> executeRequest(BlockingHttpClient client, String path) {
        return client.retrieve(HttpRequest.GET(path),
                               Argument.listOf(String.class));
    }
}
```

> @Refreshable scope é um escopo personalizado que permite que o estado de um bean seja atualizado por meio do endpoint /refresh .

``` xml
<dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-management</artifactId>
    <scope>compile</scope>
</dependency>
```

 ``` java
import io.micronaut.context.annotation.Property;
import io.micronaut.core.type.Argument;
import io.micronaut.core.util.StringUtils;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Collections;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.assertEquals;

@Property(name = "endpoints.refresh.enabled", value = StringUtils.TRUE)
@Property(name = "endpoints.refresh.sensitive", value = StringUtils.FALSE)
@MicronautTest
class RefreshableScopeTest {
    @Inject
    @Client("/")
    HttpClient httpClient;

    @Test
    void refreshableScopeIsACustomScopeThatAllowsABeansStateToBeRefreshedViaTheRefreshEndpoint() {

        String path = "/refreshable";
        BlockingHttpClient client = httpClient.toBlocking();
        Set<String> responses = new HashSet<>(executeRequest(client, path));
        assertEquals(1, responses.size());
        responses.addAll(executeRequest(client, path));
        assertEquals(1, responses.size());
        refresh(client);
        responses.addAll(executeRequest(client, path));
        assertEquals(2, responses.size());
    }

    private void refresh(BlockingHttpClient client) {
        client.exchange(HttpRequest.POST("/refresh",
                        Collections.singletonMap("force", true)));
    }

    private List<String> executeRequest(BlockingHttpClient client, String path) {
        return client.retrieve(HttpRequest.GET(path),
                               Argument.listOf(String.class));
    }
}```


#### Teste

``` java
import io.micronaut.context.annotation.Property;
import io.micronaut.core.type.Argument;
import io.micronaut.core.util.StringUtils;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Collections;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.assertEquals;

@Property(name = "endpoints.refresh.enabled", value = StringUtils.TRUE) //<1>
@Property(name = "endpoints.refresh.sensitive", value = StringUtils.FALSE)//<2>
@MicronautTest
class RefreshableScopeTest {
    @Inject
    @Client("/")
    HttpClient httpClient;

    @Test
    void refreshableScopeIsACustomScopeThatAllowsABeansStateToBeRefreshedViaTheRefreshEndpoint() {

        String path = "/refreshable";
        BlockingHttpClient client = httpClient.toBlocking();
        Set<String> responses = new HashSet<>(executeRequest(client, path));
		//A mesma instância preenche ambos os pontos de injeção em RobotFathere RobotMother.
        assertEquals(1, responses.size());
        responses.addAll(executeRequest(client, path));
		//A mesma instância preenche ambos os pontos de injeção em RobotFathere RobotMother.
        assertEquals(1, responses.size());

		// atualiza o bean
        refresh(client);

        responses.addAll(executeRequest(client, path));
		//	Uma nova instância de Roboté criada na próxima vez que o objeto for solicitado.
        assertEquals(2, responses.size());
    }

    private void refresh(BlockingHttpClient client) {
        client.exchange(HttpRequest.POST("/refresh",
                        Collections.singletonMap("force", true)));
    }

    private List<String> executeRequest(BlockingHttpClient client, String path) {
        return client.retrieve(HttpRequest.GET(path),
                               Argument.listOf(String.class));
    }
}
```

### @Context


Indica que o bean será criado ao mesmo tempo que o ApplicationContext.
``` java
package example.micronaut.context;

import io.micronaut.context.annotation.ConfigurationProperties;
import io.micronaut.context.annotation.Context;
import jakarta.validation.constraints.Pattern;

@Context
@ConfigurationProperties("micronaut")
public class MicronautConfiguration {

    @Pattern(regexp = "groovy|java|kotlin")
    private String language;

    public String getLanguage() {
        return language;
    }

    public void setLanguage(String language) {
        this.language = language;
    }
}
```

#### Teste

``` java
package example.micronaut;

import io.micronaut.context.ApplicationContext;
import io.micronaut.context.exceptions.BeanInstantiationException;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.function.Executable;

import java.util.Collections;

import static org.junit.jupiter.api.Assertions.*;

class ContextTest {

    @Test
    void lifeCycleOfClassesAnnotatedWithAtContextIsBoundToThatOfTheBeanContext() {
        Executable e = () -> ApplicationContext.run(Collections.singletonMap("micronaut.language", "scala"));
        BeanInstantiationException thrown = assertThrows(BeanInstantiationException.class, e);
        assertTrue(thrown.getMessage().contains("language - must match \"groovy|java|kotlin\""));
    }
}

```

### Outros escopos

1. **@@Infrastructure**  -  representa um bean que não pode ser substituído ou anulado @Replaces porque é crítico para o funcionamento do sistema.
2. **@ThreadLocal** - scope que está associado a um thread específico.

