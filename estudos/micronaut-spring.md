**05-05-25**

O Guia Mostra uam comparação da aplicação class do spring bott vs micronaut.

## spring boot aplication class

```java

package example.micronaut;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

## micronaut aplication class

```java

package example.micronaut;

import io.micronaut.runtime.Micronaut;

public class Application {

    public static void main(String[] args) {
        Micronaut.run(Application.class, args);
    }
}

```


A sintaxe do micronaut é mais simples, não precisa de anotações como @SpringBootApplication, o micronaut já sabe que a classe é a aplicação principal.

## Manually define a bean - spring boot

## Implementação

```java

package example.micronaut;

public interface Greeter {
    String greet();
}

```

``` java

package example.micronaut;

public class HelloGreeter implements Greeter {
    @Override
    public String greet() {
        return "Hello";
    }
}

```
## Spring Boot



spring, @Configuration criar um bean

``` java

package example.micronaut;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class GreeterFactory {

    @Bean
    Greeter helloGreeter() {
        return new HelloGreeter();
    }
}
```

### Test

``` java

package example.micronaut;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
class GreeterTest {

    @Autowired
    Greeter greeter;

    @Test
    void helloGreeterIsInjectedAsBeanOfTypeGreeter() {
        assertNotNull(greeter);
        assertEquals("Hello", greeter.greet());
    }

}
```

1. **@SpringBootTest** - inicializa o contexto do spring e injeta o bean.
2. **@Autowired** - injeta o bean do spring.

## Micronaut @Factory

``` java

package example.micronaut;

import io.micronaut.context.annotation.Factory;
import io.micronaut.context.annotation.Bean;

@Factory
class GreeterFactory {
    @Bean
    Greeter helloGreeter() {
        return new HelloGreeter();
    }
}
```

1. **@Factory** - criar um bean do micronaut.

### Micronaut Test

``` java

package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@MicronautTest
class GreeterTest {

    @Inject
    Greeter greeter;

    @Test
    void helloGreeterIsInjectedAsBeanOfTypeGreeter() {
        assertNotNull(greeter);
        assertEquals("Hello", greeter.greet());
    }

}
```

1. **@MicronautTest** - inicializa o contexto do micronaut e injeta o bean.
2. **@Inject** - injeta o bean do micronaut.

## Conclusão

Esse guia ilustra que a Api @Configuration do spring e factory em ambos frameworks são muito similares, o que muda é a anotação e a forma de injetar o bean. de modo simples, vamos podemos utilizar o @Factory do micronaut para criar um bean e o @Inject para injetar o bean.


## Referências

- [Manually define a Bean](https://guides.micronaut.io/latest/spring-boot-to-micronaut-at-configuration-at-bean-at-factory-maven-java.html)


# mark a classe como um bean

## Criação da interface

``` java
package example.micronaut;

public interface Greeter {
    String greet();
}
```


## Spring boot

Nos spring utilizamos a anotação @Component para marcar a classe como um bean.

``` java

package example.micronaut;
import org.springframework.stereotype.Component;

@Component
public class HelloGreeter implements Greeter {
    @Override
    public String greet() {
        return "Hello";
    }
}
```

### Test

``` java

package example.micronaut;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
class GreeterTest {

    @Autowired
    Greeter greeter;

    @Test
    void helloGreeterIsInjectedAsBeanOfTypeGreeter() {
        assertNotNull(greeter);
        assertEquals("Hello", greeter.greet());
    }

}
```

## Micronaut

No micronaut utilizamos a anotação @Singleton para marcar a classe como um bean.

``` java

package example.micronaut;

import jakarta.inject.Singleton;

@Singleton
public class HelloGreeter implements Greeter {
    @Override
    public String greet() {
        return "Hello";
    }
}

```


1. **@Singleton** - marca a classe como um bean do micronaut, jakata.inject.Singleton

### Test

``` java

package example.micronaut;

import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@MicronautTest
class GreeterTest {

    @Inject
    Greeter greeter;

    @Test
    void helloGreeterIsInjectedAsBeanOfTypeGreeter() {
        assertNotNull(greeter);
        assertEquals("Hello", greeter.greet());
    }

}
```

## Conclusão

O guia mostra a diferença entre a geração de beans do spring e do micronaut, enquanto o spring utilizar a anotação @Componente que é uma anotação constomizada para o spring, o micronaut utiliza a anotação @Singleton que é uma anotação do jakarta.inject.Singleton.

> O micronaut gerar is informações necessárias para preencher os pontos de injeção no momento da compilação.


> 	O Spring depende da varredura de classpath para encontrar classes anotadas com @Component. No aplicativo Spring Boot, HelloGreeter é detectado porque o aplicativo contém uma classe Applicationcom a @SpringBootApplication anotação. A @SpringBootApplication anotação aplica a @ComponentScan anotação, que instrui o Spring a varrer o pacote onde a Applicationclasse está localizada e seus subpacotes.
