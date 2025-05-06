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


# [Custom constraint annotation for validation](https://guides.micronaut.io/latest/micronaut-custom-validation-annotation-maven-java.html)

``` java

package example.micronaut;

import io.micronaut.core.annotation.Nullable;
import jakarta.annotation.Nonnull;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * Every country code in the world.
 * @see <a href="https://www.itu.int/dms_pub/itu-t/opb/sp/T-SP-E.164D-11-2011-PDF-E.pdf">LIST OF ITU-T RECOMMENDATION E.164 ASSIGNED COUNTRY CODES</a>
 */
public enum CountryCode {

    AFGHANISTAN("93", "Afghanistan"),
    BRAZIL("55", "Brazil (Federative Republic of)");

    private final String code;
    private final String countryName;

    /**
     * Constructor for countries whose name does not match the enum.
     * @param code country code
     * @param countryName full country name
     */
    CountryCode(String code, String countryName) {
        this.code = code;
        this.countryName = countryName;
    }

    public String getCode() {
        return this.code;
    }

    /**
     * Country name.
     * @return country name
     */
    public String getCountryName() {
        return this.countryName;
    }

    private static final String PLUS_SIGN = "+";

    private static final Map<String, List<CountryCode>> COUNTRYCODESBYCODE =
            Arrays.stream(CountryCode.values())
                    .collect(Collectors.groupingBy(CountryCode::getCode));

    private static final List<String> CODES = COUNTRYCODESBYCODE.keySet().stream()
            .sorted(Comparator.comparing(String::length).reversed()).toList();

    /**
     *
     * @param code Country code
     * @return a List of {@link CountryCode} for a found code or an empty list
     */
    @NotNull
    @Nonnull
    public static List<CountryCode> countryCodesByCode(@Nonnull @NotBlank String code) {
        if (COUNTRYCODESBYCODE.containsKey(code)) {
            return COUNTRYCODESBYCODE.get(code);
        }
        return new ArrayList<>();
    }

    /**
     *
     * @return Country codes ordered from codes of longer length to less length.
     */
    public static List<String> getCodes() {
        return CODES;
    }

    /**
     *
     * @param number Phone number
     * @return the Country code found in the phone number or {@code null} if not found.
     */
    @Nullable
    public static String parseCountryCode(@Nonnull @NotBlank String number) {
        String phone = number.startsWith(PLUS_SIGN) ? number.substring(1) : number;
        for (String code : getCodes()) {
            if (phone.startsWith(code)) {
                return code;
            }
        }
        return null;
    }

    @Override
    public String toString() {
        return this.code;
    }
}
```

### teste
``` java

package example.micronaut;

import org.junit.jupiter.api.Test;

import java.util.Arrays;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNull;
import static org.junit.jupiter.api.Assertions.assertTrue;

class CountryCodeTest {

    @Test
    void preferredNameGetsUsed() {
        String name = CountryCode.YEMEN.getCountryName();
        assertEquals(name, "Yemen (Republic of)");
    }

    @Test
    void defaultNameIsCapitalizedCorrectly() {
        String name = CountryCode.SPAIN.getCountryName();
        assertEquals(name, "Spain");
    }

    @Test
    void toStringReturnsCorrectValue() {
        String code = CountryCode.AMERICAN_SAMOA.toString();
        assertEquals(code, "1");
    }

    @Test
    void countryCodeGetCodesReturnEveryCodeWithLongestCodesFirst() {
        assertTrue(CountryCode.getCodes().get(0).length() > 1);
    }

    @Test
    void countryCodeParseCountryCodeParseCodes() {

        assertNull(CountryCode.parseCountryCode("999999"));

        assertEquals("34",CountryCode.parseCountryCode("34630443322"));
        assertEquals("268",CountryCode.parseCountryCode("2684046441"));
        assertEquals("1",CountryCode.parseCountryCode("+14155552671"));
    }

    @Test
    void countryCodeCountryCodesByCodeReturnAListOfCountryCodeWithTheSameCountryCode() {
        assertTrue(CountryCode.countryCodesByCode("999999").isEmpty());
        assertEquals(CountryCode.countryCodesByCode("1"), Arrays.asList(
                CountryCode.AMERICAN_SAMOA,
                CountryCode.ANGUILLA,
                CountryCode.ANTIGUA_AND_BARBUDA,
                CountryCode.BAHAMAS,
                CountryCode.BARBADOS,
                CountryCode.BERMUDA,
                CountryCode.BRITISH_VIRGIN_ISLANDS,
                CountryCode.CANADA,
                CountryCode.CAYMAN_ISLANDS,
                CountryCode.DOMINICA,
                CountryCode.DOMINICAN_REPUBLIC,
                CountryCode.GRENADA,
                CountryCode.GUAM,
                CountryCode.JAMAICA,
                CountryCode.MONTSERRAT,
                CountryCode.NORTHERN_MARIANA_ISLANDS,
                CountryCode.PUERTO_RICO,
                CountryCode.SAINT_KITTS_AND_NEVIS,
                CountryCode.SAINT_LUCIA,
                CountryCode.SAINT_VINCENT_AND_THE_GRENADINES,
                CountryCode.SINT_MAARTEN,
                CountryCode.TRINIDAD_AND_TOBAGO,
                CountryCode.TURKS_AND_CAICOS_ISLANDS,
                CountryCode.UNITED_STATES));
    }
}
```

## Telefone e164

