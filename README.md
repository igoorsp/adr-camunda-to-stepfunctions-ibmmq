# Análise de Arquitetura Técnica
## Migração Camunda → AWS Step Functions — Integração com IBM MQ via Service Task

| Campo | Valor |
|---|---|
| **Documento** | Architecture Decision Record (ADR) |
| **Contexto** | Plataforma de Integração Empresarial |
| **Status** | Proposto — Pendente de Aprovação |
| **Autor** | Time de Arquitetura Técnica |
| **Data** | Março 2026 |

> ⚠️ **CONFIDENCIAL — USO INTERNO**

---

## Índice

1. [Contexto e Escopo](#1-contexto-e-escopo)
2. [Alternativas Avaliadas](#2-alternativas-avaliadas)
3. [Diagrama de Arquitetura — Visão de Componentes](#3-diagrama-de-arquitetura--visão-de-componentes)
4. [Diagrama de Sequência — Fluxo de Integração](#4-diagrama-de-sequência--fluxo-de-integração)
5. [Fundamentação da Decisão Técnica](#5-fundamentação-da-decisão-técnica)
6. [Comparativo Consolidado das Alternativas](#6-comparativo-consolidado-das-alternativas)
7. [Riscos e Mitigações](#7-riscos-e-mitigações)
8. [Referências Técnicas](#8-referências-técnicas)
9. [Conclusão e Próximos Passos](#9-conclusão-e-próximos-passos)

---

## 1. Contexto e Escopo

Este documento registra a decisão de arquitetura referente à estratégia de integração com IBM MQ durante a migração dos processos Camunda para AWS Step Functions. O cenário central envolve processos de negócio que, no modelo atual, utilizam **Service Tasks do Camunda** para enviar e receber mensagens em filas IBM MQ.

A decisão aqui tomada é de natureza **estrutural** e será replicada para múltiplos processos. Por isso, os critérios de escalabilidade, padronização operacional e custo total de propriedade (TCO) foram ponderados com o mesmo peso que os critérios técnicos imediatos.

### 1.1 Modelo Atual (AS-IS)

No cenário atual, o Camunda Engine orquestra os processos de negócio e, via Service Task com implementação JMS/IBM MQ Client, publica e consome mensagens em filas IBM MQ. O fluxo é síncrono ou assíncrono dependendo da configuração do processo, e o Camunda mantém o estado da instância enquanto aguarda respostas.

### 1.2 Restrições e Premissas

- O IBM MQ permanece na infraestrutura legada durante o período de transição
- O IBM MQ Client (biblioteca nativa) e o protocolo JMS são as únicas opções de conectividade
- A solução escolhida precisa ser **replicável para N processos** distintos com custo marginal baixo
- A empresa opera em escala enterprise: alta disponibilidade, observabilidade e governança são mandatórios
- Existe investimento estabelecido em EKS (Kubernetes) e em workloads Spring Boot

---

## 2. Alternativas Avaliadas

### 2.1 Alternativa 1 — Lambda Nativa com Quarkus + GraalVM/Mandrel

> **Resumo:** Uma AWS Lambda implementada em Java 17 com Quarkus compilado via Mandrel/GraalVM (native image) se conecta diretamente ao IBM MQ usando o IBM MQ Client nativo (`com.ibm.mq.allclient`).

**Racional técnico — JMS vs IBM MQ Client nativo:**
A biblioteca JMS é uma abstração (`javax.jms` / `jakarta.jms`) que, para o IBM MQ, requer de qualquer forma o driver IBM MQ como provider. O uso direto do IBM MQ Client elimina uma camada de indireção e oferece acesso a features IBM-específicas (MQMD headers, reply queues correlacionadas, PCF). Para esse cenário onde o destino é exclusivamente IBM MQ, JMS adiciona abstração sem benefício concreto.

**Racional técnico — GraalVM/Mandrel:**
O cold start de uma Lambda JVM tradicional para um workload IBM MQ Client pode atingir 8–15 segundos, incompatível com SLAs de processo. O Quarkus compilado via Mandrel gera um native executable com cold start abaixo de 500ms e footprint de memória 70–80% menor.

**❌ Limitações críticas:**

- **IBM MQ Client + GraalVM Native Image:** O `allclient` realiza reflection massiva internamente para inicialização do canal de conexão, JSSE e serialização de mensagens. A IBM **não fornece nem mantém** oficialmente configurações de `reflect-config.json` para compilação native image. Isso impõe trabalho artesanal de mapeamento de classes refletidas, frágil a cada upgrade de versão do client.
- **Gestão de conexões:** IBM MQ favorece conexões persistentes (connection pooling via MQCF). Lambda é stateless por natureza — cada invocação potencialmente abre e fecha uma conexão, gerando overhead no broker e risco de atingir limites de canal (`MAXCHL`).
- **Replicabilidade baixa:** Cada novo processo exige uma nova Lambda com o mesmo setup de native image. Pipelines de build GraalVM custam 10–20 minutos por módulo.
- **Conectividade de rede:** Lambda em VPC exige Security Groups e rotas específicas para alcançar o IBM MQ na porta 1414.

---

### 2.2 Alternativa 2 — Lambda Proxy → Microserviço no EKS (Spring Boot)

> **Resumo:** Uma Lambda thin atua como proxy entre a Step Function e um microserviço Spring Boot no EKS, traduzindo o evento em chamada HTTP (REST/gRPC) para o microserviço, que interage com o IBM MQ.

**Aspecto positivo:** O Spring Boot possui suporte maduro ao IBM MQ via `mq-jms-spring-boot-starter`. O connection pool é gerenciado pelo Spring (`CachingConnectionFactory`), permitindo reuso de conexões — crítico para IBM MQ.

**❌ Antipadrão identificado — camada extra sem valor agregado:**

A Lambda nesta alternativa **não executa nenhuma lógica de negócio**. Ela existe apenas para bridgear o protocolo de invocação da Step Function para HTTP. Isso introduz:

1. Latência adicional de rede desnecessária
2. Mais um componente para monitorar, deployar e manter
3. Custo de invocação da Lambda sem contrapartida funcional
4. **Dois pontos de falha distintos** (Lambda + EKS service) aumentam a surface de erro
5. Acoplamento temporal: se o EKS estiver com latência alta, a Lambda e a Step Function ficam bloqueadas

A Step Function já possui integração SDK nativa com SQS que resolve o mesmo problema de bridging de forma mais elegante, sem Lambda.

---

### 2.3 Alternativa 3 — Step Function → SQS → EKS → IBM MQ ✅ RECOMENDADA

> **Resumo:** A Step Function publica em SQS via integração SDK nativa com `.waitForTaskToken`. Um microserviço Spring Boot no EKS consome o SQS e interage com o IBM MQ via IBM MQ Spring Boot Starter (JMS). O callback é feito via `SendTaskSuccess` / `SendTaskFailure`.

---

## 3. Diagrama de Arquitetura — Visão de Componentes

O diagrama abaixo apresenta a visão completa de componentes da solução recomendada (Alternativa 3). As zonas coloridas delimitam os domínios de responsabilidade: orquestração AWS, camada de integração no EKS (Anti-Corruption Layer) e infraestrutura legada on-premise.

![Diagrama de Arquitetura — Alternativa 3](/images/arch_diagram.png)

*Figura 1 — Diagrama de Arquitetura: Step Functions → SQS → Integration Adapter EKS (Spring Boot) → IBM MQ*

### 3.1 Descrição dos Componentes

| Componente | Tecnologia | Responsabilidade |
|---|---|---|
| **AWS Step Functions** | State Machine | Orquestrador do processo de negócio. Publica no SQS e aguarda callback via `TaskToken`. |
| **Amazon SQS** | Queue + DLQ | Buffer desacoplador entre orquestrador e executor. DLQ para mensagens com falha repetida. |
| **Integration Adapter (EKS)** | Spring Boot 3.x | Anti-Corruption Layer. Consome SQS, interage com IBM MQ via JMS, envia callback à Step Function. |
| **KEDA** | K8s Autoscaler | Escala os pods do Integration Adapter proporcionalmente ao número de mensagens no SQS. |
| **IBM MQ Broker** | IBM MQ 9.x | Broker legado. Mantém as filas REQUEST e REPLY consumidas/publicadas pelos sistemas existentes. |
| **SSM / AppConfig** | AWS Config | Mapeamento externalizado SQS Queue → IBM MQ Queue. Permite onboarding de novos processos sem redeploy. |
| **CloudWatch + X-Ray** | Observabilidade | Métricas, alarmes e rastreamento distribuído end-to-end. |

---

## 4. Diagrama de Sequência — Fluxo de Integração

O diagrama de sequência detalha o fluxo de interação entre os participantes. O padrão `.waitForTaskToken` é central: a Step Function publica no SQS com o TaskToken como atributo da mensagem, pausa a execução e retoma somente após receber o callback do Integration Adapter — replicando exatamente o comportamento síncrono do Service Task do Camunda.

![Diagrama de Sequência — Fluxo completo](/images/seq_diagram.png)

*Figura 2 — Diagrama de Sequência: Step Functions → SQS → Integration Adapter (EKS) → IBM MQ → Callback*

### 4.1 Detalhamento dos Passos

| Passo | Descrição Técnica |
|---|---|
| **[1] SF → SQS** | Step Function invoca o estado de integração. Via `Resource: arn:aws:states:::sqs:sendMessage`, publica no SQS com payload + `TaskToken` como `MessageAttribute`. Sem Lambda intermediária. |
| **[2] SF pausa** | Step Function entra em modo `waitForTaskToken`. Execução pausada. Nenhum recurso computacional consumido durante a espera — billing baseado em transições de estado. |
| **[3] EKS → SQS** | Integration Adapter faz polling via `@SqsListener` / `DefaultMessageListenerContainer`. Consome a mensagem, extrai payload e TaskToken. Visibility timeout ativado. |
| **[4] EKS → IBM MQ** | Via IBM MQ Spring Boot Starter (JMS), o adapter executa `MQPUT` na fila REQUEST. A `CachingConnectionFactory` garante reuso de conexão — sem abrir nova conexão por mensagem. |
| **[5] Legado → REPLY** | Sistema legado processa a mensagem na fila REQUEST e publica a resposta na fila REPLY correlacionada por `CorrelationId` / `ReplyToQueue` (MQMD header). |
| **[6] EKS ← IBM MQ** | Integration Adapter executa `MQGET` na fila REPLY e obtém a resposta. |
| **[7] EKS → SF callback** | Adapter chama `stepFunctions.sendTaskSuccess(taskToken, output)` via AWS SDK. Em erro: `sendTaskFailure(taskToken, errorCode, cause)`. |
| **[8] DeleteMessage** | Adapter confirma o processamento com `DeleteMessage` no SQS. Em falha, a mensagem retorna após o visibility timeout e pode ser reprocessada até `maxReceiveCount`, depois segue para DLQ. |

> ⚠️ **Caminho de erro:** `SendTaskFailure` → Step Function trata via `Catch` / `Retry` configurados na State Machine. Mensagens que excedem `maxReceiveCount` são redirecionadas automaticamente para a Dead Letter Queue (DLQ).

---

## 5. Fundamentação da Decisão Técnica

### 5.1 Desacoplamento via Mensageria (Princípio de Design Fundamental)

O SQS atua como buffer desacoplador entre o orquestrador (Step Function) e o executor (Integration Adapter EKS). Esse padrão segue o princípio de **Loose Coupling**, referenciado em Richardson (*Microservices Patterns*, 2018) e Newman (*Building Microservices*, 2021), e é o alicerce do Event-Driven Architecture (EDA).

> 💡 **Princípio:** A Step Function não precisa saber como o IBM MQ funciona. O Integration Adapter não precisa saber como a Step Function funciona. O SQS é o **contrato entre eles** — qualquer lado pode evoluir independentemente.

Na Alternativa 2, a Lambda proxy cria um **acoplamento temporal** — se o microserviço EKS estiver com latência alta, a Lambda e a Step Function ficam bloqueadas esperando a resposta HTTP. Com SQS, a Step Function apenas publica e continua.

### 5.2 Integração Nativa Step Function → SQS

AWS Step Functions possui integração nativa com SQS como **Optimized Integration (SDK Integration)**. A Step Function pode publicar em SQS sem uma Lambda intermediária:

```json
"PublishToMQ": {
  "Type": "Task",
  "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
  "Parameters": {
    "QueueUrl": "https://sqs.us-east-1.amazonaws.com/account/process-queue",
    "MessageBody.$": "$.payload",
    "MessageAttributes": {
      "TaskToken": {
        "DataType": "String",
        "StringValue.$": "$$.Task.Token"
      }
    }
  },
  "HeartbeatSeconds": 300,
  "Retry": [{
    "ErrorEquals": ["States.TaskFailed"],
    "IntervalSeconds": 5,
    "MaxAttempts": 3,
    "BackoffRate": 2.0
  }],
  "Catch": [{
    "ErrorEquals": ["States.ALL"],
    "Next": "HandleIntegrationError"
  }]
}
```

### 5.3 Conectividade IBM MQ — IBM MQ Spring Boot Starter (JMS)

**Decisão:** usar o `com.ibm.mq:mq-jms-spring-boot-starter`, mantido **oficialmente pela IBM** no repositório [`ibm-messaging/mq-jms-spring`](https://github.com/ibm-messaging/mq-jms-spring).

Ele utiliza JMS como API com IBM MQ Client como provider, entregando:

| Aspecto | IBM MQ Client Direto | IBM MQ Spring Starter (JMS) ✅ |
|---|---|---|
| Integração Spring | Manual (bean MQCF) | Auto-configurada via `application.properties` |
| Connection Pooling | Implementação manual | `CachingConnectionFactory` nativa |
| Listener Container | Não disponível OOB | `DefaultMessageListenerContainer` gerenciado |
| Portabilidade futura | Acoplada IBM MQ API | API JMS padrão (Jakarta EE) |
| Observabilidade | Manual | Micrometer + Spring Actuator nativos |
| Suporte IBM | Sim | **Sim — mantido oficialmente** |

**Configuração base (`application.properties`):**

```properties
ibm.mq.queue-manager=QM1
ibm.mq.channel=DEV.APP.SVRCONN
ibm.mq.conn-name=mq-host(1414)
ibm.mq.user=app
ibm.mq.password=${MQ_PASSWORD}
ibm.mq.ssl-cipher-suite=TLS_RSA_WITH_AES_256_CBC_SHA256
```

### 5.4 Replicabilidade para N Processos (Critério Estratégico)

> 🎯 Este é o critério mais relevante dado que a solução será replicada para múltiplos processos.

O Integration Adapter é projetado como um **Anti-Corruption Layer** (Evans, *Domain-Driven Design*, 2003) com configuração externalizada. Novos processos são onboardados via:

1. **Criação de nova fila SQS** — IaC via Terraform/CDK, operação trivial e rastreável
2. **Adição de rota no SSM/AppConfig** — mapeamento `SQS Queue → IBM MQ Queue` sem redeploy
3. **Definição do novo estado na Step Function** — integração SQS com `.waitForTaskToken`

Nas Alternativas 1 e 2, cada novo processo tende a exigir rebuild de Lambda (com peso de compilação GraalVM na Alt. 1) ou nova rota + deploy explícito (Alt. 2). Na Alternativa 3, o custo marginal de onboarding de novos processos é mínimo.

### 5.5 Resiliência e Escalabilidade

- **Backpressure natural:** Se o EKS estiver sobrecarregado, mensagens permanecem no SQS (retenção configurável até 14 dias) sem perda. Nas Alternativas 1 e 2, sobrecarga gera falhas síncronas diretas.
- **Dead Letter Queue (DLQ):** SQS suporta DLQ nativo. Mensagens que falham repetidamente são redirecionadas automaticamente — mecanismo inexistente em chamadas síncronas HTTP.
- **Auto-scaling com KEDA:** Os pods do Integration Adapter escalam diretamente com base no `ApproximateNumberOfMessages` do SQS, sem necessidade de configuração adicional de métricas.
- **Retry com backoff:** O SQS visibility timeout + Spring JMS retry policies absorvem falhas transitórias no IBM MQ (rede, canal indisponível) sem impactar o orquestrador.

### 5.6 Observabilidade End-to-End

- **AWS X-Ray:** Step Functions integra nativamente. O Integration Adapter propaga o trace context via Micrometer Tracing / Spring Cloud Sleuth.
- **CloudWatch Metrics:** SQS expõe métricas nativas (`NumberOfMessagesSent`, `ApproximateAgeOfOldestMessage`, `NumberOfMessagesNotVisible`) para alarmes sem instrumentação adicional.
- **Auditoria:** SQS + S3 + Kinesis Data Firehose permite auditoria completa de mensagens em trânsito — fundamental para compliance enterprise.

---

## 6. Comparativo Consolidado das Alternativas

Pontuação relativa por critério (1 = baixo/ruim → 5 = alto/bom):

| Critério de Avaliação | Alt. 1 — Lambda GraalVM | Alt. 2 — Lambda Proxy + EKS | Alt. 3 — SQS + EKS ✅ |
|---|:---:|:---:|:---:|
| Complexidade de implementação inicial | ⚠️ Baixa (1) | ✅ Média (3) | ✅ Baixa (4) |
| Compatibilidade IBM MQ Client + runtime | ❌ Baixa (1) | ✅ Alta (5) | ✅ Alta (5) |
| Resiliência / Tolerância a falhas | ❌ Baixa (2) | ⚠️ Média (3) | ✅ Alta (5) |
| Escalabilidade | ⚠️ Média (3) | ⚠️ Média (3) | ✅ Alta (5) |
| Replicabilidade para N processos | ❌ Baixa (1) | ⚠️ Média (3) | ✅ Alta (5) |
| Desacoplamento orquestrador/integração | ❌ Baixo (1) | ⚠️ Médio (3) | ✅ Alto (5) |
| Observabilidade nativa | ⚠️ Média (3) | ⚠️ Média (3) | ✅ Alta (5) |
| Manutenibilidade / Operação | ❌ Baixa (1) | ⚠️ Média (3) | ✅ Alta (5) |
| Custo TCO estimado (relativo) | ⚠️ Médio (3) | ❌ Médio-alto (2) | ✅ Médio (4) |
| Aderência aos padrões AWS enterprise | ⚠️ Parcial (3) | ⚠️ Parcial (3) | ✅ Total (5) |
| **PONTUAÇÃO TOTAL** | **19 / 50** | **31 / 50** | **48 / 50** |

---

## 7. Riscos e Mitigações

| Risco | Severidade | Probabilidade | Mitigação |
|---|:---:|:---:|---|
| Latência adicionada pelo SQS para processos com SLA < 100ms | Alta | Baixa | Para casos excepcionais, avaliar Lambda com IBM MQ em JVM + SnapStart (Java 17). |
| IBM MQ Channel limit (`MAXCHL`) atingido com múltiplos pods EKS | Média | Média | `CachingConnectionFactory` com pool configurado. HPA com limite de replicas alinhado ao `MAXCHL` do broker. |
| Mensagem perdida entre SQS e microserviço | Alta | Muito Baixa | SQS com DLQ. Visibility timeout configurado acima do SLA máximo de processamento. |
| `TaskToken` expirado antes do processamento | Média | Baixa | Heartbeat periódico (`SendTaskHeartbeat`) enviado pelo adapter durante processamentos longos. |
| Incompatibilidade de versão IBM MQ Client ao atualizar | Média | Média | Fixar versão do `mq-jms-spring-boot-starter` no BOM. Testes de regressão no pipeline CI/CD. |

---

## 8. Referências Técnicas

- **AWS Documentation** — Step Functions SQS SDK Integration (`.waitForTaskToken`)
  `https://docs.aws.amazon.com/step-functions/latest/dg/connect-sqs.html`

- **IBM Documentation** — IBM MQ Spring Boot Starter (oficial)
  `https://www.ibm.com/docs/en/ibm-mq/9.3?topic=spring-using-mq-jms-starter`
  `https://github.com/ibm-messaging/mq-jms-spring`

- **GraalVM Documentation** — Reflection in Native Image (limitações)
  `https://www.graalvm.org/latest/reference-manual/native-image/dynamic-features/Reflection/`

- **KEDA** — SQS Scaler para Kubernetes
  `https://keda.sh/docs/scalers/aws-sqs-queue/`

- **Richardson, C.** (2018). *Microservices Patterns*. Manning Publications.
  Capítulo 3: Interprocess communication — Messaging patterns

- **Newman, S.** (2021). *Building Microservices*, 2ª Ed. O'Reilly Media.
  Capítulo 4: Communication styles — Asynchronous Nonblocking Communication

- **Evans, E.** (2003). *Domain-Driven Design*. Addison-Wesley.
  Anti-Corruption Layer pattern — Part IV: Strategic Design

- **AWS Blog** — Best practices for AWS Step Functions
  `https://docs.aws.amazon.com/step-functions/latest/dg/bp-express-workflows.html`

---

## 9. Conclusão e Próximos Passos

> ✅ **DECISÃO FINAL:** A **Alternativa 3** (Step Function → SQS → Integration Adapter EKS Spring Boot → IBM MQ) é a arquitetura recomendada. O **IBM MQ Spring Boot Starter (JMS)** é a forma de conectividade recomendada dentro do microserviço. O padrão `.waitForTaskToken` garante a semântica síncrona do processo de negócio sem acoplamento temporal.

### Próximos Passos

1. **Processo piloto** — Seleção de um processo Camunda para PoC da Alternativa 3
2. **Módulo base** — Criação do Integration Adapter Spring Boot com configuração externalizada via SSM/AppConfig
3. **IaC** — Pipeline Terraform/CDK para provisionamento de SQS + DLQ por processo
4. **Observabilidade** — Definição da estratégia de trace end-to-end (X-Ray + Micrometer Tracing + Grafana)
5. **Capacidade MQ** — Alinhamento com time de infra legada para dimensionamento do `MAXCHL` do IBM MQ
6. **ARB** — Apresentação desta ADR ao Architecture Review Board para aprovação formal

---

*Este documento é um Architecture Decision Record (ADR) e deve ser versionado junto ao repositório de arquitetura da empresa.*

*Time de Arquitetura Técnica — Março 2026*