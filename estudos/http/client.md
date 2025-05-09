**09-05-25**

# [Micronaut HTTP Client](https://guides.micronaut.io/latest/micronaut-http-client-maven-java.html)

``` xml
<dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-http-client</artifactId>
    <scope>compile</scope>
</dependency>
 ```

``` xml
<dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-http-client-jdk</artifactId>
    <scope>compile</scope>
</dependency>
```


## Github


``` java

package example.micronaut;

import io.micronaut.serde.annotation.Serdeable;

@Serdeable
public record GithubRelease(String name, String url) {
}
```


## Configuração

``` properties
github.organization=micronaut-projects
github.repo=micronaut-core
```

``` java
package example.micronaut;

import io.micronaut.context.annotation.ConfigurationProperties;
import io.micronaut.context.annotation.Requires;
import io.micronaut.core.annotation.Nullable;

@ConfigurationProperties(GithubConfiguration.PREFIX)
@Requires(property = GithubConfiguration.PREFIX)
public record GithubConfiguration(String organization,
                                  String repo,
                                  @Nullable String username,
                                  @Nullable String token) {
    public static final String PREFIX = "github";
}
```

## json codec

``` java
micronaut.codec.json.additional-types[0]=application/vnd.github.v3+json
```

## Http cliente Service

``` java
micronaut.http.services.github.url=https://api.github.com
```

## Low level client

``` java
package example.micronaut;

import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.client.HttpClient;
import io.micronaut.http.client.annotation.Client;
import io.micronaut.http.uri.UriBuilder;
import jakarta.inject.Singleton;
import org.reactivestreams.Publisher;

import java.net.URI;
import java.util.List;

import static io.micronaut.http.HttpHeaders.ACCEPT;
import static io.micronaut.http.HttpHeaders.USER_AGENT;

@Singleton // <1>
public class GithubLowLevelClient {

    private final HttpClient httpClient; // <2>
    private final URI uri;

    public GithubLowLevelClient(@Client(id = "github") HttpClient httpClient,
                                GithubConfiguration configuration) { // <3>
        this.httpClient = httpClient;
        uri = UriBuilder.of("/repos")
                .path(configuration.organization())
                .path(configuration.repo())
                .path("releases")
                .build();
    }

    Publisher<List<GithubRelease>> fetchReleases() {
        HttpRequest<?> req = HttpRequest.GET(uri) // <4>
                .header(USER_AGENT, "Micronaut HTTP Client") // <5>
                .header(ACCEPT, "application/vnd.github.v3+json, application/json"); // <6>
        return httpClient.retrieve(req, Argument.listOf(GithubRelease.class)); // <7>
    }
}
```
1. O cliente é um singleton.
2. O cliente é injetado no construtor.
3. O URI é construído com base na configuração.
4. 	Creating HTTP Requests is easy thanks to the Micronaut framework fluid API.
5. 	GitHub API requires to set the User-Agent header.
6. 	GitHub encourages to explicitly request the version 3 via the Accept header. With @Header, you add the Accept: application/vnd.github.v3+json HTTP header to every request.
7. Use retrieve to perform an HTTP request for the given request object and convert the full HTTP response’s body into the specified type. e.g. List<GithubRelease>.

## Declarative Client

``` java

package example.micronaut;

import io.micronaut.core.async.annotation.SingleResult;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Header;
import io.micronaut.http.client.annotation.Client;
import org.reactivestreams.Publisher;

import java.util.List;

import static io.micronaut.http.HttpHeaders.ACCEPT;
import static io.micronaut.http.HttpHeaders.USER_AGENT;

@Client(id = "github")
@Header(name = USER_AGENT, value = "Micronaut HTTP Client")
@Header(name = ACCEPT, value = "application/vnd.github.v3+json, application/json")
public interface GithubApiClient {

    @Get("/repos/${github.organization}/${github.repo}/releases")
    @SingleResult
    Publisher<List<GithubRelease>> fetchReleases();
}
```


