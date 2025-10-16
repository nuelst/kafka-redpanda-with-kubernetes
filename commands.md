
# Comandos `kubectl exec … rpk …` (administrar o Redpanda de dentro do pod)

> Padrão: `kubectl exec -it <pod> -n redpanda -- rpk <subcomando …>`
> `-it` abre sessão interativa; `--` separa argumentos do `kubectl` dos do `rpk`.

### 1) Consumir mensagens

```
kubectl exec -it redpanda-0 -n redpanda -- rpk topic consume test-topic
```

**Para que serve:** lê (consome) mensagens do tópico `test-topic` a partir do offset atual (ou earliest, se configurado).
**Quando usar:** testar rapidamente se há mensagens chegando, depurar consumidores, verificar conteúdo de um tópico sem precisar escrever um cliente externo.

### 2) Produzir mensagens

```
kubectl exec -it redpanda-0 -n redpanda -- rpk topic produce test-topic
```

**Para que serve:** produz mensagens para `test-topic` (o `rpk` abre um prompt para você digitar linhas que viram mensagens).
**Quando usar:** smoke test do cluster/tópico, validar conectividade e permissões, fazer um “hello world” sem criar uma app.

### 3) Status do cluster

```
kubectl exec -it redpanda-0 -n redpanda -- rpk status
```

**Para que serve:** mostra um resumo do cluster: brokers, saúde, versão, etc.
**Quando usar:** checar rapidamente se todos os nós estão “up” e se o `rpk` consegue falar com o cluster.

> ⚠️ Você também escreveu `rpk stat`. Normalmente o subcomando é `rpk status`. Se `rpk stat` falhar, use `rpk status`.

### 4) Descrever um tópico

```
kubectl exec -it redpanda-0 -n redpanda -- rpk topic describe test-topic
```

**Para que serve:** exibe metadados do tópico (partições, líderes, réplicas, configs).
**Quando usar:** entender distribuição das partições, fator de replicação, checar em qual broker está a liderança, diagnosticar desequilíbrio/replicação.

---

# Aplicar manifests e acompanhar o rollout

### 5) Aplicar recursos (na ordem)

```
kubectl -n redpanda apply -f namespace.yaml
kubectl -n redpanda apply -f redpanda-config.yaml
kubectl -n redpanda apply -f redpanda-services.yaml
kubectl -n redpanda apply -f redpanda-statefulset.yaml
```

**Para que serve:** cria/atualiza objetos do Kubernetes a partir dos YAMLs.

**Quando usar:** em deploy inicial e atualizações.

**Dicas importantes:**

* **Namespace:** o recurso `Namespace` **não pertence** a nenhum namespace. O mais correto é:

  ```
  kubectl apply -f namespace.yaml
  ```

  e só depois usar `-n redpanda` para os demais.
* **Ordem:** `namespace` → `services/configmap` → `statefulset` é a sequência saudável.

### 6) Acompanhar o rollout do StatefulSet

```
kubectl -n redpanda rollout status sts/redpanda
```

**Para que serve:** acompanha até que todos os pods do StatefulSet estejam prontos.
**Quando usar:** depois de aplicar/atualizar o StatefulSet, para saber se subiu bem.

---

# Checar saúde pela Admin API

### 7) Readiness HTTP (fora do cluster, via kind/NodePort)

```
curl http://localhost:9644/v1/status/ready
```

**Para que serve:** endpoint de readiness do Redpanda; retorna “ready” quando o nó está OK.
**Quando usar:** health-check manual, troubleshooting de probes, confirmar que o mapeamento do NodePort (ou LB) está correto.

---

# Operações de Kubernetes gerais

### 8) Observar pods ao vivo

```
kubectl get pods -n redpanda -w
```

**Para que serve:** “watch” de pods mudando de estado em tempo real.
**Quando usar:** durante rollout, debugging, queda/restart de pods.

### 9) Ver classes de storage

```
kubectl get storageclass
```

**Para que serve:** lista StorageClasses disponíveis e a default (essencial para PVCs do StatefulSet).
**Quando usar:** antes de aplicar o StatefulSet ou quando PVCs ficam `Pending`.

---

# Limpeza e destruição (cuidado!)

### 10) Deletar o StatefulSet

```
kubectl delete statefulset redpanda -n redpanda
```

