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



Perfeito — vamos adicionar isso no README como uma seção “comando completo + exemplo de output” ✅
Aqui vai um bloco pronto pra você colar no `.md` (já com explicação do que cada coluna significa):

````md
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

```

Se você quiser, eu também te mando uma seção curta explicando **por que 1 pod por node** (podAntiAffinity) e **por que 2 pods podem cair na mesma AZ** (maxSkew=1).
::contentReference[oaicite:0]{index=0}
```
