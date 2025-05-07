**07-05-25**

# [Schedule periodic tasks inside your Micronaut applications](https://guides.micronaut.io/latest/micronaut-scheduled-maven-java.html)

Ajusta o log level par info

`src/main/resources/logback.xml`
``` xml
    <logger name="example.micronaut" level="INFO"/>
```


## Criando um Job

```  java

package example.micronaut;

import io.micronaut.scheduling.annotation.Scheduled;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import jakarta.inject.Singleton;
import java.text.SimpleDateFormat;
import java.util.Date;

@Singleton
public class HelloWorldJob {
    private static final Logger LOG = LoggerFactory.getLogger(HelloWorldJob.class);

    @Scheduled(fixedDelay = "10s")
    void executeEveryTen() {
        LOG.info("Simple Job every 10 seconds: {}", new SimpleDateFormat("dd/M/yyyy hh:mm:ss").format(new Date()));
    }

    @Scheduled(fixedDelay = "45s", initialDelay = "5s")
    void executeEveryFourtyFive() {
        LOG.info("Simple Job every 45 seconds: {}", new SimpleDateFormat("dd/M/yyyy hh:mm:ss").format(new Date()));
    }
}
```

1. @Singleton marca a classe como um bean gerenciado pelo Micronaut. Isso significa que o Micronaut irá instanciar e gerenciar o ciclo de vida do bean.
2. Injetando o Logger na classe HelloWorldJob.
3. @Scheduled Create trigger every 10 seconds
4. @Scheduled Create trigger every 45 seconds with an initial delay of 5 seconds.


## Executando o Job

``` bash
... Simple Job every 10 seconds :15/5/2018 12:48:02 #1
... Simple Job every 45 seconds :15/5/2018 12:48:07 #2
... Simple Job every 10 seconds :15/5/2018 12:48:12 #3
... Simple Job every 10 seconds :15/5/2018 12:48:22
... Simple Job every 10 seconds :15/5/2018 12:48:32
... Simple Job every 10 seconds :15/5/2018 12:48:42
... Simple Job every 45 seconds :15/5/2018 12:48:52 #4
... Simple Job every 10 seconds :15/5/2018 12:48:52
```

1. Primeira execução do Job de 10 segundos após o início do aplicativo
2. O job de 45 segundos começa 5 segundos após o início do aplicativo
3. Segunda execução do Job de 10 segundos
4. O job de 45 segundos é executado após 45 segundos

## Logica de negócio em casos de uso dedicado

Embora o exemplo anterior seja válido, normalmente você não quer colocar sua lógica de negócios em um Job. Uma abordagem melhor é criar um bean adicional que o Jobinvoca. Essa abordagem desacopla sua lógica de negócios da lógica de escalonamento. Além disso, facilita os testes e a manutenção. Vejamos um exemplo:

``` java
package example.micronaut;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import jakarta.inject.Singleton;
import java.text.SimpleDateFormat;
import java.util.Date;

@Singleton
public class EmailUseCase {
    private static final Logger LOG = LoggerFactory.getLogger(EmailUseCase.class);

    void send(String user, String message) {
        LOG.info("Sending email to {}: {} at {}", user, message, new SimpleDateFormat("dd/M/yyyy hh:mm:ss").format(new Date()));
    }
}

```

``` java
package example.micronaut;

import io.micronaut.scheduling.annotation.Scheduled;

import jakarta.inject.Singleton;

@Singleton // <1>
public class DailyEmailJob {

    protected final EmailUseCase emailUseCase;

    public DailyEmailJob(EmailUseCase emailUseCase) { // <2>
        this.emailUseCase = emailUseCase;
    }

    @Scheduled(cron = "0 30 4 1/1 * ?") // <3>
    void execute() {
        emailUseCase.send("john.doe@micronaut.example", "Test Message");// <4>
    }
}
```

1. @Singleton marca a classe como um bean gerenciado pelo Micronaut. Isso significa que o Micronaut irá instanciar e gerenciar o ciclo de vida do bean.
2. Injetando o EmailUseCase na classe DailyEmailJob.
3. @Scheduled é criado um cron trigger que executa o job todos os dias às 4:30 da manhã.
4. O método send do EmailUseCase é chamado para enviar um email. O Micronaut irá injetar o EmailUseCase no construtor do DailyEmailJob.

