## [Kafka e o Micronaut Framework - Aplicações Orientadas a Eventos](https://guides.micronaut.io/latest/micronaut-kafka-maven-java.html)


O guia mostra a criação de dois aplicativos micronuat um vai ser o consumidor o outro o produtor.

ele vai usar um filtro http para enviar mensagens para o kafka.

## Criando o projeto

```bash
mn create-app --features=kafka,reactor,graalvm,serialization-jackson example.micronaut.books --build=maven --lang=java
```

``` xml

<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <version>4.2.2</version>
    <scope>test</scope>
</dependency>

```



## Producer

``` java
package example.micronaut;

import io.micronaut.configuration.kafka.annotation.KafkaClient;
import io.micronaut.configuration.kafka.annotation.Topic;
import org.reactivestreams.Publisher;
import reactor.core.publisher.Mono;

@KafkaClient
public interface AnalyticsClient {

    @Topic("analytics")
    Mono<Book> updateAnalytics(Book book);
}
```


## Consumer

``` java
package example.micronaut;

import io.micronaut.configuration.kafka.annotation.KafkaListener;
import io.micronaut.configuration.kafka.annotation.Topic;
import io.micronaut.context.annotation.Requires;
import io.micronaut.context.env.Environment;

@Requires(notEnv = Environment.TEST)
@KafkaListener
public class AnalyticsListener {

    private final AnalyticsService analyticsService;

    public AnalyticsListener(AnalyticsService analyticsService) {
        this.analyticsService = analyticsService;
    }

    @Topic("analytics")
    public void updateAnalytics(Book book) {
        analyticsService.updateBookAnalytics(book);
    }
}
```
