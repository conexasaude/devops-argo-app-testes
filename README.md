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