``` java
package example.micronaut;

import io.micronaut.core.annotation.Nullable;
import io.micronaut.core.util.StringUtils;

/**
 * Utility methods to ease {@link E164} validation.
 */
public final class E164Utils {

    private static final int MAX_NUMBER_OF_DIGITS = 15;
    private static final String PLUS_SIGN = "+";

    private E164Utils() {
    }

    /**
     * @param value phone number
     * @return Whether a phone is E.164 formatted
     */
    public static boolean isValid(@Nullable String value) {
        if (value == null || value.isEmpty()) {
            return false;
        }

        String phone = value.startsWith(PLUS_SIGN) ? value.substring(1) : value;
        if (phone.length() > MAX_NUMBER_OF_DIGITS) {
            return false;
        }
        if (phone.isEmpty()) {
            return false;
        }
        if (!StringUtils.isDigits(phone) || phone.charAt(0) == '0') {
            return false;
        }

        return CountryCode.parseCountryCode(phone) != null;
    }
}
```
### Teste

``` java
package example.micronaut;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;

class E164UtilsTest {

    @ParameterizedTest
    @ValueSource(strings = {
            "+04630443322",
            "+1415555267102345",
            "+1-4155552671",
            ""
    })
    void invalidPhones(String phone) {
        assertFalse(E164Utils.isValid(phone));
    }

    @ParameterizedTest
    @ValueSource(strings = {
            "+14155552671",
            "+442071838750",
            "+55115525632",
            "14155552671",
            "442071838750",
            "55115525632",
            "55115525632",
    })
    void validPhones(String phone) {
        assertTrue(E164Utils.isValid(phone));
    }
}
```

### Anotação

``` java

package example.micronaut;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Repeatable;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * The annotated element must be a E.164 phone number.
 *
 * @see <a href="https://www.itu.int/rec/T-REC-E.164/en">ITU E.164 recommendation</a>
 * @see <a href="https://www.twilio.com/docs/glossary/what-e164">E.614</a>
 */
@Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.PARAMETER, ElementType.TYPE_USE})
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(E164.List.class)
@Documented
@Constraint(validatedBy = {})
public @interface E164 {

    String MESSAGE = "example.micronaut.E164.message";

    /**
     * @return message The error message
     */
    String message() default "{" + MESSAGE + "}";

    /**
     * @return Groups to control the order in which constraints are evaluated,
     * or to perform validation of the partial state of a JavaBean.
     */
    Class<?>[] groups() default {};

    /**
     * @return Payloads used by validation clients to associate some metadata information with a given constraint declaration
     */
    Class<? extends Payload>[] payload() default {};

    /**
     * List annotation.
     */
    @Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.PARAMETER, ElementType.TYPE_USE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @interface List {

        /**
         * @return An array of E164.
         */
        E164[] value();
    }
}
```

## Fabrica da validação


``` java

package example.micronaut;

import io.micronaut.context.annotation.Factory;
import io.micronaut.validation.validator.constraints.ConstraintValidator;
import jakarta.inject.Singleton;

@Factory
class CustomValidationFactory {

    /**
     * @return A {@link ConstraintValidator} implementation of a {@link E164} constraint for type {@link String}.
     */
    @Singleton
    ConstraintValidator<E164, String> e164Validator() {
        return (value, annotationMetadata, context) -> E164Utils.isValid(value);
    }
}
```


## Mensagem de validação

``` java

package example.micronaut;

import io.micronaut.context.StaticMessageSource;
import jakarta.inject.Singleton;

/**
 * Adds validation messages.
 */
@Singleton
public class CustomValidationMessages extends StaticMessageSource {

    public static final String E164_MESSAGE = "must be a phone in E.164 format";
    /**
     * The message suffix to use.
     */
    private static final String MESSAGE_SUFFIX = ".message";

    /**
     * Default constructor to initialize messages.
     * via {@link #addMessage(String, String)}
     */
    public CustomValidationMessages() {
        addMessage(E164.class.getName() + MESSAGE_SUFFIX, E164_MESSAGE);
    }
}
```

## Teste validação

A anotação @Introspected é parte fundamental do que torna o Micronaut eficiente. Ao usá-la corretamente, você aproveita uma das principais vantagens do framework: a capacidade de realizar introspecção de beans sem reflexão, resultando em aplicações mais rápidas e com menor consumo de memória.


Apartir  Micronaut 4.x+, o @Serdeable ja implementa o @Introspected, então não é necessário adicionar a anotação @Introspected.

``` java
package example.micronaut;

import io.micronaut.core.annotation.Introspected;
import io.micronaut.core.annotation.NonNull;

import jakarta.validation.constraints.NotBlank;

@Introspected
public class Contact {

    @E164
    @NotBlank
    @NonNull
    private final String phone;

    public Contact(@NonNull String phone) {
        this.phone = phone;
    }


    @NonNull
    public String getPhone() {
        return phone;
    }
}
```

``` java

package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import jakarta.validation.ConstraintViolation;
import jakarta.validation.Validator;

import java.util.Set;

import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest(startApplication = false)
class ContactTest {

    @Inject
    Validator validator;

    @Test
    void contactValidation() {
        assertTrue(validator.validate(new Contact("+14155552671")).isEmpty());
        Set<ConstraintViolation<Contact>> violationSet = validator.validate(new Contact("+1-4155552671"));
        assertFalse(violationSet.isEmpty());
        String template = "{example.micronaut.E164.message}";
        assertTrue(violationSet.stream().anyMatch(violation ->
                violation.getMessageTemplate().equals(template)
                        && violation.getInvalidValue().equals("+1-4155552671")
                        && violation.getMessage().equals("must be a phone in E.164 format"))
        );
    }
}
```
