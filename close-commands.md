## 🧩 primeiro: o que está a correr

O **kind** (Kubernetes in Docker) cria **containers Docker** que são os nós do cluster.
Se fizeres:

```bash
docker ps
```

vais ver algo assim:

```
CONTAINER ID   IMAGE                  NAMES
a1b2c3d4e5f6   kindest/node:v1.30.0   kind-control-plane
b2c3d4e5f6g7   kindest/node:v1.30.0   kind-worker
c3d4e5f6g7h8   kindest/node:v1.30.0   kind-worker2
```

Ou seja: **o teu cluster Kubernetes está a correr dentro de containers Docker.**

---

## 🧱 Opção 1 — Pausar temporariamente (recomendado se vais voltar a usar)

Se só queres **parar tudo** (sem apagar dados), basta **parar os containers** do kind:

```bash
kind stop cluster
```

Se estiveres com uma versão antiga do kind que ainda não suporta `stop`, podes usar Docker diretamente:

```bash
docker stop kind-control-plane kind-worker kind-worker2
```

👉 Isso **pausa o cluster**, libertando CPU/memória, mas mantém tudo (pods, volumes, PVCs, configurações).

Depois, para voltar:

```bash
docker start kind-control-plane kind-worker kind-worker2
```

ou simplesmente:

```bash
kind start cluster
```

---

## 💣 Opção 2 — Apagar completamente o cluster

Se queres **remover tudo** (pods, volumes, namespaces, etc.):

```bash
kind delete cluster
```

👉 Isso **deleta todos os containers** e **os dados armazenados dentro do kind** (inclusive os PVCs do Redpanda).
É o equivalente a “desligar e formatar” o teu cluster.

---

## ⚙️ Opção 3 — Parar só os pods (sem desligar o cluster)

Se queres **deixar o cluster ligado**, mas **parar o Redpanda**:

```bash
kubectl delete -n redpanda statefulset redpanda
```

ou

```bash
kubectl scale statefulset redpanda -n redpanda --replicas=0
```

👉 Isso remove (ou reduz a zero) os pods do Redpanda, mas **mantém os volumes (PVCs)**, caso queiras voltar a subir depois:

```bash
kubectl scale statefulset redpanda -n redpanda --replicas=3
```

---

## 🧭 Resumo rápido

| O que queres fazer                           | Comando                                               | O que acontece                     |
| -------------------------------------------- | ----------------------------------------------------- | ---------------------------------- |
| **Pausar tudo temporariamente**              | `kind stop cluster` ou `docker stop kind-*`           | Cluster desliga, mas mantém dados  |
| **Apagar tudo (reset total)**                | `kind delete cluster`                                 | Remove cluster e dados             |
| **Parar só Redpanda (sem desligar cluster)** | `kubectl scale sts/redpanda -n redpanda --replicas=0` | Para pods do Redpanda, mantém PVCs |
| **Subir de novo Redpanda**                   | `kubectl scale sts/redpanda -n redpanda --replicas=3` | Recria pods usando os mesmos dados |

---

💡 **Dica útil:**
Se quiseres confirmar que o cluster desligou, faz:

```bash
kubectl get nodes
```

Deve aparecer um erro tipo:

```
The connection to the server localhost:6443 was refused
```

➡️ isso significa que o servidor Kubernetes (dentro do kind) está parado.
