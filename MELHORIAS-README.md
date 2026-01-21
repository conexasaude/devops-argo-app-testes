# Mudança recomendada: `podAntiAffinity` de **required** → **preferred** (apps com HPA)

Este documento explica **por que** mudar o agendamento (scheduling) do Deployment `conexa-api-pagamento`
para evitar **pods Pending**, principalmente em cenários com **HPA**, **Spot** e **RollingUpdate**.

---

## Contexto (manifesto atual)

No YAML atual do `conexa-api-pagamento`, você usa:

- **Spot como preferência (soft)** ✅
- **Anti-affinity como requisito (hard)** ⚠️
- **HPA minReplicas=2 / maxReplicas=3** ✅

Trecho atual:

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 1
      preference:
        matchExpressions:
        - key: "karpenter.sh/capacity-type"
          operator: "In"
          values: ["spot"]

  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - conexa-api-pagamento
      topologyKey: "kubernetes.io/hostname"
````

---

## O problema: `requiredDuringSchedulingIgnoredDuringExecution` pode deixar pod **Pending**

`requiredDuringSchedulingIgnoredDuringExecution` significa:

> **É obrigatório** agendar cada pod em um node diferente (pela `topologyKey`),
> senão o scheduler **não agenda** e o pod fica **Pending**.

Como a `topologyKey` é `kubernetes.io/hostname`, o “domínio” é o **node**.

Ou seja:

* 2 réplicas → precisa de **2 nodes disponíveis**
* 3 réplicas → precisa de **3 nodes disponíveis**

Se não tiver nodes suficientes *naquele momento*, o Kubernetes bloqueia.

---

## Cenários reais onde isso quebra (muito comum)

### 1) HPA escala para cima (ex.: 2 → 3)

* Se houver só 2 nodes elegíveis (por spot/arquitetura/capacidade), a 3ª réplica fica:

  * ❌ **Pending** (não pode compartilhar o mesmo node)

### 2) RollingUpdate com `maxSurge: 50%`

Mesmo sem HPA, o RollingUpdate pode criar pod extra temporariamente.

* Com 2 réplicas, `50%` pode tentar criar **+1 pod** durante o rollout
* Se não existir node extra elegível, esse pod pode ficar:

  * ❌ **Pending**

### 3) Spot interruption / troca de node

* Spot pode ser interrompido
* Enquanto Karpenter recria capacidade, pode faltar node suficiente
* Pods novos ficam:

  * ❌ Pending

### 4) Cluster com poucos nodes / pools segmentados

* Se você tiver pools por arch (`amd64`/`arm64`) ou por constraints de capacidade
* Pode acontecer de **existirem nodes no cluster**, mas **não elegíveis** para esse pod

---

## Observação importante: Spot já está como “preferência” (bom)

Seu `nodeAffinity` usa `preferred…` para spot. Isso é o comportamento ideal:

* ✅ tenta Spot
* ✅ se não tiver Spot, cai em On-Demand (fallback automático)
* ✅ não bloqueia agendamento por falta de Spot

O ponto de risco não é o spot — é o anti-affinity como **required**.

---

## Objetivo da mudança

Manter as vantagens de HA (espalhar) **sem** causar indisponibilidade por Pending.

Queremos:

* ✅ “Tente não colocar 2 pods no mesmo node”
* ✅ “Mas se precisar (falta capacidade), pode colocar, para não ficar Pending”

Isso é exatamente `preferredDuringSchedulingIgnoredDuringExecution`.

---

## Mudança recomendada (diff)

### ✅ Antes (hard / pode Pending)

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values:
        - conexa-api-pagamento
    topologyKey: kubernetes.io/hostname
```

### ✅ Depois (soft / não bloqueia)

```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values:
                - conexa-api-pagamento
        topologyKey: kubernetes.io/hostname
```

---

## Resultado esperado após a mudança

* ✅ Quando houver nodes suficientes: pods ficam **um por node** (HA)
* ✅ Se faltar node: o scheduler pode co-locar temporariamente (degradação controlada)
* ✅ HPA consegue escalar sem travar
* ✅ Rollout fica mais resiliente
* ✅ Menos chance de indisponibilidade em eventos de spot

---

## Quando `required` faz sentido (exceções)

Use `required` apenas se o risco de dois pods no mesmo node for inaceitável **e** você aceita Pending, por exemplo:

* quorum systems específicos
* alguns stateful muito críticos
* workloads que podem ficar Pending sem impacto (batch/Job)

Para **APIs stateless com HPA**, o padrão mais seguro é `preferred`.

---

## Referências oficiais (Kubernetes)

* Affinity/Anti-affinity (docs oficiais):
  [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)

* Node affinity / scheduling:
  [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)

* Topology Spread Constraints (espalhar por AZ):
  [https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)

---

## TL;DR (pra Slack)

> Hoje `podAntiAffinity` está como **required**, o que pode deixar pods **Pending** quando o HPA escala, durante rollout, ou quando falta node (ex.: spot interruption).
> Recomendação: mudar para **preferred** para manter o espalhamento quando possível, mas sem bloquear a aplicação.

```
::contentReference[oaicite:0]{index=0}
```
