**04-05-25**

# Micronaut

## O que é Micronaut

Micronaut é um framework moderno, baseado em JVM e full stack, projetado para criar aplicativos JVM modulares e facilmente testáveis, com suporte para Java, Kotlin e Groovy.

O Micronaut foi criado pela equipe que também trabalhou no Grails.

O Micronaut visa fornecer todas as ferramentas necessárias para construir aplicativos JVM:

- Injeção de dependência (IoC)
- Programação orientada a aspectos (AOP)
- Padrões sensatos e configuração automática

Com o Micronaut, você pode criar aplicativos orientados a mensagens, aplicativos de linha de comando, servidores HTTP e fornece:

- Configuração distribuída
- Descoberta de serviços
- Roteamento HTTP
- Balanceamento de carga do lado do cliente

O framework tenta evitar as desvantagens de frameworks como Spring, Spring Boot e Grails:

- Tempo de inicialização rápido
- Redução do consumo de memória
- Uso mínimo de reflexão
- Uso mínimo de proxies
- Nenhuma geração de bytecode em tempo de execução
- Testes de unidade fáceis

Historicamente, frameworks como Spring e Grails não foram projetados para rodar em cenários com funções serverless, aplicativos Android ou microsserviços com baixo consumo de memória. Em contraste, o framework Micronaut foi projetado para ser adequado a todos esses cenários.

Esse objetivo é alcançado através do uso de processadores de anotação Java. Processadores de anotação pré-compilam os metadados necessários para executar DI, definir proxies AOP e configurar sua aplicação para ser executada em um ambiente com pouca memória.


# Referências

[Micronaut](https://docs.micronaut.io/4.8.11/guide/#whatsNew)
