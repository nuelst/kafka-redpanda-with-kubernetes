# Guia Completo: Comandos Kubernetes para Redpanda

## Conceitos Fundamentais

### O que é um Cluster Kubernetes?
Um **cluster** é como um **computador virtual gigante** feito de vários computadores (nós) trabalhando juntos. É o ambiente onde seus aplicativos rodam.

**Analogia:** Se a nuvem fosse um prédio, o cluster seria todo o prédio.

### O que é um Namespace?
Um **namespace** é como uma **pasta/diretório lógico** dentro do cluster que isola e organiza recursos.

**Analogia:** Se o cluster é um prédio, o namespace é um andar específico. Você pode ter múltiplos andares sem que se interfiram.

---

## Passo 1: Criar o Cluster

### Comando
```bash
kind create cluster --config kind-cluster.yaml
```

### O que faz?
Cria um cluster Kubernetes **completo e funcional** dentro do Docker. Usando o arquivo `kind-cluster.yaml`, ele provisiona:
- 1 nó control-plane (mestre que coordena tudo)
- 2 nós workers (executam suas aplicações)
- Rede Docker isolada
- Mapeamento de portas para acesso externo

### Por que precisa?
Sem um cluster, não há lugar para rodar seus aplicativos. É como tentar instalar um programa sem ter um computador.

### Quanto tempo leva?
**2-5 minutos** (primeira vez mais lento, pré-carrega a imagem do Kubernetes)

### Resultado esperado
```
✓ Ensuring node image (kindest/node:v1.34.0)
✓ Preparing nodes
✓ Writing configuration
✓ Starting control-plane
✓ Installing CNI
✓ Installing StorageClass
✓ Waiting for control plane to be ready
✓ Join the workers
Set kubectl context to "kind-kind"
```

---

## Passo 2: Criar o Namespace

### Comando
```bash
kubectl apply -f namespace.yaml
```

### O que faz?
O `kubectl apply` **envia uma instrução para o cluster** executar. Neste caso, cria um namespace chamado "redpanda".

**Detalhe:** Se o namespace já existe, o comando não faz nada (é idempotente).

### Por que precisa?
Organização e isolamento. Se você tiver 10 aplicações diferentes, pode colocar cada uma em um namespace separado.

### Resultado esperado
```
namespace/redpanda created
```

---

## Passo 3: Aplicar Configurações (3 comandos)

### 3.1 - ConfigMap
```bash
kubectl apply -f redpanda-config.yaml
```

**O que faz:** Cria um arquivo de configuração que o Redpanda vai usar (portas, endereços, modo developer, etc.)

**Por que:** Redpanda precisa saber como se comportar (onde escutar, que portas usar, etc.)

**Resultado:** 
```
configmap/redpanda-config created
```

### 3.2 - Services
```bash
kubectl apply -f redpanda-services.yaml
```

**O que faz:** Cria dois serviços (como portas de entrada):
- **redpanda (headless):** Para comunicação interna entre os 3 nós
- **redpanda-external (NodePort):** Para acesso do seu computador às portas 9092 e 9644

**Por que:** Sem services, suas aplicações não conseguem encontrar o Redpanda. É como uma placa de endereço.

**Resultado:**
```
service/redpanda created
service/redpanda-external created
```

### 3.3 - StatefulSet
```bash
kubectl apply -f redpanda-statefulset.yaml
```

**O que faz:** Cria e gerencia 3 **pods** (containers) do Redpanda com:
- Armazenamento persistente (dados não desaparecem se o pod reiniciar)
- Health checks (verifica se está saudável)
- Configuração automática de cada nó

**Por que:** StatefulSets garantem que cada nó tenha identidade fixa e dados persistentes. Se um nó cair, ele volta com seus dados intactos.

**Diferença de Deployment:** 
- **Deployment:** Para aplicações stateless (sem estado) - qualquer pod pode substituir outro
- **StatefulSet:** Para aplicações stateful (com estado) - cada pod é único

**Resultado:**
```
statefulset.apps/redpanda created
```

---

## Passo 4: Monitorar o Deploy

### Comando
```bash
kubectl get pods -n redpanda -w
```

### O que faz?
Lista todos os pods no namespace "redpanda" em **modo watch** (`-w`). Atualiza em tempo real conforme mudam.

### O que procurar
```
NAME         READY   STATUS    RESTARTS   AGE
redpanda-0   1/1     Running   0          2m
redpanda-1   1/1     Running   0          1m
redpanda-2   1/1     Running   0          45s
```

