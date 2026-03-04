# Análise de Arquitetura Técnica
## Migração Camunda → AWS Step Functions — Integração com IBM MQ via Service Task

| Campo | Valor |
|---|---|
| **Documento** | Architecture Decision Record (ADR) |
| **Contexto** | Plataforma de Integração Empresarial |
| **Status** | Proposto — Pendente de Aprovação |
| **Autor** | Time de Arquitetura Técnica |
| **Data** | Março 2026 |
| **Versão** | 2.0 — Análise da Alternativa 1 revisada com base em documentação oficial IBM |

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

No cenário atual, o Camunda Engine orquestra os processos de negócio e, via Service Task com implementação JMS/IBM MQ Client (`com.ibm.mq.allclient`), publica e consome mensagens em filas IBM MQ. O padrão predominante é **request/reply correlacionado**: o processo publica em uma fila REQUEST, aguarda resposta em uma fila REPLY usando `JMSCorrelationID` ou `ReplyToQueue` como mecanismo de correlação, e o Camunda mantém o estado da instância durante a espera.

### 1.2 Restrições e Premissas

- O IBM MQ permanece na infraestrutura legada durante o período de transição
- O IBM MQ Client (`allclient`) é a biblioteca em uso hoje nos processos Camunda
- Os processos usam padrão request/reply com correlação de mensagens — padrão JMS com seletores
- A solução escolhida precisa ser **replicável para N processos** distintos com custo marginal baixo
- A empresa opera em escala enterprise: alta disponibilidade, observabilidade e governança são mandatórios
- Existe investimento estabelecido em EKS (Kubernetes) e em workloads Spring Boot

---

## 2. Alternativas Avaliadas

### 2.1 Alternativa 1 — Lambda Nativa com Quarkus + GraalVM/Mandrel

> **Resumo:** Uma AWS Lambda implementada em Java 17 com Quarkus compilado via Mandrel/GraalVM (native image) se conecta diretamente ao IBM MQ.

#### 2.1.1 Análise inicial — IBM MQ allclient + GraalVM

A primeira abordagem considerada foi usar o IBM MQ Client nativo (`com.ibm.mq.allclient`) compilado com GraalVM Native Image. Esse caminho foi descartado por um bloqueador técnico fundamental: o `allclient` realiza **reflection dinâmica massiva** internamente para inicialização de canais, JSSE e serialização de mensagens — incompatível com a compilação ahead-of-time do GraalVM, que exige que todos os acessos por reflection sejam declarados estaticamente em `reflect-config.json`. A IBM **não fornece nem mantém** essas configurações oficialmente.

#### 2.1.2 Caminho alternativo investigado — Quarkus + GraalVM + Apache Qpid AMQP

Após identificar o bloqueador acima, foram consultados dois tutoriais oficiais IBM Developer que descrevem o caminho endossado pela IBM para rodar aplicações IBM MQ com Quarkus e GraalVM:

- **Tutorial 1:** *"Running your IBM MQ apps on Quarkus and GraalVM using QPid AMQP JMS classes"* — IBM Developer (Jun/2025)
- **Tutorial 2:** *"Building cloud-native reactive Java messaging applications using Quarkus, Smallrye, and IBM MQ"* — IBM Developer (Fev/2025)

A conclusão dos dois documentos é unânime: **o único caminho IBM-endossado para Quarkus + GraalVM + IBM MQ é via protocolo AMQP 1.0**, usando o **Apache Qpid JMS Client** como biblioteca — não o `allclient` proprietário.

O Tutorial 1 é explícito:

> *"Let's be clear here. These are Qpid AMQP client apps that use the Qpid JMS API stack, and nothing is IBM proprietary. The configuration jndi.properties file is the only link to IBM MQ."*

O Tutorial 2 descreve a stack resultante:

> *"The Quarkus application relies on two key open-source technologies — the Smallrye reactive messaging framework and the vert.x AMQP client."*

A IBM contornou a incompatibilidade com GraalVM **trocando de protocolo** (MQ nativo → AMQP 1.0), não resolvendo a limitação de reflection do `allclient`.

#### 2.1.3 Por que o caminho Qpid/AMQP não resolve o cenário de migração Camunda

O caminho Qpid/AMQP apresenta **três bloqueadores independentes e cumulativos** quando aplicado ao contexto de migração de processos Camunda:

