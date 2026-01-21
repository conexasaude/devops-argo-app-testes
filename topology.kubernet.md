# Teste GitOps + HPA + ArgoCD (replicas, AZ, Spot)

Este teste valida o comportamento do Kubernetes + ArgoCD quando usamos **HPA** para controlar `spec.replicas`, além de confirmar:
- valor padrão de `replicas` quando não definido no Deployment
- conflito ArgoCD × HPA (e como resolver)
- espalhamento de pods por **AZ**
- preferência por **nodes Spot**
- convivência com **amd64 + arm64**

---

## 🎯 Objetivos do teste

1. Confirmar que:
   - sem `replicas` no Deployment → Kubernetes usa **1 por padrão**
   - com HPA → **HPA passa a controlar `spec.replicas`**
2. Validar comportamento do ArgoCD:
   - sem `ignoreDifferences` → pode gerar **OutOfSync**
   - com `ignoreDifferences + RespectIgnoreDifferences` → fica **Synced**
3. Verificar:
   - pods distribuídos em **AZs diferentes**
   - uso de **Spot instances**
   - suporte a **amd64 e arm64**

---

## 📦 Setup utilizado

- **Controller GitOps**: ArgoCD
- **Autoscaling**: HPA (CPU)
- **App de teste**: nginx
- **Cluster**: EKS (sa-east-1)
- **Provisionamento de nodes**: Karpenter
- **Namespace**: `nginx-hpa-test`

---

## 🧪 Manifestos usados

### Deployment
- **Sem `spec.replicas`**
- Health checks via HTTP (`/`)
- `topologySpreadConstraints` por AZ
- `nodeAffinity` preferindo Spot
- `podAntiAffinity` por hostname

### HPA
```yaml
minReplicas: 3
maxReplicas: 6
metrics:
  - cpu averageUtilization: 30
```


## 🔎 Verificar AZ + Spot/On-Demand + Arquitetura (pod → node)

Este comando mostra, para cada pod do teste:
- **NODE** onde ele está rodando
- **ZONE (AZ)** do node (`topology.kubernetes.io/zone`)
- **CAPACITY TYPE** do Karpenter (`karpenter.sh/capacity-type`) → `spot` ou `on-demand`
- **ARCH** do node (`kubernetes.io/arch`) → `amd64` ou `arm64`

### ✅ Comando

```bash
kubectl -n conexa get pods -l app=nginx-hpa-test \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}' \
| while read -r POD NODE; do
  ZONE=$(kubectl get node "$NODE" -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}')
  CAP=$(kubectl get node "$NODE" -o jsonpath='{.metadata.labels.karpenter\.sh/capacity-type}')
  ARCH=$(kubectl get node "$NODE" -o jsonpath='{.metadata.labels.kubernetes\.io/arch}')
  echo -e "$POD\t$NODE\t$ZONE\t$CAP\t$ARCH"
done | column -t
````

### ✅ Exemplo de saída (resultado real do teste)

```text
nginx-hpa-test-54bcdb89f5-4scm4  ip-10-20-20-228.sa-east-1.compute.internal  sa-east-1b  spot  arm64
nginx-hpa-test-54bcdb89f5-kcwjp  ip-10-20-17-149.sa-east-1.compute.internal  sa-east-1a  spot  amd64
nginx-hpa-test-54bcdb89f5-v766s  ip-10-20-27-242.sa-east-1.compute.internal  sa-east-1c  spot  amd64
nginx-hpa-test-54bcdb89f5-wkxj7  ip-10-20-22-128.sa-east-1.compute.internal  sa-east-1b  spot  arm64
```

### ✅ Interpretação rápida

* **AZ**: está espalhado entre `sa-east-1a`, `sa-east-1b`, `sa-east-1c`
* **Spot**: todos os nodes estão como `spot` (fallback para on-demand não foi necessário nesse teste)
* **Arquitetura**: mistura de `amd64` e `arm64` (testando multi-arch)
* `sa-east-1b` aparece duas vezes porque com 4 pods e 3 AZs, a distribuição mais comum é **2 / 1 / 1** (respeitando `maxSkew: 1`)



Perfeito 👍
Segue a explicação **no formato `.md`**, pronta pra você colar no repositório (ex: `SCHEDULING.md` ou no próprio `README.md`).


# Entendendo Affinity, Anti-Affinity e Spread por AZ (Kubernetes)

Este documento explica as regras de **agendamento de pods** usadas no teste `nginx-hpa-test`
para alcançar **alta disponibilidade**, **uso de Spot**, e **distribuição por AZ**.

---

## 📌 Bloco analisado

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: karpenter.sh/capacity-type
              operator: In
              values: ["spot"]

    # requiredDuringSchedulingIgnoredDuringExecution:
    #   nodeSelectorTerms:
    #   - matchExpressions:
    #     - key: kubernetes.io/arch
    #       operator: In
    #       values: ["amd64"]

  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: nginx-hpa-test
          topologyKey: kubernetes.io/hostname

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: nginx-hpa-test
````

---

## 1️⃣ Node Affinity — escolher **QUE TIPO DE NODE** usar

### Preferir Spot (soft rule)

```yaml
preferredDuringSchedulingIgnoredDuringExecution
```

📖 Significado:

* O scheduler **tenta** rodar o pod em nodes Spot
* Se **não houver Spot disponível**, ele **pode usar On-Demand**
* Nunca deixa o pod em `Pending` por falta de Spot

✅ Comportamento desejado:

> **Spot se tiver, senão On-Demand**

---

### Forçar arquitetura (hard rule – opcional)

```yaml
requiredDuringSchedulingIgnoredDuringExecution
```

📖 Significado:

* Só agenda em nodes com a arquitetura especificada (`amd64` ou `arm64`)
* Se não existir node compatível → pod fica **Pending**

🧪 Usado apenas para testes de arquitetura.

---

## 2️⃣ Pod Anti-Affinity — evitar pods no mesmo node

```yaml
podAntiAffinity:
  topologyKey: kubernetes.io/hostname