1. URL do serviço remoto
2. A API do GitHub requer a definição do User-Agentcabeçalho.
3. O GitHub recomenda solicitar explicitamente a versão 3 por meio do Acceptcabeçalho. Com @Header, você adiciona o Accept: application/vnd.github.v3+jsoncabeçalho HTTP a cada solicitação.
4. Você pode usar a interpolação de parâmetros de configuração ao definir o caminho do ponto de extremidade GET.
5. Anotação para descrever que uma API emite um único resultado mesmo que o tipo de retorno seja um org.reactivestreams.Publisher.
6. Você pode retornar qualquer tipo reativo de qualquer implementação (RxJava, Reactor…​), mas é melhor usar as interfaces públicas do Reactive Streams como Publisher.


## Controlador

``` java
package example.micronaut;

import io.micronaut.core.async.annotation.SingleResult;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import org.reactivestreams.Publisher;
import java.util.List;

@Controller("/github")
public class GithubController {

    private final GithubLowLevelClient githubLowLevelClient;
    private final GithubApiClient githubApiClient;
    public GithubController(GithubLowLevelClient githubLowLevelClient,
                            GithubApiClient githubApiClient) {
        this.githubLowLevelClient = githubLowLevelClient;
        this.githubApiClient = githubApiClient;
    }

    @Get("/releases-lowlevel")
    @SingleResult
    Publisher<List<GithubRelease>> releasesWithLowLevelClient() {
        return githubLowLevelClient.fetchReleases();
    }

    @Get("/releases")
    @SingleResult
    Publisher<List<GithubRelease>> fetchReleases() {
        return githubApiClient.fetchReleases();
    }
}
```

1. A classe é definida como um controlador com a anotação @Controller mapeada para o caminho /github.
2. Injetar beans por meio de injeção de construtor.
3. A anotação @Get mapeia o indexmétodo para todas as solicitações que usam um HTTP GET
Anotação para descrever que uma API emite um único resultado mesmo que o tipo de retorno seja um org.reactivestreams.Publisher.
A anotação @Get mapeia o fetchReleasesmétodo para uma solicitação HTTP GET em /releases.

## Teste

