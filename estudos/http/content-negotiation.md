**09-05-25**

# Content negotiation in a Micronaut Application

## Controler

``` java

package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;
import io.micronaut.views.ModelAndView;

import java.util.Collections;
import java.util.Map;

@Controller
class MessageController {

    @Produces(value = {MediaType.TEXT_HTML, MediaType.APPLICATION_JSON}) //<1>
    @Get
    HttpResponse<?> index(HttpRequest<?> request) { //<2>
        Map<String, Object> model = Collections.singletonMap("message", "Hello World");
        Object body = accepts(request, MediaType.TEXT_HTML_TYPE)
                ? new ModelAndView<>("message.html", model)
                : model;
        return HttpResponse.ok(body);
    }

    private static boolean accepts(HttpRequest<?> request, MediaType mediaType) {
        return request.getHeaders()
                .accept()
                .stream()
                .anyMatch(it -> it.getName().contains(mediaType));
    }
}
```

1. O endpoint aceita tanto `text/html` quanto `application/json` como tipos de mídia.
2. é Injetado o `HttpRequest` para verificar o tipo de mídia aceito pelo cliente. O corpo da resposta é definido como um `ModelAndView` se o tipo de mídia aceito for `text/html`, caso contrário, é definido como um mapa simples.


## Teste

``` java

package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.MediaType;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.junit.jupiter.api.Assertions.assertEquals;

@MicronautTest
class MessageControllerTest {

    @Test
    void contentNegotiation(@Client("/") HttpClient httpClient) {
        BlockingHttpClient client = httpClient.toBlocking();

        String expectedJson = """
                {"message":"Hello World"}""";
        String json = assertDoesNotThrow(() -> client.retrieve(HttpRequest.GET("/")
                .accept(MediaType.APPLICATION_JSON)));
        assertEquals(expectedJson, json);

        String expectedHtml = """
                <!DOCTYPE html>
                <html lang="en">
                <body>
                <h1>Hello World</h1>
                </body>
                </html>
                """;
        String html = assertDoesNotThrow(() -> client.retrieve(HttpRequest.GET("/")
                .accept(MediaType.TEXT_HTML)));
        assertEquals(expectedHtml, html);
    }
}
```


