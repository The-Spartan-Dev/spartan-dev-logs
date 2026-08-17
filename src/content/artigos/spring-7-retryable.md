---
title: "Spring Framework 7: entendendo a nova anotação @Retryable na prática"
description: "O mecanismo de retry agora é nativo no Spring. Como usar @Retryable com exponential backoff para construir aplicações resilientes sem poluir o código."
pubDate: 2026-08-16
pilar: engenharia
tags: ["java", "spring"]
---

A resiliência é um aspecto central em sistemas distribuídos. Falhas transitórias — como instabilidades de rede, timeouts ou indisponibilidades momentâneas — são esperadas e comuns.

A partir da nova versão do Spring Framework, o mecanismo de _retry_ passa a ser nativo, eliminando a necessidade de dependências externas como o `spring-retry` e aprimorando a experiência de desenvolvimento de aplicações resilientes dentro do ecossistema Spring.

## Habilitando métodos resilientes

Antes de utilizar as anotações de resiliência, é necessário habilitar o suporte aos métodos resilientes por meio da anotação `@EnableResilientMethods`, que deve ser aplicada a uma classe de configuração da aplicação.

```java
package com.example.retryabledemo;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.resilience.annotation.EnableResilientMethods;

import com.example.retryabledemo.services.PagamentoService;

@SpringBootApplication
@EnableResilientMethods // Anotação que habilita os métodos resilientes
public class RetryableDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(RetryableDemoApplication.class, args);
    }

    @Bean
    CommandLineRunner run(PagamentoService pagamentoService) {
        return args -> {
            pagamentoService.cobrar();
        };
    }
}
```

## Usando a anotação @Retryable

A anotação `@Retryable` é aplicada diretamente ao método, podendo ter seus parâmetros configurados ou não (conforme será detalhado adiante). No exemplo apresentado, o serviço de pagamento delega o processamento a um serviço externo, cuja execução está sujeita a falhas e pode lançar uma exceção customizada de indisponibilidade.

```java
package com.example.retryabledemo.services;

import org.springframework.resilience.annotation.Retryable;
import org.springframework.stereotype.Service;

import com.example.retryabledemo.exception.IndisponibilidadeException;

@Service
public class PagamentoService {

    private final GatewayPagamentoService gatewayPagamento;

    public PagamentoService(GatewayPagamentoService gatewayPagamento) {
        this.gatewayPagamento = gatewayPagamento;
    }

    // Configuração declarativa da política de retry
    @Retryable(
        includes = IndisponibilidadeException.class,
        maxRetries = 5,
        delay = 2000,
        multiplier = 2.0
    )
    public void cobrar() {
        gatewayPagamento.processar();
    }
}
```

```java
package com.example.retryabledemo.services;

import java.time.LocalTime;
import java.util.concurrent.ThreadLocalRandom;

import org.springframework.stereotype.Service;

import com.example.retryabledemo.exception.IndisponibilidadeException;

@Service
public class GatewayPagamentoService {

    public void processar() {
        System.out.println(LocalTime.now() + " - Chamando API do Gateway de Pagamento...");

        // Simula 80% de chance de falha transitória
        if (ThreadLocalRandom.current().nextDouble() < 0.8) {
            throw new IndisponibilidadeException("Gateway instável. Tente novamente.");
        }

        System.out.println("Pagamento processado com sucesso!");
    }
}
```

## Parâmetros da anotação @Retryable

**`includes`** — define quais exceções disparam o retry. Neste caso, apenas a exceção `IndisponibilidadeException`. Por padrão, qualquer exceção é considerada. Também é possível definir mais de uma exceção utilizando chaves (_curly braces_) e separando-as por vírgula, por exemplo: `{ IndisponibilidadeException.class, TimeoutException.class }`.

**`excludes`** — define quais exceções **não** disparam o retry. No exemplo, o parâmetro foi omitido. Deste modo, seu valor permanece vazio.

**`maxRetries`** — indica o número máximo de tentativas adicionais após a primeira execução. Neste caso, o método pode ser executado até 6 vezes no total (1 tentativa inicial + 5 tentativas adicionais). Por padrão, o valor desse parâmetro é 3. Nesse caso, poderiam ocorrer, no máximo, quatro tentativas (1 tentativa inicial + 3 tentativas adicionais).

**`delay`** — define o intervalo inicial (em milissegundos) entre as tentativas. Neste caso, a primeira tentativa de retry ocorrerá após 2 segundos.

**`multiplier`** — define uma estratégia de _exponential backoff_, na qual o intervalo de espera é progressivamente aumentado a cada nova tentativa. Assim, o tempo de atraso é multiplicado a cada retry, reduzindo a frequência das requisições ao longo do tempo. O intervalo antes da tentativa $n$ segue a fórmula:

$$
\text{delay}_n = \text{delay} \times \text{multiplier}^{\,n-1}
$$

Essa abordagem mitiga o anti-padrão conhecido como _retry storm_, que ocorre quando múltiplas tentativas são disparadas contra um serviço indisponível ou sobrecarregado, agravando ainda mais a condição de falha em vez de permitir sua recuperação. Com `delay = 2000` e `multiplier = 2.0`, o comportamento será:

| Tentativa | Intervalo de espera |
| --------- | ------------------- |
| 1ª        | 2s                  |
| 2ª        | 4s                  |
| 3ª        | 8s                  |
| 4ª        | 16s                 |
| 5ª        | 32s                 |

Caso todas as tentativas falhem, a exceção é propagada para o método chamador. A abordagem declarativa permite definir políticas de retry que não poluem o código com estruturas imperativas como `try/catch`, `while` ou `Thread.sleep`, amplamente utilizadas em sistemas legados.

## Referências

- [Spring Docs — Resilience Features](https://docs.spring.io/spring-framework/reference/integration/resilience.html)
