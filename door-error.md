# Erro: Porta já está em uso

O erro significa que **a porta 9092 (ou 9644) já está sendo usada por outro processo** no seu Ubuntu.

## Soluções

### Opção 1: Liberar as portas (Recomendado)

```bash
# Descobrir qual processo está usando a porta 9092
sudo lsof -i :9092

# Se encontrar algo, mate o processo
sudo kill -9 <PID>

# Fazer o mesmo para a porta 9644
sudo lsof -i :9644
sudo kill -9 <PID>

# Deletar o cluster anterior
kind delete cluster

# Tentar criar novamente
kind create cluster --config kind-cluster.yaml
```

### Opção 2: Usar portas diferentes

Se não conseguir liberar as portas, edite o arquivo `kind-cluster.yaml` com portas livres:

```bash
cat > kind-cluster.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
    - containerPort: 30000
      hostPort: 9093          # Mudou de 9092
      protocol: TCP
    - containerPort: 30001
      hostPort: 9645          # Mudou de 9644
      protocol: TCP
  - role: worker
  - role: worker
EOF
```

Depois atualize também o arquivo `redpanda-services.yaml`:

```bash
cat > redpanda-services.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: redpanda
  namespace: redpanda
  labels:
    app: redpanda
spec:
  clusterIP: None
  ports:
  - port: 9092
    name: kafka
  - port: 33145
    name: rpc
  - port: 9644
    name: admin
  selector:
    app: redpanda
---
apiVersion: v1
kind: Service
metadata:
  name: redpanda-external
  namespace: redpanda
spec:
  type: NodePort
  selector:
    app: redpanda
  ports:
  - name: kafka
    port: 9092
    targetPort: 9092
    nodePort: 30000
  - name: admin
    port: 9644
    targetPort: 9644
    nodePort: 30001
EOF
```

E teste com as novas portas:

```bash
curl http://localhost:9645/v1/status/ready
```

### Opção 3: Ver qual processo está ocupando

```bash
# Listar todas as portas em uso
sudo netstat -tlnp | grep -E "9092|9644"

# Ou
sudo ss -tlnp | grep -E "9092|9644"
```

## Após resolver

```bash
# Deletar cluster anterior
kind delete cluster

# Criar novo cluster
kind create cluster --config kind-cluster.yaml

# Continuar com o deploy
kubectl apply -f namespace.yaml
kubectl apply -f redpanda-config.yaml
kubectl apply -f redpanda-services.yaml
kubectl apply -f redpanda-statefulset.yaml
```

**Dica:** Se tiver muitos processos Docker/containers antigos, limpe-os:

```bash
docker system prune -a --volumes
```