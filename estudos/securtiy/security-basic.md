**08-05-25**

# [Micronaut Basic Auth](https://guides.micronaut.io/latest/micronaut-security-basicauth-maven-java.html)


## Authentication provader

``` java
package example.micronaut;

import io.micronaut.core.annotation.NonNull;
import io.micronaut.core.annotation.Nullable;
import io.micronaut.http.HttpRequest;
import io.micronaut.security.authentication.AuthenticationFailureReason;
import io.micronaut.security.authentication.AuthenticationRequest;
import io.micronaut.security.authentication.AuthenticationResponse;
import io.micronaut.security.authentication.provider.HttpRequestAuthenticationProvider;
import jakarta.inject.Singleton;

@Singleton
class AuthenticationProviderUserPassword<B> implements HttpRequestAuthenticationProvider<B> { // <1>

    public AuthenticationResponse authenticate(
            @Nullable HttpRequest<B> httpRequest,
            @NonNull AuthenticationRequest<String, String> authenticationRequest
    ) {
        return authenticationRequest.getIdentity().equals("sherlock") && authenticationRequest.getSecret().equals("password")
                ? AuthenticationResponse.success(authenticationRequest.getIdentity())
                : AuthenticationResponse.failure(AuthenticationFailureReason.CREDENTIALS_DO_NOT_MATCH);
    }
}
```

1. Um Provedor de Autenticação Micronaut implementa a interface io.micronaut.security.authentication.provider.HttpRequestAuthenticationProvider.


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

@Secured(SecurityRule.IS_AUTHENTICATED) // <1>
@Controller
public class HomeController {

    @Produces(MediaType.TEXT_PLAIN)
    @Get
    String index(Principal principal) { // <2>
        return principal.getName();
    }
}
```


1. O controlador é protegido por meio da anotação @Secured(SecurityRule.IS_AUTHENTICATED), que exige autenticação para acessar o controlador.
2. 	Se um usuário for autenticado, a estrutura do Micronaut vinculará o objeto do usuário a um argumento do tipo java.security.Principal(se presente).

## Autentica WWW

``` java
package example.micronaut;

import io.micronaut.context.annotation.Replaces;
import io.micronaut.core.annotation.Nullable;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.MutableHttpResponse;
import io.micronaut.http.server.exceptions.response.ErrorResponseProcessor;
import io.micronaut.security.authentication.AuthorizationException;
import io.micronaut.security.authentication.DefaultAuthorizationExceptionHandler;
import io.micronaut.security.config.RedirectConfiguration;
import io.micronaut.security.config.RedirectService;
import io.micronaut.security.errors.PriorToLoginPersistence;
import jakarta.inject.Singleton;

import static io.micronaut.http.HttpHeaders.WWW_AUTHENTICATE;
import static io.micronaut.http.HttpStatus.FORBIDDEN;
import static io.micronaut.http.HttpStatus.UNAUTHORIZED;

@Singleton
@Replaces(DefaultAuthorizationExceptionHandler.class) // <1>
public class DefaultAuthorizationExceptionHandlerReplacement extends DefaultAuthorizationExceptionHandler {

    public DefaultAuthorizationExceptionHandlerReplacement(
            ErrorResponseProcessor<?> errorResponseProcessor,
            RedirectConfiguration redirectConfiguration,
            RedirectService redirectService,
            @Nullable PriorToLoginPersistence priorToLoginPersistence
    ) {
        super(errorResponseProcessor, redirectConfiguration, redirectService, priorToLoginPersistence);
    }

    @Override
    protected MutableHttpResponse<?> httpResponseWithStatus(HttpRequest request,
                                                            AuthorizationException e) {
        if (e.isForbidden()) {
            return HttpResponse.status(FORBIDDEN);
        }
        return HttpResponse.status(UNAUTHORIZED)
                .header(WWW_AUTHENTICATE, "Basic realm=\"Micronaut Guide\"");
    }
}
```

1. O manipulador de exceção padrão de autorização do Micronaut é substituído para fornecer um cabeçalho WWW-Authenticate personalizado.

## Teste

``` java

package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.client.exceptions.HttpClientResponseException;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.function.Executable;

import static io.micronaut.http.HttpStatus.OK;
import static io.micronaut.http.HttpStatus.UNAUTHORIZED;
import static io.micronaut.http.MediaType.TEXT_PLAIN;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest
public class BasicAuthTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    void verifyHttpBasicAuthWorks() {
        //when: 'Accessing a secured URL without authenticating'
        Executable e = () -> client.toBlocking().exchange(HttpRequest.GET("/").accept(TEXT_PLAIN));

        // then: 'returns unauthorized'
        HttpClientResponseException thrown = assertThrows(HttpClientResponseException.class, e); // <1>
        assertEquals(UNAUTHORIZED, thrown.getStatus());

        assertTrue(thrown.getResponse().getHeaders().contains("WWW-Authenticate"));
        assertEquals("Basic realm=\"Micronaut Guide\"", thrown.getResponse().getHeaders().get("WWW-Authenticate"));

        //when: 'A secured URL is accessed with Basic Auth'
        HttpResponse<String> rsp = client.toBlocking().exchange(HttpRequest.GET("/")
                        .accept(TEXT_PLAIN)
                        .basicAuth("sherlock", "password"), // <2>
                String.class); // <3>
        //then: 'the endpoint can be accessed'
        assertEquals(OK, rsp.getStatus());
        assertEquals("sherlock", rsp.getBody().get()); // <4>
    }
}
```

1. Se você tentar acessar um ponto de extremidade seguro sem autenticação, 401 será retornado
2. 	Ao usar basicAutho método, você preenche o Authorization cabeçalho com pares de ID de usuário:senha, codificados usando Base64.
3. O Micronaut HttpClient simplifica a análise do payload da resposta HTTP para objetos Java. Neste exemplo, analisamos a resposta para String.
4. Use .body()para recuperar a carga útil analisada.

## Use o cliente HTTP micronaut e a autenticação básica

Se quiser acessar um ponto de extremidade seguro, você também pode usar um cliente HTTP Micronaut e fornecer a autenticação básica como o valor do cabeçalho de autorização.

Primeiro crie um @Client com um método homeque aceite um Authorization cabeçalho HTTP.

``` java

package example.micronaut;

import io.micronaut.http.annotation.Consumes;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Header;
import io.micronaut.http.client.annotation.Client;

import static io.micronaut.http.MediaType.TEXT_PLAIN;

@Client("/")
public interface AppClient {

    @Consumes(TEXT_PLAIN)
    @Get
    String home(@Header String authorization);  // <1>
}

```

1. O primeiro caractere do nome do parâmetro é escrito em maiúscula e esse valor ( Authorization) é usado como nome do cabeçalho HTTP. Para alterar o nome do parâmetro, especifique o @Headervalor da anotação.

## Teste

``` java

package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Base64;

import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
public class BasicAuthClientTest {

    @Inject
    AppClient appClient;

    @Test
    void verifyBasicAuthWorks() {
        String creds = basicAuth("sherlock", "password");
        String rsp = appClient.home(creds);
        assertEquals("sherlock", rsp);
    }
    private static String basicAuth(String username, String password) {
        return "Basic " + Base64.getEncoder().encodeToString((username + ":" + password).getBytes());
    }
}
```