``` json
[{"url":"https://api.github.com/repos/micronaut-projects/micronaut-core/releases/91622014","assets_url":"https://api.github.com/repos/micronaut-projects/micronaut-core/releases/91622014/assets","upload_url":"https://uploads.github.com/repos/micronaut-projects/micronaut-core/releases/91622014/assets{?name,label}","html_url":"https://github.com/micronaut-projects/micronaut-core/releases/tag/v3.8.4","id":91622014,"author":{"login":"sdelamo","id":864788,"node_id":"MDQ6VXNlcjg2NDc4OA==","avatar_url":"https://avatars.githubusercontent.com/u/864788?v=4","gravatar_id":"","url":"https://api.github.com/users/sdelamo","html_url":"https://github.com/sdelamo","followers_url":"https://api.github.com/users/sdelamo/followers","following_url":"https://api.github.com/users/sdelamo/following{/other_user}","gists_url":"https://api.github.com/users/sdelamo/gists{/gist_id}","starred_url":"https://api.github.com/users/sdelamo/starred{/owner}{/repo}","subscriptions_url":"https://api.github.com/users/sdelamo/subscriptions","organizations_url":"https://api.github.com/users/sdelamo/orgs","repos_url":"https://api.github.com/users/sdelamo/repos","events_url":"https://api.github.com/users/sdelamo/events{/privacy}","received_events_url":"https://api.github.com/users/sdelamo/received_events","type":"User","site_admin":false},"node_id":"RE_kwDOB2eaPM4Fdgp-","tag_name":"v3.8.4","target_commitish":"3.8.x","name":"Micronaut Framework 3.8.4","draft":false,"prerelease":false,"created_at":"2023-02-07T15:53:03Z","published_at":"2023-02-07T15:53:05Z","assets":[{"url":"https://api.github.com/repos/micronaut-projects/micronaut-core/releases/assets/94667997","id":94667997,"node_id":"RA_kwDOB2eaPM4FpITd","name":"artifacts.zip","label":"","uploader":{"login":"github-actions[bot]","id":41898282,"node_id":"MDM6Qm90NDE4OTgyODI=","avatar_url":"https://avatars.githubusercontent.com/in/15368?v=4","gravatar_id":"","url":"https://api.github.com/users/github-actions%5Bbot%5D","html_url":"https://github.com/apps/github-actions","followers_url":"https://api.github.com/users/github-actions%5Bbot%5D/followers","following_url":"https://api.github.com/users/github-actions%5Bbot%5D/following{/other_user}","gists_url":"https://api.github.com/users/github-actions%5Bbot%5D/gists{/gist_id}","starred_url":"https://api.github.com/users/github-actions%5Bbot%5D/starred{/owner}{/repo}","subscriptions_url":"https://api.github.com/users/github-actions%5Bbot%5D/subscriptions","organizations_url":"https://api.github.com/users/github-actions%5Bbot%5D/orgs","repos_url":"https://api.github.com/users/github-actions%5Bbot%5D/repos","events_url":"https://api.github.com/users/github-actions%5Bbot%5D/events{/privacy}","received_events_url":"https://api.github.com/users/github-actions%5Bbot%5D/received_events","type":"Bot","site_admin":false},"content_type":"application/zip","state":"uploaded","size":28063979,"download_count":33,"created_at":"2023-02-07T16:18:05Z","updated_at":"2023-02-07T16:18:06Z","browser_download_url":"https://github.com/micronaut-projects/micronaut-core/releases/download/v3.8.4/artifacts.zip"},{"url":"https://api.github.com/repos/micronaut-projects/micronaut-core/releases/assets/94669478","id":94669478,"node_id":"RA_kwDOB2eaPM4FpIqm","name":"multiple.intoto.jsonl","label":"","uploader":{"login":"github-actions[bot]","id":41898282,"node_id":"MDM6Qm90NDE4OTgyODI=","avatar_url":"https://avatars.githubusercontent.com/in/15368?v=4","gravatar_id":"","url":"https://api.github.com/users/github-actions%5Bbot%5D","html_url":"https://github.com/apps/github-actions","followers_url":"https://api.github.com/users/github-actions%5Bbot%5D/followers","following_url":"https://api.github.com/users/github-actions%5Bbot%5D/following{/other_user}","gists_url":"https://api.github.com/users/github-actions%5Bbot%5D/gists{/gist_id}","starred_url":"https://api.github.com/users/github-actions%5Bbot%5D/starred{/owner}{/repo}","subscriptions_url":"https://api.github.com/users/github-actions%5Bbot%5D/subscriptions","organizations_url":"https://api.github.com/users/github-actions%5Bbot%5D/orgs","repos_url":"https://api.github.com/users/github-actions%5Bbot%5D/repos","events_url":"https://api.github.com/users/github-actions%5Bbot%5D/events{/privacy}","received_events_url":"https://api.github.com/users/github-actions%5Bbot%5D/received_events","type":"Bot","site_admin":false},"content_type":"application/octet-stream","state":"uploaded","size":65612,"download_count":34,"created_at":"2023-02-07T16:32:49Z","updated_at":"2023-02-07T16:32:49Z","browser_download_url":"https://github.com/micronaut-projects/micronaut-core/releases/download/v3.8.4/multiple.intoto.jsonl"}],"tarball_url":"https://api.github.com/repos/micronaut-projects/micronaut-core/tarball/v3.8.4","zipball_url":"https://api.github.com/repos/micronaut-projects/micronaut-core/zipball/v3.8.4","body":"<!-- Release notes generated using configuration in .github/release.yml at 3.8.x -->\r\n\r\n\r\n## What's Changed\r\n\r\n### Bugs 🐛\r\n* Allow programmatic logback config if xml config is absent (#8674)\r\n* Tweak uri to account for WebLogic/Windows URIs by @mattmoss in https://github.com/micronaut-projects/micronaut-core/pull/8704\r\n\r\n### Docs 📖\r\n* Docs on self-signed cert setup (#8684)\r\n\r\n### Dependency Upgrades 🚀\r\n\r\n* Micronaut Test to 3.8.2 (#8728)\r\n* Micronaut OpenAPI to 4.8.3 (#8724)\r\n* Micronaut Data to 3.9.6 (#8711)\r\n* Micronaut Rabbit to 3.4.1 (#8709)\r\n* Micronaut Azure to 3.7.1 (#8682)\r\n* Micronaut Micrometer to 4.7.2 (#8681)\r\n\r\n### Tests ✅\r\n\r\n* Require docker for test (#8698)\r\n* Add octet stream serialization to the TCK (#8712)\r\n\r\n**Full Changelog**: https://github.com/micronaut-projects/micronaut-core/compare/v3.8.3...v3.8.4","mentions_count":1}]
```

