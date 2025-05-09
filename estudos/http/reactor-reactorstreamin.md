**09-05-25**

# Download a big file with StreamingHttpClient


``` groovy
implementation("io.micronaut.reactor:micronaut-reactor-http-client")
```

``` java
package example.micronaut;

import io.micronaut.context.exceptions.ConfigurationException;
import io.micronaut.core.io.buffer.ByteBuffer;
import io.micronaut.core.io.buffer.ReferenceCounted;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.reactor.http.client.ReactorStreamingHttpClient;
import jakarta.annotation.PreDestroy;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import reactor.core.publisher.Flux;

import java.net.MalformedURLException;
import java.net.URI;
import java.net.URL;

@Controller // <1>
class HomeController implements AutoCloseable {
    private static final Logger LOG = LoggerFactory.getLogger(HomeController.class);
    private static final URI DEFAULT_URI = URI.create("https://guides.micronaut.io/micronaut5K.png");

    private final ReactorStreamingHttpClient reactorStreamingHttpClient;

    HomeController() {
        String urlStr = "https://guides.micronaut.io/";
        URL url;
        try {
            url = new URL(urlStr);
        } catch (MalformedURLException e) {
            throw new ConfigurationException("malformed URL" + urlStr);
        }
        this.reactorStreamingHttpClient = ReactorStreamingHttpClient.create(url); // <2>
    }

    @Get // <3>
    Flux<ByteBuffer<?>> download() {
        HttpRequest<?> request = HttpRequest.GET(DEFAULT_URI);
        return reactorStreamingHttpClient.dataStream(request).doOnNext(bb -> {
            if (bb instanceof ReferenceCounted rc) {
                rc.retain();
            }
        }); // <4>
    }

    @PreDestroy // <5>
    @Override
    public void close() {
        if (reactorStreamingHttpClient != null) {
            reactorStreamingHttpClient.close();
        }
    }
}
```

1. A classe é definida como um controlador com a anotação @Controller mapeada para o caminho /.
2. ReactorStreamingHttpClient é uma variação da StreamingHttpClient interface do Projeto Reactor, que estende o HttpClient para oferecer suporte ao streaming de respostas.
A anotação @Get mapeia o método para uma solicitação HTTP GET.
3. O dataStream método emite instâncias de ByteBuffer. Essas instâncias são coletadas como lixo após a emissão de cada bloco. Para propagar os blocos para o fluxo de resposta, eles precisam ser retidos chamando o retain() método. O framework chamará release() esses blocos coletados como lixo assim que cada um for gravado na resposta.
4. Para invocar um método quando o bean é destruído, use a jakarta.annotation.PreDestroyanotação.



## Teste

``` java
package example.micronaut;

import io.micronaut.core.io.ResourceLoader;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.DisabledInNativeImage;

import java.io.IOException;
import java.net.URISyntaxException;
import java.net.URL;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

@MicronautTest
class HomeControllerTest {

    private final HexFormat hexFormat = HexFormat.of();

    @DisabledInNativeImage
    @Test
    void downloadFile(@Client("/") HttpClient httpClient,
                      ResourceLoader resourceLoader)
            throws URISyntaxException, IOException, NoSuchAlgorithmException {
        Optional<URL> resource = resourceLoader.getResource("micronaut5K.png");
        assertTrue(resource.isPresent());
        byte[] expectedByteArray = Files.readAllBytes(Path.of(resource.get().toURI()));

        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] expectedEncodedHash = digest.digest(expectedByteArray);
        String expected = hexFormat.formatHex(expectedEncodedHash);

        BlockingHttpClient client = httpClient.toBlocking();
        HttpResponse<byte[]> resp = assertDoesNotThrow(() -> client.exchange(HttpRequest.GET("/"), byte[].class));
        byte[] responseBytes = resp.body();

        byte[] responseEncodedHash = digest.digest(responseBytes);
        String response = hexFormat.formatHex(responseEncodedHash);
        assertEquals(expected, response);
    }
}
```