**Para que serve:** remove **apenas** o StatefulSet (os PVCs e PVs **permanecem**, por padrão).
**Quando usar:** recriar o controlador mantendo dados ou para mudar espec de pod sem perder disco.

### 11) Deletar PVCs dos pods Redpanda

```
kubectl delete pvc -l app=redpanda -n redpanda
```

**Para que serve:** apaga **os volumes persistentes** usados pelos pods.
**Quando usar:** **reset TOTAL de dados** do cluster, ambientes de teste, quando precisa começar do zero.

> ⚠️ **Isto remove dados definitivamente** (tópicos, offsets, everything). Use com extremo cuidado.

### 12) Deletar o ConfigMap

```
kubectl delete configmap redpanda-config -n redpanda
```

**Para que serve:** remove o ConfigMap (se estiver montado, os pods podem precisar de restart).
**Quando usar:** vai substituir config via flags, ou atualizar o config e re-aplicar outro.

---

# Boas práticas e atalhos

* **Port-forward (alternativa ao NodePort/LB):**

  ```
  kubectl -n redpanda port-forward svc/redpanda 9644:9644
  ```

  Útil para testar Admin API rapidamente sem expor NodePort.

* **Logs de um pod (diagnóstico):**

  ```
  kubectl logs -n redpanda redpanda-0
  ```

* **Executar `rpk` em qualquer pod do cluster:** você usou `redpanda-0`, mas pode ser `-1` ou `-2`. Prefira um **líder** para comandos sensíveis.

* **Criação de tópico rápida:**

  ```
  kubectl exec -it redpanda-0 -n redpanda -- \
    rpk topic create test-topic --partitions 3 --replication 3
  ```

* **Ver estado de partições/replicação:**

  ```
  kubectl exec -it redpanda-0 -n redpanda -- rpk cluster metadata
  ```



PVCs: PVCs (ou PersistentVolumeClaims) são um conceito central no Kubernetes quando falamos de armazenamento persistente, como no caso do Redpanda.

PV (PersistentVolume) = o disco físico ou lógico real (um volume, um path NFS, um disco EBS, etc.);

PVC (PersistentVolumeClaim) = a requisição feita pelo app para obter um PV que atenda aos requisitos (tamanho, modo de acesso);

StorageClass = define como os PVs são criados automaticamente (dinamicamente).

kubectl get pvc -n redpanda



| Modo                 | Sigla  | Significado                                                                                                               | Pode ser montado por                  | Cenário típico                                                                           |
| -------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | ---------------------------------------------------------------------------------------- |
| **ReadWriteOnce**    | `RWO`  | O volume pode ser montado **por apenas um nó** (pod) **com leitura e escrita**.                                           | um único pod por vez, em um único nó. | Bancos de dados, Redpanda, Kafka, Redis, etc.                                            |
| **ReadOnlyMany**     | `ROX`  | O volume pode ser montado **em muitos nós/pods ao mesmo tempo**, mas **apenas leitura**.                                  | múltiplos pods, leitura-only.         | Compartilhar dados estáticos, arquivos de configuração.                                  |
| **ReadWriteMany**    | `RWX`  | O volume pode ser montado **em muitos nós/pods ao mesmo tempo** com **leitura e escrita**.                                | múltiplos pods simultaneamente.       | Aplicações que precisam compartilhar arquivos, logs centralizados, CMS, etc.             |
| **ReadWriteOncePod** | `RWOP` | Novo modo (Kubernetes 1.22+). O volume é montado **por apenas um pod específico** (mesmo que outros estejam no mesmo nó). | um único pod.                         | Controlar concorrência mais rígida que RWO; útil em ambientes com múltiplos pods por nó. |




| Conceito         | Explicação                                                                 |
| ---------------- | -------------------------------------------------------------------------- |
| **Node**         | É como um “computador” dentro do cluster (no kind, é um container Docker). |
| **Pod**          | É o “programa” que roda dentro de um node.                                 |
| **PVC**          | É o “disco” do pod, ligado ao node.                                        |
| **Cluster Kind** | É o “conjunto” de nodes que roda tudo.                                     |
| **Distribuição** | O Kubernetes espalha os pods entre os nodes disponíveis.                   |




👉 em resumo:

No teu cluster kind, os três pods do Redpanda (redpanda-0, -1, -2) são distribuídos automaticamente pelos três nodes (control-plane, worker1 e worker2).
Cada um roda o seu container Redpanda e usa o seu próprio disco (PVC) local.