**Bloqueador 1 — Dependência operacional: AMQP Channel no broker legado**

O Tutorial 1 exige explicitamente:

> *"An AMQP channel enabled instance of IBM MQ."*

Para habilitar AMQP, é necessário configurar um AMQP Channel e ativar o listener na porta 5672 no Queue Manager. Em ambientes enterprise com IBM MQ gerenciado por um time de infraestrutura separado, isso é uma **dependência externa com aprovação e prazo imprevisíveis** — um bloqueador operacional antes mesmo de escrever código.

**Bloqueador 2 — Features JMS essenciais não suportadas via AMQP**

O Tutorial 1 lista features que **podem não funcionar** via AMQP no IBM MQ:

> *"As well as features that may not be supported such as: use of a selector, transactions, message expiry, and message delay."*

Isso é diretamente incompatível com o padrão dos Service Tasks Camunda, que usa:
- **Seletores JMS** (`JMSCorrelationID = '...'`) para consumir respostas correlacionadas na fila REPLY
- **Transações JMS** para garantir consistência no par publicar/consumir
- **Message expiry** para controle de TTL das mensagens de processo

Sem suporte a seletores, o padrão request/reply correlacionado — o coração da integração Camunda-IBM MQ — **não pode ser replicado via AMQP**.

**Bloqueador 3 — Modelo reativo incompatível com semântica síncrona dos Service Tasks**

O Tutorial 2 demonstra arquitetura pub/sub reativa com Smallrye:

> *"By default, IBM MQ assumes the address is an AMQP topic."*

O modelo reativo (fire-and-forget, pub/sub) é filosoficamente oposto ao comportamento de um Service Task Camunda, que bloqueia a instância do processo até receber uma resposta correlacionada. Adaptar o modelo Smallrye para replicar a semântica request/reply síncrona exigiria engenharia adicional significativa, eliminando as vantagens do framework reativo.

**❌ Conclusão sobre a Alternativa 1:**

A Alternativa 1 **não é tecnicamente impossível** — existe um caminho IBM-documentado com Quarkus + GraalVM + Qpid AMQP. Ela é **inadequada para este cenário específico** por razões concretas e verificáveis na documentação oficial IBM: troca de protocolo incompatível com o legado, ausência de suporte a features JMS de correlação, e modelo reativo incompatível com a semântica síncrona dos Service Tasks. Os problemas de gestão de conexões stateless e custo de replicabilidade para N processos permanecem independentemente do protocolo.

---

### 2.2 Alternativa 2 — Lambda Proxy → Microserviço no EKS (Spring Boot)

> **Resumo:** Uma Lambda thin atua como proxy entre a Step Function e um microserviço Spring Boot no EKS, traduzindo o evento em chamada HTTP, que interage com o IBM MQ.

**Aspecto positivo:** O Spring Boot com `mq-jms-spring-boot-starter` usa o `allclient` como provider JMS — **protocolo nativo IBM MQ**, sem necessidade de AMQP channel. Seletores JMS, transações e correlação funcionam plenamente.

**❌ Antipadrão identificado — camada extra sem valor agregado:**

A Lambda não executa nenhuma lógica de negócio. Ela existe apenas para bridgear o evento da Step Function em chamada HTTP. Isso introduz latência adicional, segundo ponto de falha, custo de invocação sem contrapartida, acoplamento temporal e um componente extra para operar. A Step Function possui integração SDK nativa com SQS que resolve o mesmo bridging sem Lambda.

---

### 2.3 Alternativa 3 — Step Function → SQS → EKS → IBM MQ ✅ RECOMENDADA

> **Resumo:** A Step Function publica em SQS via integração SDK nativa com `.waitForTaskToken`. Um microserviço Spring Boot no EKS consome o SQS e interage com o IBM MQ via IBM MQ Spring Boot Starter (JMS/`allclient`). O callback é feito via `SendTaskSuccess` / `SendTaskFailure`.

Esta alternativa mantém o **protocolo nativo IBM MQ** (`allclient` via JMS Spring Starter), preserva todas as features de correlação, seletores e transações, e **não requer nenhuma alteração no broker legado**.

---

## 3. Diagrama de Arquitetura — Visão de Componentes