```

📖 Significado:

* Evita colocar **dois pods iguais no mesmo node**
* O domínio aqui é o **hostname (node)**

🔹 `preferred` (soft):

* Tenta espalhar
* Se não tiver node suficiente, ainda agenda

✅ Resultado observado:

* Um pod por máquina sempre que possível

---

## 3️⃣ Topology Spread Constraints — espalhar por AZ

```yaml
topologyKey: topology.kubernetes.io/zone
```

📖 Significado:

* Espalha pods entre **Availability Zones**
* Ajuda a evitar perda total da aplicação caso uma AZ caia

### `maxSkew: 1`

* Diferença máxima de pods entre AZs é **1**
* Exemplo válido com 4 pods / 3 AZs:

  * 2 / 1 / 1

### `whenUnsatisfiable: ScheduleAnyway`

* Se não der pra balancear perfeitamente:

  * **agenda mesmo assim**
  * evita pod `Pending`

---

## 🤝 Como essas regras trabalham juntas

Quando o HPA escala os pods, o scheduler tenta, **nesta ordem**:

1. Preferir **Spot** (nodeAffinity)
2. Evitar 2 pods no **mesmo node** (podAntiAffinity)
3. Espalhar por **AZ** (topologySpreadConstraints)

---

## ✅ Resultado prático observado

* Pods distribuídos entre `sa-east-1a`, `sa-east-1b`, `sa-east-1c`
* Todos rodando em **Spot**
* Mistura de `amd64` e `arm64`
* Um pod por node quando possível

---

## ⚖️ Soft vs Hard (Resumo)

| Tipo | Keyword     | Comportamento                     |
| ---- | ----------- | --------------------------------- |
| Soft | `preferred` | Tenta respeitar, mas não bloqueia |
| Hard | `required`  | Obrigatório, pode deixar Pending  |

---

## 🧠 Boas práticas recomendadas

* ✅ `preferred` para Spot (fallback automático)
* ✅ `podAntiAffinity` para HA sem risco de Pending
* ✅ `topologySpreadConstraints` para HA entre AZs
* ❌ Não usar regras `required` sem necessidade real

---

## 🔗 Referências oficiais

* Kubernetes – Node Affinity
  [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)

* Kubernetes – Pod Anti-Affinity
  [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)

* Kubernetes – Topology Spread Constraints
  [https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)

```

---

Se quiser, no próximo passo eu posso:
- 🔖 criar um **diagrama visual** desse agendamento
- 🧩 gerar uma **policy padrão** pro time (copiar/colar)
- 📦 separar isso em **README + SCHEDULING.md + HPA.md**

Esse material já está em nível **documentação de time senior** 👌
```
