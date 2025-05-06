**06-05-2025**

# [Error Handling ](https://guides.micronaut.io/latest/micronaut-error-handling-maven-java.html)


``` java

package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Error;
import io.micronaut.http.hateoas.JsonError;
import io.micronaut.http.hateoas.Link;
import io.micronaut.views.ViewsRenderer;

import java.util.Collections;

@Controller("/notfound") // <1>
public class NotFoundController {

    private final ViewsRenderer viewsRenderer;

    public NotFoundController(ViewsRenderer viewsRenderer) { // <2>
        this.viewsRenderer = viewsRenderer;
    }

    @Error(status = HttpStatus.NOT_FOUND, global = true) // <3>
    public HttpResponse notFound(HttpRequest request) {
        if (request.getHeaders()
                .accept()
                .stream()
                .anyMatch(mediaType -> mediaType.getName().contains(MediaType.TEXT_HTML))) {
            return HttpResponse.ok(viewsRenderer.render("notFound", Collections.emptyMap(), request))
                    .contentType(MediaType.TEXT_HTML);
        }

        JsonError error = new JsonError("Page Not Found")
                .link(Link.SELF, Link.of(request.getUri()));

        return HttpResponse.<JsonError>notFound()
                .body(error); // <4>
    }
}
```

1. **@Controller**: A anotação mapeia o path `/notfound`
2. **ViewRenderer**: Bena to renderizar o template
3. **@Error**: A anotação mapeia o método `notFound` para o erro 404
4. **Accept HTTP Header**: caso Content-Type seja HTML, renderiza o template, caso contrário retorna um JSON com o erro

## @Error

validação
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

serialização

``` xml
<dependency>
    <groupId>io.micronaut.serde</groupId>
    <artifactId>micronaut-serde-jackson</artifactId>
    <scope>compile</scope>
</dependency>
```

``` java
package example.micronaut;

import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Body;
import io.micronaut.http.annotation.Consumes;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Error;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Post;
import io.micronaut.http.annotation.Produces;
import io.micronaut.views.View;

import jakarta.validation.ConstraintViolationException;
import jakarta.validation.Valid;
import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

@Controller("/books") // <1>
public class BookController {

    @View("bookscreate") // <2>
    @Get("/create") // <3>
    public Map<String, Object> create() {
        return createModelWithBlankValues();
    }

    @Consumes(MediaType.APPLICATION_FORM_URLENCODED)// <4>
    @Post("/save")// <5>
    public HttpResponse save(@Valid @Body CommandBookSave cmd) { // <6>
        return HttpResponse.ok();
    }

    private Map<String, Object> createModelWithBlankValues() {
        final Map<String, Object> model = new HashMap<>();
        model.put("title", "");
        model.put("pages", "");
        return model;
    }

}
```

1. **@Controller**: A anotação mapeia o path `/books`
2. **@View**: A anotação renderiza o template `bookscreate`
3. **@Get**: A anotação mapeia o método `create` para o verbo HTTP GET
4. **@Consumes**: A anotação mapeia que possuir varios tipos de Content-Type
5. **@Post**: A anotação mapeia o método `save` para o verbo HTTP POST
6. **@Valid**: A anotação valida o objeto `CommandBookSave` e retorna um erro 400 caso não seja válido

``` java

package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;

@Serdeable
public class CommandBookSave {

    @NotBlank
    private String title;

    @Positive
    private int pages;

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public int getPages() {
        return pages;
    }

    public void setPages(int pages) {
        this.pages = pages;
    }
}
```


``` java

...
class BookController {
...
..
    private final MessageSource messageSource;

    public BookController(MessageSource messageSource) { // <1>
        this.messageSource = messageSource;
    }
...
.
    @View("bookscreate")
    @Error(exception = ConstraintViolationException.class)// <2>
    public Map<String, Object> onSavedFailed(HttpRequest request, ConstraintViolationException ex) { // <3>
        final Map<String, Object> model = createModelWithBlankValues();
        model.put("errors", messageSource.violationsMessages(ex.getConstraintViolations()));
        Optional<CommandBookSave> cmd = request.getBody(CommandBookSave.class);
        cmd.ifPresent(bookSave -> populateModel(model, bookSave));
        return model;
    }

    private void populateModel(Map<String, Object> model, CommandBookSave bookSave) {
        model.put("title", bookSave.getTitle());
        model.put("pages", bookSave.getPages());
    }


    private Map<String, Object> createModelWithBlankValues() {
        final Map<String, Object> model = new HashMap<>();
        model.put("title", "");
        model.put("pages", "");
        return model;
    }
..
...
}
```

1. Injeção do `MessageSource` para traduzir as mensagens de erro
2. **@Error**: Você pode especificar uma exceção a ser tratada localmente com a @Error anotação.
3. Você pode acessar o original HttpRequestque acionou a exceção.

``` java
package example.micronaut;

import jakarta.inject.Singleton;
import jakarta.validation.ConstraintViolation;
import jakarta.validation.Path;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

@Singleton
public class MessageSource {

    public List<String> violationsMessages(Set<ConstraintViolation<?>> violations) {
        return violations.stream()
                .map(MessageSource::violationMessage)
                .collect(Collectors.toList());
    }

    private static String violationMessage(ConstraintViolation violation) {
        StringBuilder sb = new StringBuilder();
        Path.Node lastNode = lastNode(violation.getPropertyPath());
        if (lastNode != null) {
            sb.append(lastNode.getName());
            sb.append(" ");
        }
        sb.append(violation.getMessage());
        return sb.toString();
    }

    private static Path.Node lastNode(Path path) {
        Path.Node lastNode = null;
        for (final Path.Node node : path) {
            lastNode = node;
        }
        return lastNode;
    }
}
```

## Manipulando exceções

Forma tradicional que usamos para lidar com exceções. È utilizar o ExpetionsHandlers

``` java
@Controller("/books")
public class BookController {
...
..
.
    @Produces(MediaType.TEXT_PLAIN)
    @Get("/stock/{isbn}")
    public Integer stock(String isbn) {
        throw new OutOfStockException();
    }
}
```

``` java
package example.micronaut;

public class OutOfStockException extends RuntimeException {
}
```

``` java
package example.micronaut;

import io.micronaut.context.annotation.Requires;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.annotation.Produces;
import io.micronaut.http.server.exceptions.ExceptionHandler;
import jakarta.inject.Singleton;

@Produces
@Singleton // <1>
@Requires(classes = {OutOfStockException.class, ExceptionHandler.class}) // <2>
public class OutOfStockExceptionHandler implements ExceptionHandler<OutOfStockException, HttpResponse> {

    @Override
    public HttpResponse handle(HttpRequest request, OutOfStockException exception) {
        return HttpResponse.ok(0); // <3>
    }
}
```

1. **@Singleton**: A anotação registra o handler como um bean singleton
2. **@Requires**: A anotação registra o handler apenas se a classe `OutOfStockException` e `ExceptionHandler` estiverem disponíveis
3. **handle**: O método `handle` é chamado quando a exceção `OutOfStockException` é lançada. O método retorna um `HttpResponse` com o valor 0.