![Diagrama de Arquitetura — Alternativa 3](arch_diagram.png)

*Figura 1 — Diagrama de Arquitetura: Step Functions → SQS → Integration Adapter EKS (Spring Boot) → IBM MQ*

### 3.1 Descrição dos Componentes

| Componente | Tecnologia | Responsabilidade |
|---|---|---|
| **AWS Step Functions** | State Machine | Orquestrador do processo de negócio. Publica no SQS e aguarda callback via `TaskToken`. |
| **Amazon SQS** | Queue + DLQ | Buffer desacoplador entre orquestrador e executor. DLQ para mensagens com falha repetida. |
| **Integration Adapter (EKS)** | Spring Boot 3.x | Anti-Corruption Layer. Consome SQS, interage com IBM MQ via JMS (`allclient`), envia callback à Step Function. |
| **KEDA** | K8s Autoscaler | Escala os pods do Integration Adapter proporcionalmente ao número de mensagens no SQS. |
| **IBM MQ Broker** | IBM MQ 9.x | Broker legado. **Sem alterações necessárias** — protocolo nativo MQ, sem AMQP channel. |
| **SSM / AppConfig** | AWS Config | Mapeamento externalizado SQS Queue → IBM MQ Queue. Onboarding de novos processos sem redeploy. |
| **CloudWatch + X-Ray** | Observabilidade | Métricas, alarmes e rastreamento distribuído end-to-end. |

---

## 4. Diagrama de Sequência — Fluxo de Integração

![Diagrama de Sequência — Fluxo completo](seq_diagram.png)

*Figura 2 — Diagrama de Sequência: Step Functions → SQS → Integration Adapter (EKS) → IBM MQ → Callback*

### 4.1 Detalhamento dos Passos

| Passo | Descrição Técnica |
|---|---|
| **[1] SF → SQS** | Step Function publica no SQS com payload + `TaskToken` como `MessageAttribute`. Integração SDK nativa — sem Lambda intermediária. |
| **[2] SF pausa** | Execução pausada via `waitForTaskToken`. Nenhum recurso computacional consumido durante a espera. |
| **[3] EKS → SQS** | Integration Adapter faz polling via `@SqsListener`. Consome a mensagem, extrai payload e TaskToken. Visibility timeout ativado. |
| **[4] EKS → IBM MQ** | Via IBM MQ Spring Boot Starter (JMS / `allclient`), executa `MQPUT` na fila REQUEST com `ReplyToQueue` configurado. `CachingConnectionFactory` garante reuso de conexão. |
| **[5] Legado → REPLY** | Sistema legado processa e publica resposta na fila REPLY. Correlação via `CorrelationId` (MQMD header nativo — indisponível via AMQP). |
| **[6] EKS ← IBM MQ** | Integration Adapter executa `MQGET` com seletor `JMSCorrelationID` na fila REPLY. |
| **[7] EKS → SF callback** | `stepFunctions.sendTaskSuccess(taskToken, output)`. Em erro: `sendTaskFailure(taskToken, errorCode, cause)`. |
| **[8] DeleteMessage** | Confirmação do processamento com `DeleteMessage` no SQS. Mensagem vai para DLQ após `maxReceiveCount`. |

> ⚠️ **Caminho de erro:** `SendTaskFailure` → Step Function trata via `Catch` / `Retry`. DLQ ativada após `maxReceiveCount` excedido.

---

## 5. Fundamentação da Decisão Técnica

### 5.1 Desacoplamento via Mensageria

> 💡 **Princípio:** A Step Function não precisa saber como o IBM MQ funciona. O Integration Adapter não precisa saber como a Step Function funciona. O SQS é o **contrato entre eles** — qualquer lado pode evoluir independentemente.

