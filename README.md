# ☁️ Cloud-Native Voting Application on AWS EKS

A cloud-native voting application deployed on **Amazon EKS** using Kubernetes, a React frontend, a Go API, and a three-member MongoDB replica set with persistent AWS EBS storage.

> **Validated end-to-end:** browser voting → React frontend → Go API → MongoDB replica set → vote persisted.

## 🏗️ Architecture

```text
                         Internet
                            |
                            v
                 Frontend LoadBalancer
                            |
                 +----------+----------+
                 |                     |
          React Frontend        React Frontend
             Pod 1                   Pod 2
                 |
                 v
                    API LoadBalancer
                            |
                 +----------+----------+
                 |                     |
              Go API Pod 1         Go API Pod 2
                 |                     |
                 +----------+----------+
                            |
                            v
                  MongoDB Replica Set
                 +----------+----------+
                 |          |          |
              mongo-0    mongo-1    mongo-2
              PRIMARY   SECONDARY   SECONDARY
                 |          |          |
                EBS        EBS        EBS
```

## 🧰 Technology Stack

- **Amazon EKS** — managed Kubernetes control plane
- **Amazon EC2** — administration host and managed worker nodes
- **Kubernetes** — Deployments, StatefulSet, Services, Secrets, PVCs
- **React** — frontend
- **Go (Golang)** — backend API
- **MongoDB 4.2** — three-member replica set
- **Amazon EBS CSI Driver** — persistent storage
- **gp3 EBS volumes** — encrypted persistent storage
- **AWS LoadBalancer Services** — public frontend and API endpoints

## 📦 Kubernetes Components

| Component | Configuration |
|---|---|
| Namespace | `cloudchamp` |
| Frontend | 2 replicas |
| API | 2 replicas |
| MongoDB | 3 replicas |
| MongoDB storage | 3 × 1 GiB PVCs |
| StorageClass | `ebs-gp3` |
| MongoDB replica set | `rs0` |
| External access | LoadBalancer Services |
| EBS CSI | Managed EKS add-on |

## 🚀 Deployment

### 1. Create the namespace

```bash
kubectl create namespace cloudchamp
kubectl config set-context --current --namespace=cloudchamp
```

### 2. Create the encrypted gp3 StorageClass

```bash
kubectl apply -f manifests/storageclass-gp3.yaml
kubectl get storageclass
```

### 3. Create the MongoDB secret

A real database password is intentionally **not committed** to this public repository.

```bash
kubectl create secret generic mongodb-secret \
  --from-literal=username=admin \
  --from-literal=password='<CHANGE_THIS_PASSWORD>' \
  -n cloudchamp
```

### 4. Deploy MongoDB

```bash
kubectl apply -f manifests/mongo-statefulset.yaml
kubectl apply -f manifests/mongo-service.yaml
kubectl get pods
kubectl get pvc
```

### 5. Initialize the MongoDB replica set

```bash
cat <<'EOF' | kubectl exec -i mongo-0 -- mongo
rs.initiate();
sleep(2000);
rs.add("mongo-1.mongo:27017");
sleep(2000);
rs.add("mongo-2.mongo:27017");
sleep(2000);
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
sleep(5000);
EOF
```

Verify:

```bash
kubectl exec -it mongo-0 -- mongo --eval "rs.status()" | grep "PRIMARY\|SECONDARY"
```

Expected:

```text
PRIMARY
SECONDARY
SECONDARY
```

### 6. Seed the voting data

```bash
cat <<'EOF' | kubectl exec -i mongo-0 -- mongo
use langdb;

db.languages.insert({"name":"csharp","codedetail":{"usecase":"system, web, server-side","rank":5,"compiled":false,"homepage":"https://dotnet.microsoft.com/learn/csharp","download":"https://dotnet.microsoft.com/download/","votes":0}});
db.languages.insert({"name":"python","codedetail":{"usecase":"system, web, server-side","rank":3,"script":false,"homepage":"https://www.python.org/","download":"https://www.python.org/downloads/","votes":0}});
db.languages.insert({"name":"javascript","codedetail":{"usecase":"web, client-side","rank":7,"script":false,"homepage":"https://en.wikipedia.org/wiki/JavaScript","download":"n/a","votes":0}});
db.languages.insert({"name":"go","codedetail":{"usecase":"system, web, server-side","rank":12,"compiled":true,"homepage":"https://golang.org","download":"https://golang.org/dl/","votes":0}});
db.languages.insert({"name":"java","codedetail":{"usecase":"system, web, server-side","rank":1,"compiled":true,"homepage":"https://www.java.com/en/","download":"https://www.java.com/en/download/","votes":0}});
db.languages.insert({"name":"nodejs","codedetail":{"usecase":"system, web, server-side","rank":20,"script":false,"homepage":"https://nodejs.org/en/","download":"https://nodejs.org/en/download/","votes":0}});
EOF
```

### 7. Deploy the API

```bash
kubectl apply -f manifests/api-deployment.yaml
kubectl apply -f manifests/api-service.yaml

kubectl get pods
kubectl get svc api
```

When the LoadBalancer hostname is ready:

```bash
export API_ELB_PUBLIC_FQDN=$(kubectl get svc api -o jsonpath="{.status.loadBalancer.ingress[0].hostname}")

curl http://$API_ELB_PUBLIC_FQDN/ok
curl -s http://$API_ELB_PUBLIC_FQDN/languages | jq .
```

### 8. Configure and deploy the frontend

Before deployment, replace:

```yaml
- name: REACT_APP_APIHOSTPORT
  value: REPLACE_WITH_API_LOAD_BALANCER_DNS
```

with the actual API LoadBalancer hostname.

Then:

```bash
kubectl apply -f manifests/frontend-deployment.yaml
kubectl apply -f manifests/frontend-service.yaml
kubectl get pods
kubectl get svc frontend
```

Get the public URL:

```bash
export FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc frontend -o jsonpath="{.status.loadBalancer.ingress[0].hostname}")
echo "http://$FRONTEND_ELB_PUBLIC_FQDN"
```

Open that URL in a browser and vote.

## ✅ Validation

Verify the persisted vote data:

```bash
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```

Useful health checks:

```bash
kubectl get nodes
kubectl get pods
kubectl get svc
kubectl get pvc
kubectl get pv
kubectl get statefulset,deployment
kubectl get endpoints api frontend
```

## 🔐 AWS / EKS Access Model

Separate IAM responsibilities were used:

- **EKS cluster role** — EKS control plane
- **Node group role** — worker-node permissions
- **EKSAccessEC2** — EC2 administration and EKS access
- **EBS CSI role** — EBS permissions through EKS Pod Identity

The EC2 administration role was granted a **STANDARD EKS access entry** with cluster-wide Kubernetes permissions.

## 🛠️ Project Improvements

The original tutorial manifests were adapted for the current deployment:

- API and frontend Service selectors were aligned with their Deployment pod labels.
- MongoDB storage uses the `ebs.csi.aws.com` provisioner and an encrypted `gp3` StorageClass.
- The frontend API endpoint is supplied at deployment time instead of hard-coding a cluster-specific value.
- Real database credentials are excluded from the public repository.

## 🎯 Skills Demonstrated

- Amazon EKS
- IAM and EKS Access Entries
- Kubernetes Deployments
- Kubernetes StatefulSets
- MongoDB Replica Sets
- Persistent Volumes and PVCs
- Amazon EBS CSI
- Kubernetes Services / LoadBalancers
- Containerized microservices
- End-to-end troubleshooting and validation

## 📚 Original Project

Based on the public project by [N4si](https://github.com/N4si/K8s-voting-app) and the associated deployment walkthrough.

