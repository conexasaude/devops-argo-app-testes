Sim — **é recomendação/documentação oficial do ArgoCD** ignorar `spec.replicas` quando esse campo é “controlado por outro controller” (tipo **HPA**), pra evitar **OutOfSync/loop**.

A documentação do ArgoCD mostra explicitamente:

* **Como configurar `ignoreDifferences` pra ignorar `spec.replicas` em Deployments** (exemplo no guia de *Diff Customization*). ([argo-cd.readthedocs.io][1])
* E também mostra o **Sync Option `RespectIgnoreDifferences=true`** dizendo que ele faz o Argo **ignorar `spec.replicas` durante o sync** (não só no diff). ([argo-cd.readthedocs.io][2])

### O que isso significa na prática (boa prática GitOps)

Quando você usa HPA:

* ✅ **HPA manda em `spec.replicas`**
* ✅ **Argo ignora `spec.replicas`** (porque é drift esperado)
* ✅ Você evita Argo ficar **OutOfSync** ou tentando “voltar replicas pro Git”

### E a recomendação que eu te passo (bem objetiva)

**Para workloads com HPA:**

1. **Remover `replicas:` do Deployment**
2. Deixar o mínimo no **`minReplicas` do HPA**
3. No ArgoCD, configurar **ignore `/spec/replicas`** + `RespectIgnoreDifferences=true`

Isso é exatamente o padrão que o Argo documenta nos exemplos acima. ([argo-cd.readthedocs.io][1])

[1]: https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/?utm_source=chatgpt.com "Diff Customization - Declarative GitOps CD for Kubernetes"
[2]: https://argo-cd.readthedocs.io/en/latest/user-guide/sync-options/?utm_source=chatgpt.com "Sync Options - Argo CD - Declarative GitOps CD for Kubernetes"