### 5.2 Integração Nativa Step Function → SQS com `.waitForTaskToken`

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
  "Retry": [{ "ErrorEquals": ["States.TaskFailed"], "IntervalSeconds": 5, "MaxAttempts": 3, "BackoffRate": 2.0 }],
  "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "HandleIntegrationError" }]
}
```

### 5.3 Conectividade IBM MQ — IBM MQ Spring Boot Starter

`com.ibm.mq:mq-jms-spring-boot-starter` — mantido oficialmente pela IBM em [`ibm-messaging/mq-jms-spring`](https://github.com/ibm-messaging/mq-jms-spring). Usa `allclient` como provider: **protocolo nativo IBM MQ, sem AMQP, sem alterações no broker**.

```properties
ibm.mq.queue-manager=QM1
ibm.mq.channel=DEV.APP.SVRCONN
ibm.mq.conn-name=mq-host(1414)
ibm.mq.user=app
ibm.mq.password=${MQ_PASSWORD}
ibm.mq.ssl-cipher-suite=TLS_RSA_WITH_AES_256_CBC_SHA256
```

### 5.4 Por que não AMQP — Decisão Explicitamente Registrada

Durante a análise da Alternativa 1, dois tutoriais oficiais IBM Developer foram lidos integralmente (listados na Seção 8). Ambos confirmam que o caminho GraalVM + IBM MQ requer **protocolo AMQP 1.0** via Qpid — não o `allclient`. Esse caminho foi descartado por três razões verificáveis diretamente na documentação IBM:

| Razão | Evidência na documentação IBM |
|---|---|
| AMQP channel obrigatório no broker legado | Tutorial 1: *"An AMQP channel enabled instance of IBM MQ"* |
| Seletores JMS e transações possivelmente não suportados | Tutorial 1: *"features that may not be supported: selector, transactions"* |
| Modelo reativo incompatível com request/reply síncrono | Tutorial 2: *"By default, IBM MQ assumes the address is an AMQP topic"* |

### 5.5 Replicabilidade para N Processos

Onboarding de novos processos na Alternativa 3 via:

1. **Nova fila SQS** — IaC Terraform/CDK, rastreável e sem fricção
2. **Rota no SSM/AppConfig** — mapeamento `SQS Queue → IBM MQ Queue`, sem redeploy
3. **Novo estado na Step Function** — integração SQS com `.waitForTaskToken`

---

## 6. Comparativo Consolidado das Alternativas

| Critério | Alt. 1 — Lambda GraalVM | Alt. 2 — Lambda Proxy + EKS | Alt. 3 — SQS + EKS ✅ |
|---|:---:|:---:|:---:|
| Compatibilidade `allclient` + GraalVM | ❌ Inviável; Qpid/AMQP possível mas com restrições | ✅ Via Spring Starter | ✅ Via Spring Starter |
| Seletores JMS e transações | ❌ Não suportado via AMQP (doc. IBM) | ✅ Nativo | ✅ Nativo |
| Alteração necessária no broker IBM MQ | ❌ AMQP channel obrigatório (porta 5672) | ✅ Nenhuma | ✅ Nenhuma |
| Protocolo nativo IBM MQ | ❌ AMQP ≠ protocolo nativo | ✅ Protocolo nativo | ✅ Protocolo nativo |
| Resiliência / DLQ | ❌ Sem DLQ nativo | ⚠️ Média | ✅ DLQ nativo SQS |
| Escalabilidade | ⚠️ Stateless + conn limit | ⚠️ Média | ✅ KEDA + HPA |
| Replicabilidade N processos | ❌ Rebuild GraalVM por processo | ⚠️ Nova rota + Lambda | ✅ Config-driven (SSM) |
| Desacoplamento | ❌ Baixo | ⚠️ HTTP síncrono | ✅ SQS como contrato |
| Manutenibilidade | ❌ Builds GraalVM frágeis | ⚠️ 2 componentes | ✅ Spring Boot padrão |
| Custo TCO | ⚠️ CI custoso | ❌ Lambda + EKS | ✅ Apenas EKS + SQS |
| **PONTUAÇÃO TOTAL** | **9 / 50** | **31 / 50** | **48 / 50** |

> **Nota:** A revisão baseada nos tutoriais IBM reduziu a pontuação da Alt. 1 em relação à análise inicial. O caminho Qpid/AMQP introduz incompatibilidades diretas com o cenário de migração Camunda, tornando-a ainda menos adequada do que a avaliação preliminar indicava.

---

## 7. Riscos e Mitigações

| Risco | Severidade | Probabilidade | Mitigação |
|---|:---:|:---:|---|
| Latência SQS para SLA < 100ms | Alta | Baixa | Lambda JVM + SnapStart (Java 17) com `allclient` — sem GraalVM. |
| `MAXCHL` atingido com múltiplos pods EKS | Média | Média | `CachingConnectionFactory` com pool. HPA com réplicas alinhadas ao `MAXCHL`. |
| Mensagem perdida entre SQS e microserviço | Alta | Muito Baixa | SQS com DLQ. Visibility timeout acima do SLA máximo. |
| `TaskToken` expirado | Média | Baixa | `SendTaskHeartbeat` periódico durante processamentos longos. |
| Incompatibilidade `allclient` em upgrade | Média | Média | Versão fixada no BOM. Testes de regressão no CI/CD. |

---

## 8. Referências Técnicas

- **IBM Developer** — *Running your IBM MQ apps on Quarkus and GraalVM using QPid AMQP JMS classes*
  Soheel Chughtai, Avinash V G, Richard J. Coppen — Jun/2025
  `https://developer.ibm.com/tutorials/mq-running-ibm-mq-apps-on-quarkus-and-graalvm-using-qpid-amqp-jms-classes/`
  > Lido integralmente. Confirma que GraalVM + IBM MQ requer AMQP 1.0 via Qpid — não o `allclient`. Base para descarte da Alternativa 1.

