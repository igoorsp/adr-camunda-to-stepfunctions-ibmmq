# Node.js Event Loop — Liveness Probe

**Referência de threshold para detecção de event loop bloqueado**
Time de Arquitetura · Março 2026 · v1.0

---

## 1. Contexto

O Node.js opera em **single thread**. Qualquer operação síncrona que bloqueie o event loop impede que **todas as demais requisições sejam processadas** — incluindo a requisição do kubelet ao endpoint de liveness. Se o loop estiver travado por mais do que o `timeoutSeconds` do probe, o Kubernetes reinicia o pod pelo motivo errado, dificultando o diagnóstico.

---

## 2. Como Medir o Lag

A técnica padrão é registrar o tempo entre o agendamento e a execução de um `setImmediate()`. Como ele roda na fase **check** — logo após a fase **poll** — qualquer atraso reflete bloqueio real do loop.

```js
const { monitorEventLoopDelay } = require('perf_hooks');

const histogram = monitorEventLoopDelay({ resolution: 10 });
histogram.enable();

app.get('/livez', (req, res) => {
  const lagP99Ms = histogram.percentile(99) / 1e6; // nanosegundos → ms

  if (lagP99Ms > Number(process.env.EVENT_LOOP_THRESHOLD_MS)) {
    return res.status(503).json({ status: 'unhealthy', lagP99Ms });
  }
  res.status(200).json({ status: 'ok', lagP99Ms });
});
```

> ⚠️ **Por que P99 e não a média?** O histograma **não coleta amostras durante bloqueios** — o que faz a média subestimar picos reais. O percentil P99 captura os piores 1% dos casos e é o indicador correto para liveness. (Ref.: [nodejs/node#34661](https://github.com/nodejs/node/issues/34661))

---

## 3. Thresholds de Referência

**Não existe um valor oficial definido pelo Node.js ou CNCF.** Os valores abaixo são o consenso da comunidade e das principais ferramentas de produção:

| Lag P99 | Classificação | HTTP | Referência |
|---|---|---|---|
| < 10 ms | ✅ Saudável | 200 OK | Baseline normal |
| 10 – 100 ms | ⚠️ Atenção | 200 OK + log | Monitorar; sem restart |
| > 100 ms | 🚨 Crítico | 503 | `@fastify/under-pressure` default; comunidade Node.js |

> 📌 **Por que não existe número fixo:** o threshold ideal depende do SLO da aplicação, do baseline de lag em produção e da carga esperada. Um worker assíncrono tolera mais lag do que uma API de alta frequência. O valor de **100 ms** é o consenso para uso geral — ajuste se o P99 em pico saudável for muito diferente disso.

---

## 4. Configuração Kubernetes Recomendada

```yaml
livenessProbe:
  httpGet:
    path: /livez
    port: 3000
  initialDelaySeconds: 30   # tempo de bootstrap da aplicação
  periodSeconds: 30         # não sobrecarregar o loop com probes frequentes
  timeoutSeconds: 5         # margem para o loop responder sob carga
  failureThreshold: 3       # 3 falhas = 90s de janela antes do restart
```

---

## 5. Referências

| Fonte | Link |
|---|---|
| Node.js Docs — Event Loop | https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick |
| Node.js Docs — perf_hooks | https://nodejs.org/api/perf_hooks.html |
| @fastify/under-pressure (200ms default) | https://github.com/fastify/under-pressure |
| Trigger.dev — event loop lag deepdive (100ms) | https://trigger.dev/blog/event-loop-lag |
| nodejs/node#34661 — limitação do histograma | https://github.com/nodejs/node/issues/34661 |
| Red Hat — Node.js Reference Architecture | https://github.com/nodeshift/nodejs-reference-architecture/blob/main/docs/operations/healthchecks.md |