**READY 1/1:** Pod pronto e funcionando
**STATUS Running:** Pod está rodando
**AGE:** Tempo desde que foi criado

**Estados possíveis:**
- `Pending:` Aguardando recursos
- `ContainerCreating:` Baixando imagem Docker
- `Running:` Funcionando normalmente
- `CrashLoopBackOff:` Pod com erro, tentando reiniciar

### Quanto tempo leva?
**2-3 minutos** para todos os 3 pods ficarem prontos

### Como sair?
Pressione `Ctrl+C`

---

## Comandos de Verificação (GET)

### Ver status dos pods
```bash
kubectl get pods -n redpanda
```

**O que faz:** Lista todos os pods uma vez (sem atualizar continuamente)

### Ver os serviços
```bash
kubectl get svc -n redpanda
```

**O que faz:** Lista os serviços criados

**Resultado esperado:**
```
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                AGE
redpanda             ClusterIP   None            <none>        9092/TCP,33145/TCP     2m
redpanda-external    NodePort    10.96.123.45    <none>        9092:30000/TCP,9644:30001/TCP   2m
```

### Ver ConfigMaps
```bash
kubectl get configmap -n redpanda
```

### Ver StatefulSets
```bash
kubectl get statefulset -n redpanda
```

### Ver volumes (armazenamento)
```bash
kubectl get pvc -n redpanda
```

---

## Testando o Redpanda

### 1. Testar Admin API
```bash
curl http://localhost:9644/v1/status/ready
```

**O que faz:** Verifica se o Redpanda está respondendo

**Resultado esperado:**
```json
{"ok":true}
```

### 2. Entrar em um pod
```bash
kubectl exec -it redpanda-0 -n redpanda -- /bin/bash
```

**O que faz:** 
- `exec:` Executa um comando dentro do pod
- `-it:` Modo interativo (você consegue digitar)
- `redpanda-0:` Nome do pod específico
- `/bin/bash:` Qual comando executar (abrir bash)

**Resultado:** Você entra dentro do container, como se fosse um `ssh`

### 3. Criar um tópico (dentro do pod)
```bash
rpk topic create test-topic
```

**O que faz:** Cria um tópico Kafka chamado "test-topic" (como uma fila de mensagens)

### 4. Listar tópicos (dentro do pod)
```bash
rpk topic list
```

**O que faz:** Lista todos os tópicos criados

### 5. Sair do pod
```bash
exit
```

---

## Limpeza

### Deletar o cluster
```bash
kind delete cluster
```

**O que faz:** Remove **tudo** - todos os pods, services, volumes, configurações, etc.

**Resultado esperado:**
```
Deleting cluster "kind"
```

---

## Fluxo Completo Visual

```
1. kind create cluster
        ↓
   [Cluster Kubernetes criado]
        ↓
2. kubectl apply namespace.yaml
        ↓
   [Namespace "redpanda" criado]
        ↓
3. kubectl apply redpanda-config.yaml
        ↓
   [Configurações do Redpanda salvas]
        ↓
4. kubectl apply redpanda-services.yaml
        ↓
   [Rotas de acesso criadas]
        ↓
5. kubectl apply redpanda-statefulset.yaml
        ↓
   [3 Pods do Redpanda iniciando...]
        ↓
6. kubectl get pods -n redpanda -w
        ↓
   [Aguarda até todos ficarem "Running"]
        ↓
7. Redpanda está pronto! ✓
```

---

## Resumo Rápido

| Comando | O que faz | Por que |
|---------|-----------|--------|
| `kind create cluster` | Cria o ambiente (cluster) | Precisa de um lugar para rodar |
| `kubectl apply` | Envia instruções para o cluster | Criar/atualizar recursos |
| `kubectl get` | Consulta status dos recursos | Verificar se tudo está ok |
| `kubectl exec` | Executa comando dentro de um pod | Para testar e diagnosticar |
| `kind delete cluster` | Remove tudo | Limpeza |

---

## Quando Você Pode Usar o Redpanda?

✅ **Quando está "Running" e "Ready 1/1"** em todos os 3 pods

Nesse momento você pode:
- Enviar mensagens para tópicos
- Consumir mensagens
- Conectar aplicações externas na porta 9092
- Usar a API de admin na porta 9644
- Fazer qualquer operação Kafka normal