- **IBM Developer** — *Building cloud-native reactive Java messaging applications using Quarkus, Smallrye, and IBM MQ*
  Richard J. Coppen, Max Kahan — Fev/2025
  `https://developer.ibm.com/tutorials/mq-building-cloud-native-reactive-java-messaging-applications/`
  > Lido integralmente. Confirma stack Smallrye + vert.x AMQP e incompatibilidade do modelo reativo com semântica síncrona dos Service Tasks Camunda.

- **AWS Documentation** — Step Functions SQS SDK Integration (`.waitForTaskToken`)
  `https://docs.aws.amazon.com/step-functions/latest/dg/connect-sqs.html`

- **IBM Documentation** — IBM MQ Spring Boot Starter (oficial)
  `https://github.com/ibm-messaging/mq-jms-spring`
  `https://www.ibm.com/docs/en/ibm-mq/9.3?topic=spring-using-mq-jms-starter`

- **GraalVM Documentation** — Reflection in Native Image
  `https://www.graalvm.org/latest/reference-manual/native-image/dynamic-features/Reflection/`

- **KEDA** — SQS Scaler para Kubernetes: `https://keda.sh/docs/scalers/aws-sqs-queue/`

- **Richardson, C.** (2018). *Microservices Patterns*. Manning. — Cap. 3: Messaging patterns

- **Newman, S.** (2021). *Building Microservices*, 2ª Ed. O'Reilly. — Cap. 4: Async communication

- **Evans, E.** (2003). *Domain-Driven Design*. Addison-Wesley. — Anti-Corruption Layer pattern

---

## 9. Conclusão e Próximos Passos

> ✅ **DECISÃO FINAL:** A **Alternativa 3** (Step Function → SQS → Integration Adapter EKS Spring Boot → IBM MQ) é a arquitetura recomendada. O **IBM MQ Spring Boot Starter** com `allclient` (protocolo nativo IBM MQ) é a forma de conectividade recomendada — preservando suporte completo a seletores JMS, transações e correlação, sem necessidade de alterações no broker legado.

A Alternativa 1 foi descartada não por desconhecimento do caminho GraalVM, mas porque o único caminho IBM-documentado para native image (Qpid/AMQP) é incompatível com três requisitos concretos e verificáveis na documentação oficial IBM (Seção 8): protocolo nativo do broker legado, features JMS de correlação, e semântica síncrona dos Service Tasks.

### Próximos Passos

1. **Processo piloto** — Seleção de um processo Camunda para PoC da Alternativa 3
2. **Módulo base** — Integration Adapter Spring Boot com configuração externalizada via SSM/AppConfig
3. **IaC** — Pipeline Terraform/CDK para SQS + DLQ por processo
4. **Observabilidade** — X-Ray + Micrometer Tracing + Grafana end-to-end
5. **Capacidade MQ** — Alinhamento com infra legada para dimensionamento do `MAXCHL`
6. **ARB** — Apresentação ao Architecture Review Board para aprovação formal

---

*Versão 2.0 — Alternativa 1 revisada com base na leitura integral dos tutoriais oficiais IBM Developer (Seção 8).*

*Time de Arquitetura Técnica — Março 2026*
