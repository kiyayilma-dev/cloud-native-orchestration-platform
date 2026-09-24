# Cloud-Native Container Orchestration Platform

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![k3s](https://img.shields.io/badge/k3s-FFC61C?style=flat&logo=k3s&logoColor=black)
![AWS EC2](https://img.shields.io/badge/AWS_EC2-232F3E?style=flat&logo=amazonaws&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat&logo=nodedotjs&logoColor=white)

A self-hosted Kubernetes cluster deployed on AWS EC2, running a containerized Node.js API with verified self-healing, secure network exposure, and hands-on resolution of a real resource-exhaustion incident on constrained hardware.

This isn't a walkthrough of a tutorial — it's a working cluster that broke under memory pressure during setup, got diagnosed, and got fixed.

## 📐 Architecture

```
                     INTERNET
                        │
                        ▼
              Public IP : NodePort 30305
                        │
                 AWS Security Group
              (scoped inbound rules)
                        │
                        ▼
                EC2 Ubuntu Server
                   (k3s node)
                        │
                        ▼
              pipeline-app-service
                 (NodePort :30305)
                        │
                        ▼
                 pipeline-app Pod
                    (:3000)
                        │
                        ▼
                  Express API
```

Traffic reaches the cluster through a Kubernetes `NodePort` Service, exposed externally through a deliberately narrow AWS Security Group rule — not a wide-open instance.

## 🖼️ Screenshots

**Cluster node ready**
`kubectl get nodes`
![cluster nodes](docs/screenshots/kubectl-get-nodes.png)

**Pod running**
`kubectl get pods -o wide`
![pods running](docs/screenshots/kubectl-get-pods.png)

**Service exposing the app**
`kubectl get svc`
![service](docs/screenshots/kubectl-get-svc.png)

**Self-healing in action** — pod deleted, Kubernetes replacing it automatically
![self-healing](docs/screenshots/self-healing-demo.png)

**Live response from the public internet**
![browser response](docs/screenshots/browser-response.png)

**AWS Security Group — scoped inbound rules**
![security group](docs/screenshots/security-group.png)

## 🛠️ Tech Stack

| Layer              | Tool                          |
| ------------------ | ----------------------------- |
| Orchestration      | Kubernetes (k3s)              |
| Cloud Provider     | AWS EC2, VPC, Security Groups |
| Containerization   | Docker                        |
| Container Registry | Docker Hub                    |
| Application        | Node.js, Express              |

## 🧩 Engineering Challenge: Resource Exhaustion Under Load

Running k3s on a small EC2 instance (1 vCPU / 1 GB RAM) surfaced a real production-pattern failure: the control plane became unstable under memory pressure, with the API server intermittently unresponsive.

**Root cause:** no swap space configured — the kernel had no memory headroom once k3s, containerd, and the workload were all resident simultaneously.

**Fix applied:**

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
free -h
```

This gave the node the memory buffer it needed to run the control plane and workload reliably on constrained hardware — a real diagnose-and-fix cycle, not just a happy-path deployment.

## ✅ Self-Healing, Verified

Kubernetes' `ReplicaSet` controller was tested directly by force-deleting a running pod and watching it get replaced automatically:

```bash
kubectl get pods
kubectl delete pod <pod-name>
kubectl get pods -w
```

The terminated pod moves to `Terminating`, and a new one reaches `Running` within seconds — no manual intervention. See the screenshot above.

## ▶️ How to Deploy This Yourself

**1. Provision an EC2 instance and install k3s**

```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes
```

**2. Apply the manifests**

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

**3. Open the NodePort in your Security Group**

| Type       | Port  | Source       | Purpose        |
| ---------- | ----- | ------------ | -------------- |
| Custom TCP | 30305 | 0.0.0.0/0    | k3s NodePort   |
| Custom TCP | 6443  | Your IP only | Kubernetes API |
| SSH        | 22    | Your IP only | Server access  |

**4. Access the app**

```
http://<your-ec2-public-ip>:30305
```

## 📄 Kubernetes Manifests

`k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pipeline-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pipeline-app
  template:
    metadata:
      labels:
        app: pipeline-app
    spec:
      containers:
        - name: pipeline-app
          image: YOUR-DOCKERHUB-USERNAME/project1-cicd-pipeline:latest
          ports:
            - containerPort: 3000
          resources:
            requests: { cpu: "100m", memory: "64Mi" }
            limits: { cpu: "250m", memory: "128Mi" }
```

`k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pipeline-app-service
spec:
  type: NodePort
  selector:
    app: pipeline-app
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30305
```

## 🔑 Key Learnings

- **Self-healing** — verified the ReplicaSet controller automatically replacing a terminated pod to maintain desired state, not just reading about it.
- **Resource-constrained troubleshooting** — diagnosed and resolved an out-of-memory instability by provisioning swap space, rather than simply upsizing the instance.
- **Secure network exposure** — exposed only the single required NodePort to the internet, while keeping SSH and the Kubernetes API restricted to a single source IP.
- **Kubernetes fundamentals** — Deployments, Pods, Services, resource requests/limits, and NodePort networking, applied end-to-end on real infrastructure.

## 🚧 Future Improvements

- [ ] Replace manual `kubectl apply` with a GitHub Actions workflow for automated CI/CD to the cluster
- [ ] Move from a single-node k3s cluster to a multi-node setup for genuine high availability
- [ ] Add an Ingress controller with a domain and TLS instead of a raw NodePort
- [ ] Add a Horizontal Pod Autoscaler driven by real load testing
- [ ] Migrate to Amazon EKS for a fully managed control plane

## 👨‍💻 Author

**Kiya Yilma Regasa**
Cloud & DevOps Engineer

[GitHub](https://github.com/kiyayilma-dev)
