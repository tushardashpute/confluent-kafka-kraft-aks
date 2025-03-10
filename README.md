```markdown
# 🚀 Confluent Kafka KRaft Mode on Azure AKS (Test Setup)

This guide provides steps to deploy **Confluent Kafka (KRaft mode, no Zookeeper)** on **Azure Kubernetes Service (AKS)** using **Confluent Operator Kickstart method**, along with a sample **Kafka producer/consumer test** using a client pod.

---

## 📦 Prerequisites

- Azure AKS cluster up and running
- `kubectl` configured for the cluster
- `helm` installed (v3+)
- Optional: AKS storage class configured (`default` assumed here)

---

## 1️⃣ Add Confluent Helm Repo

```bash
helm repo add confluentinc https://packages.confluent.io/helm
helm repo update
```

---

## 2️⃣ Create Namespace

```bash
kubectl create namespace confluent
```

---

## 3️⃣ Deploy Confluent Kafka (KRaft Mode)

Create a file named `values-test.yaml` with minimal test setup:

```yaml
kafka:
  enabled: true
  kraft:
    enabled: true
  replicas: 1
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1
      memory: 2Gi
  persistence:
    enabled: true
    size: 10Gi
    storageClass: default
  configurationOverrides:
    "offsets.topic.replication.factor": "1"
    "transaction.state.log.replication.factor": "1"
    "transaction.state.log.min.isr": "1"
  service:
    type: LoadBalancer

schema-registry:
  enabled: true

kafka-rest:
  enabled: true

zookeeper:
  enabled: false

controlcenter:
  enabled: false
connect:
  enabled: false
ksqldb:
  enabled: false
replicator:
  enabled: false
```

Install the chart:

```bash
helm install confluent-platform confluentinc/confluent-for-kubernetes \
  -n confluent -f values-test.yaml
```

---

## 4️⃣ Check Deployment Status

```bash
kubectl get pods -n confluent
kubectl get svc -n confluent
```

Ensure `kafka`, `schema-registry`, and `kafka-rest` pods are running.

---

## 5️⃣ Kafka Testing with Sample Producer/Consumer

### 🔸 Create Kafka Client Pod

```yaml
# kafka-client.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kafka-client
  namespace: confluent
spec:
  containers:
  - name: kafka-client
    image: confluentinc/cp-kafka:7.5.0
    command: ["sleep", "infinity"]
```

Apply it:
```bash
kubectl apply -f kafka-client.yaml
```

### 🔸 Connect to Client Pod

```bash
kubectl exec -it kafka-client -n confluent -- bash
```

### 🔸 Create Topic

```bash
kafka-topics --create \
  --bootstrap-server kafka:9092 \
  --topic test-topic \
  --partitions 1 \
  --replication-factor 1
```

#### Sample Output:
```bash
Created topic test-topic.
```

### 🔸 Describe Topic

```bash
kafka-topics --describe --topic test-topic --bootstrap-server kafka:9092
```

#### Sample Output:
```bash
Topic: test-topic       TopicId: m8lTpeOPRv-VkogC1EFfCw  PartitionCount: 1       ReplicationFactor: 1    Configs: min.insync.replicas=2,segment.bytes=1073741824,message.format.version=3.0-IV1
        Topic: test-topic       Partition: 0    Leader: 2       Replicas: 2     Isr: 2
```

### 🔸 Produce Messages

```bash
kafka-console-producer --broker-list kafka:9092 --topic test-topic
```

Type sample messages:
```
Hello Kafka
Test message
```

### 🔸 Consume Messages

```bash
kafka-console-consumer --bootstrap-server kafka:9092 \
  --topic test-topic --from-beginning
```

You should see the messages you produced.

---

## ✅ Done!

You now have a working **Confluent Kafka (KRaft mode)** setup on AKS with end-to-end testing.

---

## 🔗 References

- [Confluent Operator Kickstart Guide](https://docs.confluent.io/operator/current/co-quickstart.html)
```
