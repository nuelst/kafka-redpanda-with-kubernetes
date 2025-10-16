# kind-cluster.yaml
```yml
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
```



  # namespace.yaml
```yml
apiVersion: v1
kind: Namespace
metadata:
  name: redpanda
  labels:
    name: redpanda
```

# redpanda-config.yaml
```yml
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
      developer_mode: true
```


# redpanda-services.yaml
```yml
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

```

# redpanda-statefulset.yaml
```yml
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
```