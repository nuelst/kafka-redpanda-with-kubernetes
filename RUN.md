# Como executar no Ubuntu

## Pré-requisitos

```bash
# Atualizar o sistema
sudo apt update && sudo apt upgrade -y

# Instalar Docker (necessário para KinD)
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker

# Instalar KinD
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Instalar kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl

# Verificar instalações
kind --version
kubectl version --client
docker --version
```

## Criar os arquivos de configuração

**1. Criar um diretório:**
```bash
mkdir redpanda-k8s
cd redpanda-k8s
```

**2. Criar cada arquivo:**

```bash
# kind-cluster.yaml
cat > kind-cluster.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
    - containerPort: 30000
      hostPort: 9092
      protocol: TCP
    - containerPort: 30001
      hostPort: 9644
      protocol: TCP
  - role: worker
  - role: worker
EOF
```

```bash
# namespace.yaml
cat > namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: redpanda
  labels:
    name: redpanda
EOF
```

```bash
# redpanda-config.yaml
cat > redpanda-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: redpanda-config
  namespace: redpanda
data:
  redpanda.yaml: |
    redpanda:
      data_directory: /var/lib/redpanda/data
      seed_servers:
        - host:
            address: redpanda-0.redpanda.redpanda.svc.cluster.local
            port: 33145
        - host:
            address: redpanda-1.redpanda.redpanda.svc.cluster.local
            port: 33145
        - host:
            address: redpanda-2.redpanda.redpanda.svc.cluster.local
            port: 33145
      rpc_server:
        address: 0.0.0.0
        port: 33145
      kafka_api:
        - address: 0.0.0.0
          port: 9092
          name: internal
      admin:
        - address: 0.0.0.0
          port: 9644
      advertised_kafka_api:
        - address: redpanda-0.redpanda.redpanda.svc.cluster.local
          port: 9092
          name: internal
      node_id: 0
      developer_mode: true
EOF
```

```bash
# redpanda-services.yaml
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

```bash
# redpanda-statefulset.yaml
cat > redpanda-statefulset.yaml << 'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redpanda
  namespace: redpanda
spec:
  serviceName: redpanda
  replicas: 3
  selector:
    matchLabels:
      app: redpanda
  template:
    metadata:
      labels:
        app: redpanda
    spec:
      containers:
      - name: redpanda
        image: redpandadata/redpanda:v25.2.9
        command:
        - /bin/bash
        - -c
        - |
          set -e
          NODE_ID=$(hostname | sed 's/redpanda-//')
          echo "Starting node ${NODE_ID}"
          exec /usr/bin/rpk redpanda start \
            --config /etc/redpanda/redpanda.yaml \
            --node-id ${NODE_ID} \
            --kafka-addr PLAINTEXT://0.0.0.0:9092 \
            --advertise-kafka-addr PLAINTEXT://redpanda-${NODE_ID}.redpanda.redpanda.svc.cluster.local:9092 \
            --rpc-addr 0.0.0.0:33145 \
            --advertise-rpc-addr redpanda-${NODE_ID}.redpanda.redpanda.svc.cluster.local:33145 \
            --smp 1 \
            --memory 1G \
            --reserve-memory 0M \
            --overprovisioned \
            --node-index ${NODE_ID}
        env:
        - name: REDPANDA_ENVIRONMENT
          value: "kubernetes"
        ports:
        - containerPort: 9092
          name: kafka
        - containerPort: 33145
          name: rpc
        - containerPort: 9644
          name: admin
        livenessProbe:
          httpGet:
            path: /v1/status/ready
            port: 9644
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /v1/status/ready
            port: 9644
          initialDelaySeconds: 5
          periodSeconds: 10
        volumeMounts:
        - name: config
          mountPath: /etc/redpanda
        - name: data
          mountPath: /var/lib/redpanda/data
        - name: timezone
          mountPath: /etc/localtime
      volumes:
      - name: config
        configMap:
          name: redpanda-config
      - name: timezone
        hostPath:
          path: /etc/localtime
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
EOF
```

## Executar o setup completo

```bash
# 1. Criar o cluster KinD
kind create cluster --config kind-cluster.yaml

# Aguardar um momento para o cluster inicializar
sleep 10

# 2. Criar namespace
kubectl apply -f namespace.yaml

# 3. Aplicar configurações do Redpanda
kubectl apply -f redpanda-config.yaml
kubectl apply -f redpanda-services.yaml
kubectl apply -f redpanda-statefulset.yaml

# 4. Acompanhar o deploy (pode levar 2-3 minutos)
kubectl get pods -n redpanda -w

# Quando estiverem todos "Running" e "Ready 1/1", pressione Ctrl+C
```

## Testar o Redpanda

```bash
# Ver status dos pods
kubectl get pods -n redpanda

# Ver os serviços
kubectl get svc -n redpanda

# Testar Admin API
curl http://localhost:9644/v1/status/ready

# Entrar em um pod para testar
kubectl exec -it redpanda-0 -n redpanda -- /bin/bash

# Dentro do pod, criar um tópico
rpk topic create test-topic

# Listar tópicos
rpk topic list

# Sair
exit
```

## Limpar tudo

```bash
kind delete cluster
```