## Agendado um job manualmente

Considere o seguinte cenário. Você quer enviar um e-mail a cada usuário duas horas após o cadastro no seu aplicativo e perguntar sobre suas experiências durante essa primeira interação.

Neste guia, agendaremos um trabalho para ser acionado após um minuto.

Para testá-lo, chamaremos um novo caso de uso nomeado RegisterUseCase duas vezes quando o aplicativo for iniciado.

``` java
package example.micronaut;

import io.micronaut.context.event.ApplicationEventListener;
import io.micronaut.runtime.Micronaut;
import io.micronaut.runtime.server.event.ServerStartupEvent;

import jakarta.inject.Singleton;

@Singleton // <1>
public class Application implements ApplicationEventListener<ServerStartupEvent> { // <2>

    private final RegisterUseCase registerUseCase;

    public Application(RegisterUseCase registerUseCase) { // <3>
        this.registerUseCase = registerUseCase;
    }

    public static void main(String[] args) {
        Micronaut.run(Application.class);
    }

    @Override
    public void onApplicationEvent(ServerStartupEvent event) { // <4>
        try {
            registerUseCase.register("harry@micronaut.example");
            Thread.sleep(20000);
            registerUseCase.register("ron@micronaut.example");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```


1. @Singleton marca a classe como um bean gerenciado pelo Micronaut. Isso significa que o Micronaut irá instanciar e gerenciar o ciclo de vida do bean.
2. Extendendo a classe ApplicationEventListener para ouvir eventos de inicialização do servidor.
3. Injetando o RegisterUseCase na classe Application.
4. onApplicationEventé invocado quando o aplicativo é iniciado. Inscrever-se em um evento é tão fácil quanto implementar ApplicationEventListener.


Criar uma tarefa executável EmailTask.java

``` java

package example.micronaut;

public class EmailTask implements Runnable {

    private String email;
    private String message;
    private EmailUseCase emailUseCase;

    public EmailTask(EmailUseCase emailUseCase, String email, String message) {
        this.email = email;
        this.message = message;
        this.emailUseCase = emailUseCase;
    }

    @Override
    public void run() {
        emailUseCase.send(email, message);
    }

}
```

``` java
package example.micronaut;

import io.micronaut.scheduling.TaskExecutors;
import io.micronaut.scheduling.TaskScheduler;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import jakarta.inject.Named;
import jakarta.inject.Singleton;
import java.text.SimpleDateFormat;
import java.time.Duration;
import java.util.Date;

@Singleton
public class RegisterUseCase {

    private static final Logger LOG = LoggerFactory.getLogger(RegisterUseCase.class);

    protected final TaskScheduler taskScheduler;
    protected final EmailUseCase emailUseCase;

    public RegisterUseCase(EmailUseCase emailUseCase,
                           @Named(TaskExecutors.SCHEDULED) TaskScheduler taskScheduler) {
        this.emailUseCase = emailUseCase;
        this.taskScheduler = taskScheduler;
    }

    public void register(String email) {
        LOG.info("saving {} at {}", email, new SimpleDateFormat("dd/M/yyyy hh:mm:ss").format(new Date()));
        scheduleFollowupEmail(email, "Welcome to the Micronaut framework");
    }

    private void scheduleFollowupEmail(String email, String message) {
        EmailTask task = new EmailTask(emailUseCase, email, message);
        taskScheduler.schedule(Duration.ofMinutes(1), task);
    }
}
```


1. Injetando o TaskScheduler no construtor do RegisterUseCase.
2. Injetar o TaskScheduler Bean com o nome SCHEDULED.
3. Criar uma Runnable tarefa
4. Agende a tarefa para ser executada daqui a um minuto


## Resumo
Durante este guia, aprendemos como configurar Jobs usando a @Scheduled anotação usando fixedDelay, initialDelay e cron, bem como configurar Jobs manualmente com TaskScheduler tarefas e .