``` java
package example.micronaut;

import io.micronaut.context.ApplicationContext;
import io.micronaut.context.annotation.Requires;
import io.micronaut.core.io.ResourceLoader;
import io.micronaut.core.type.Argument;
import io.micronaut.http.HttpRequest;
import io.micronaut.http.HttpResponse;
import io.micronaut.http.HttpStatus;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;
import io.micronaut.http.client.BlockingHttpClient;
import io.micronaut.http.client.HttpClient;
import io.micronaut.runtime.server.EmbeddedServer;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.regex.Pattern;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static java.nio.charset.StandardCharsets.UTF_8;

class GithubControllerTest {
    private static Pattern MICRONAUT_RELEASE =
            Pattern.compile("Micronaut (Core |Framework )?v?\\d+.\\d+.\\d+( (RC|M)\\d)?");

    @Test
    void verifyGithubReleasesCanBeFetchedWithLowLevelHttpClient() {
        EmbeddedServer github = ApplicationContext.run(EmbeddedServer.class,
                Map.of("micronaut.codec.json.additional-types", "application/vnd.github.v3+json",
                        "spec.name", "GithubControllerTest"));
        EmbeddedServer embeddedServer = ApplicationContext.run(EmbeddedServer.class,
                Collections.singletonMap("micronaut.http.services.github.url",
                        "http://localhost:" + github.getPort()));
        HttpClient httpClient = embeddedServer.getApplicationContext()
                .createBean(HttpClient.class, embeddedServer.getURL());
        BlockingHttpClient client = httpClient.toBlocking();
        assertReleases(client, "/github/releases");
        assertReleases(client, "/github/releases-lowlevel");
        httpClient.close();
        embeddedServer.close();
        github.close();
    }

    private static void assertReleases(BlockingHttpClient client, String path) {
        HttpRequest<Object> request = HttpRequest.GET(path);

        HttpResponse<List<GithubRelease>> rsp = client.exchange(request,
                Argument.listOf(GithubRelease.class));

        assertEquals(HttpStatus.OK, rsp.getStatus());
        assertReleases(rsp.body());
    }

    private static void assertReleases(List<GithubRelease> releases) {
        assertNotNull(releases);
        assertTrue(releases.stream()
                .map(GithubRelease::name)
                .allMatch(name -> MICRONAUT_RELEASE.matcher(name)
                        .find()));
    }

    @Requires(property = "spec.name", value = "GithubControllerTest")
    @Controller
    static class GithubReleases {
        private final ResourceLoader resourceLoader;
        GithubReleases(ResourceLoader resourceLoader) {
            this.resourceLoader = resourceLoader;
        }

        @Produces("application/vnd.github.v3+json")
        @Get("/repos/micronaut-projects/micronaut-core/releases")
        Optional<String> coreReleases() {
            return resourceLoader.getResourceAsStream("releases.json")
                    .flatMap(inputStream -> {
                        try {
                            return Optional.of(new String(inputStream.readAllBytes(), UTF_8));
                        } catch (IOException e) {
                            return Optional.empty();
                        }
                    });
        }
    }
}
```

