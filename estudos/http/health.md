**09-05-25**

## [Exposing a Health endpoint for your Micronaut application](https://guides.micronaut.io/latest/micronaut-health-endpoint-maven-java.html)


## Dependências

O Micronaut management e uma solução similar ao Spring Actuator, que fornece endpoints de gerenciamento para aplicações Micronaut. O Micronaut management fornece um endpoint de saúde, que pode ser usado para verificar o estado da aplicação.

``` xml

<dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-management</artifactId>
    <scope>compile</scope>
</dependency>
```


``` java
package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;

import jakarta.inject.Inject;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
public class HealthTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    public void healthEndpointExposed() {
        HttpStatus status = client.toBlocking().retrieve(HttpRequest.GET("/health"), HttpStatus.class);
        assertEquals(HttpStatus.OK, status);
    }
}
```


## Base Path

Para alterar o caminho base do endpoint de saúde, adicione a seguinte propriedade ao arquivo `application.properties`:

`endpoints.all.path`


``` java

package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.function.Executable;
import io.micronaut.context.annotation.Property;
import jakarta.inject.Inject;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

@Property(name = "endpoints.all.path", value = "/endpoints/")
@MicronautTest
public class HealthPathTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    public void healthEndpointExposedAtNonDefaultEndpointsPath() {
        HttpStatus status = client.toBlocking().retrieve(HttpRequest.GET("/endpoints/health"), HttpStatus.class);
        assertEquals(HttpStatus.OK, status);

        Executable e = () -> client.toBlocking().retrieve(HttpRequest.GET("/health"), HttpStatus.class);
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, e);
        assertEquals(HttpStatus.NOT_FOUND, thrown.getStatus());
    }
}
```

### Status com de teste falho

``` java

package example.micronaut;

import io.micronaut.context.annotation.Property;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.function.Executable;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

@Property(name = "endpoints.health.disk-space.threshold", value = "999999999999999999")
@MicronautTest
public class PoorHealthTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    public void healthEndpointExposesOutOfDiscSpace() {

        Executable e = () -> client.toBlocking().retrieve(HttpRequest.GET("/health"));
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, e);

        assertEquals(HttpStatus.SERVICE_UNAVAILABLE, thrown.getStatus());
        assertTrue(thrown.getResponse().getBody(String.class).orElse("").contains("DOWN"));
    }
}
```
