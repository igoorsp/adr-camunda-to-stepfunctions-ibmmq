# OpenShift: Services, Routes e Exposição de Portas

## Conceitos Fundamentais

### O que é um Service?
Um **Service** é uma abstração que expõe um conjunto de Pods como um endpoint de rede estável. Como Pods são efêmeros (morrem e nascem com IPs diferentes), o Service fornece um IP/DNS fixo e faz load balancing entre os Pods saudáveis. É um conceito herdado do Kubernetes.

### O que é um Route?
Um **Route** é específico do OpenShift (equivalente ao Ingress do Kubernetes). Ele expõe um Service externamente via HTTP/HTTPS, atribuindo um hostname público (ex: `xpto.com.corp`) e é gerenciado pelo HAProxy do OpenShift.

> **Importante:** O Route só trabalha nas portas **80 e 443**. O roteamento é feito por `host + path`, não por porta.

---

## Hierarquia e Fluxos

### Plano de controle (criação dos recursos)
```
DeploymentConfig → ReplicationController → POD
```

### Plano de dados (tráfego)
```
Internet → Route → Service → POD
```

O Service encontra os Pods diretamente por **label selectors**, sem passar pelo DeploymentConfig ou ReplicationController.

---

## Um Service pode ter múltiplas portas?

**Sim!** Um Service suporta múltiplas portas sem nenhum problema. Cada porta recebe um nome, e esse nome é usado pelos Routes para roteamento correto.

```yaml
ports:
  - name: http      # porta da aplicação
    port: 8080
    targetPort: 8080
  - name: probes    # porta de healthcheck
    port: 8020
    targetPort: 8020
```

---

## Quando criar um Service separado?

| Situação | Service separado? |
|---|---|
| Só expor outra porta da mesma app | ❌ Não precisa |
| Controle de acesso diferente por porta | ✅ Faz sentido |
| NetworkPolicy granular por porta | ✅ Faz sentido |
| Times/equipes diferentes gerenciando | ✅ Faz sentido |

---

## Roteamento por Path no mesmo Host

Quando você precisa expor endpoints em portas diferentes mas no mesmo host, a solução é usar **path-based routing**. O HAProxy prioriza paths mais específicos, então `/liveness` tem prioridade sobre `/`.

```
https://xpto.com.corp/            → POD:8080  (app principal)
https://xpto.com.corp/liveness    → POD:8020  (probe)
https://xpto.com.corp/readiness   → POD:8020  (probe)
```

### Como o Route sabe qual porta usar?

O campo `targetPort` no Route **não é um número** — é o **nome da porta** definida no Service. Esse nome é o que faz a ligação correta:

```
Route (targetPort: probes)
        ↓
Service (name: probes → port: 8020)
        ↓
POD:8020
```

---

## Manifestos Completos

### DeploymentConfig

```yaml
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: xpto
  namespace: meu-namespace
spec:
  replicas: 2
  selector:
    app: xpto
  template:
    metadata:
      labels:
        app: xpto
    spec:
      containers:
        - name: xpto
          image: minha-imagem:latest
          ports:
            - name: http
              containerPort: 8080
            - name: probes
              containerPort: 8020
          livenessProbe:
            httpGet:
              path: /liveness
              port: 8020
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /readiness
              port: 8020
            initialDelaySeconds: 5
            periodSeconds: 5
```

> **Nota:** As probes `livenessProbe` e `readinessProbe` são usadas pelo **kubelet** do Kubernetes, que acessa o Pod **diretamente** na porta 8020, sem passar por Service ou Route.

---

### Service (único com duas portas)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: xpto-service
  namespace: meu-namespace
spec:
  selector:
    app: xpto
  ports:
    - name: http
      port: 8080
      targetPort: 8080
    - name: probes
      port: 8020
      targetPort: 8020
```

---

### Route principal

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-xpto
  namespace: meu-namespace
spec:
  host: xpto.com.corp
  path: /
  port:
    targetPort: http      # aponta para porta nomeada "http" = 8080
  to:
    kind: Service
    name: xpto-service
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

---

### Route liveness

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-xpto-liveness
  namespace: meu-namespace
spec:
  host: xpto.com.corp
  path: /liveness
  port:
    targetPort: probes    # aponta para porta nomeada "probes" = 8020
  to:
    kind: Service
    name: xpto-service
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

---

### Route readiness

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-xpto-readiness
  namespace: meu-namespace
spec:
  host: xpto.com.corp
  path: /readiness
  port:
    targetPort: probes    # aponta para porta nomeada "probes" = 8020
  to:
    kind: Service
    name: xpto-service
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

---

## Fluxo Final Completo

```
# Aplicação principal
https://xpto.com.corp/
  → HAProxy (route-xpto, targetPort: http)
  → xpto-service:8080
  → POD:8080

# Liveness (acesso manual ou monitoramento)
https://xpto.com.corp/liveness
  → HAProxy (route-xpto-liveness, targetPort: probes)
  → xpto-service:8020
  → POD:8020

# Readiness (acesso manual ou monitoramento)
https://xpto.com.corp/readiness
  → HAProxy (route-xpto-readiness, targetPort: probes)
  → xpto-service:8020
  → POD:8020

# Probes automáticas do Kubernetes (sem Route ou Service)
kubelet → POD:8020 (direto)
```

---

## Boas Práticas

- **Nomeie sempre as portas** no Service e no DeploymentConfig — facilita a leitura e evita erros nos Routes
- **Use `targetPort` pelo nome**, não pelo número — mais legível e menos propenso a erros
- **TLS sempre ativo** em produção com `termination: edge` e `insecureEdgeTerminationPolicy: Redirect`
- **Restrinja por IP** endpoints sensíveis como probes usando a annotation do HAProxy:
  ```yaml
  metadata:
    annotations:
      haproxy.router.openshift.io/ip_whitelist: "10.0.0.0/8"
  ```
- **Use `oc port-forward`** para testes pontuais sem precisar criar Routes permanentes:
  ```bash
  oc port-forward pod/xpto-pod-xyz 8020:8020
  curl http://localhost:8020/liveness
  ```
