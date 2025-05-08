**08-05-25**

``` bash
mn create-app example.micronaut.micronautguide \
    --features=yaml,security,graalvm \
    --build=maven \
    --lang=java \
    --test=junit
```


## Api Key

``` java

package example.micronaut;

import io.micronaut.context.annotation.EachProperty;
import jakarta.annotation.Nonnull;

@EachProperty("api-keys")
public interface ApiKeyConfiguration {

    @Nonnull
    String getName();

    @Nonnull
    String getKey();
}

```


``` java

package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import jakarta.validation.constraints.NotBlank;
import java.security.Principal;
import java.util.Optional;

@FunctionalInterface
public interface ApiKeyRepository {
    @NonNull
    Optional<Principal> findByApiKey(@NonNull @NotBlank String apiKey);
}

```

1. Uma interface com uma declaração de método abstrato é conhecida como interface funcional. O compilador verifica se todas as interfaces anotadas com @FunctionInterface realmente contêm um e apenas um método abstrato.

``` java
package example.micronaut;

import jakarta.inject.Singleton;
import io.micronaut.core.annotation.NonNull;
import jakarta.validation.constraints.NotBlank;
import java.security.Principal;
import java.util.HashMap;
import java.util.Map;
import java.util.Optional;
import java.util.List;

@Singleton
class ApiKeyRepositoryImpl implements ApiKeyRepository {

    private final Map<String, Principal> keys;

    ApiKeyRepositoryImpl(List<ApiKeyConfiguration> apiKeys) {
        keys = new HashMap<>();
        for (ApiKeyConfiguration configuration : apiKeys) {
            keys.put(configuration.getKey(), configuration::getName);
        }
        System.out.println("Keys #" + keys.keySet().size());
    }

    @Override
    @NonNull
    public Optional<Principal> findByApiKey(@NonNull @NotBlank String apiKey) {
        return Optional.ofNullable(keys.get(apiKey));
    }
}
```

## Leitor de tokens

``` java
package example.micronaut;

import io.micronaut.security.token.reader.HttpHeaderTokenReader;
import jakarta.inject.Singleton;

@Singleton
public class ApiKeyTokenReader extends HttpHeaderTokenReader {
    private static final String X_API_TOKEN = "X-API-KEY";

    @Override
    protected String getPrefix() {
        return null;
    }

    @Override
    protected String getHeaderName() {
        return X_API_TOKEN;
    }
}
```


## Validador de tokens

``` java
package example.micronaut;

import io.micronaut.core.async.publisher.Publishers;
import io.micronaut.http.HttpRequest;
import io.micronaut.security.authentication.Authentication;
import io.micronaut.security.token.validator.TokenValidator;
import jakarta.inject.Singleton;
import org.reactivestreams.Publisher;

@Singleton
class ApiKeyTokenValidator implements TokenValidator<HttpRequest<?>>  {

    private final ApiKeyRepository apiKeyRepository;

    ApiKeyTokenValidator(ApiKeyRepository apiKeyRepository) {
        this.apiKeyRepository = apiKeyRepository;
    }

    @Override
    public Publisher<Authentication> validateToken(String token, HttpRequest<?> request) {
        if (request == null || !request.getPath().startsWith("/api")) {
            return Publishers.empty();
        }
        return apiKeyRepository.findByApiKey(token)
                .map(principal -> Authentication.build(principal.getName()))
                .map(Publishers::just).orElseGet(Publishers::empty);
    }
}

```

## Controlador

``` java

package example.micronaut;

import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;
import io.micronaut.security.annotation.Secured;
import io.micronaut.security.rules.SecurityRule;

import java.security.Principal;

@Controller("/api")
class ApiController {

    @Produces(MediaType.TEXT_PLAIN)
    @Get
    @Secured(SecurityRule.IS_AUTHENTICATED)
    String index(Principal principal) {
        return "Hello " + principal.getName();
    }
}

```
``` java

package example.micronaut;

import io.micronaut.context.annotation.Property;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.MediaType;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.function.Executable;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

@Property(name = "api-keys.companyA.key", value = "XXX")
@Property(name = "api-keys.companyA.name", value = "John")
@Property(name = "api-keys.companyB.key", value = "YYY")
@Property(name = "api-keys.companyB.name", value = "Paul")
@MicronautTest
class ApiControllerTest {
    @Inject
    @Client("/")
    HttpClient httpClient;

    @Test
    void apiIsSecured() {
        BlockingHttpClient client = httpClient.toBlocking();
        HttpRequest<?> request = HttpRequest.GET("/api").accept(MediaType.TEXT_PLAIN);
        Executable e = () -> client.exchange(request);
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, e);
        assertEquals(HttpStatus.UNAUTHORIZED, thrown.getStatus());
    }

    @Test
    void apiNotAccessibleIfWrongKey() {
        BlockingHttpClient client = httpClient.toBlocking();
        HttpRequest<?> request = createRequest("ZZZ");
        Executable e = () -> client.exchange(request);
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, e);
        assertEquals(HttpStatus.UNAUTHORIZED, thrown.getStatus());
    }

    @Test
    void apiIsAccessibleWithAnApiKey() {
        BlockingHttpClient client = httpClient.toBlocking();

        HttpResponse<String> response = assertDoesNotThrow(() -> client.exchange(createRequest("XXX"), String.class));
        assertEquals(HttpStatus.OK, response.getStatus());
        Optional<String> body = response.getBody();
        assertTrue(body.isPresent());
        assertEquals("Hello John", body.get());

        response = assertDoesNotThrow(() -> client.exchange(createRequest("YYY"), String.class));
        assertEquals(HttpStatus.OK, response.getStatus());
        body = response.getBody();
        assertTrue(body.isPresent());
        assertEquals("Hello Paul", body.get());
    }

    private static HttpRequest<?> createRequest(String apiKey) {
        return HttpRequest.GET("/api")
                .accept(MediaType.TEXT_PLAIN)
                .header("X-API-KEY", apiKey);
    }
}
```
