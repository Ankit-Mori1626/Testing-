# Terraform + kubeadm + Django/React 3-Tier App

Complete flow: **Terraform (AWS)** → **1 master + 3 worker EC2 nodes** → **kubeadm cluster** → **Django backend + React frontend + Postgres database** → **Docker images on DockerHub** → **deployed to Kubernetes**.

```
.
├── terraform/          # AWS infra: VPC, SG, 1 master + 3 worker EC2 instances
├── backend/             # Django REST API
├── frontend/             # React app (talks to backend)
├── database/             # Postgres (custom Dockerfile + init.sql)
├── kubeadm-scripts/       # Bootstrap + kubeadm init/join scripts
├── k8s/                    # Kubernetes manifests (Deployments/Services/Secrets)
├── docker-compose.yml       # Local testing (no k8s needed)
└── build-and-push.sh        # Build all 3 images, push to DockerHub
```

---

## Step 1 — Provision infrastructure with Terraform

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars     # edit if you want (region, instance size...)
terraform init
terraform plan
terraform apply
```

This creates:
- 1 VPC + public subnet + internet gateway
- 1 security group (SSH, 6443, NodePort range 30000-32767, 80/443, full internal traffic)
- An auto-generated SSH key pair (saved locally as `terraform/k8s-cluster-key.pem`)
- **1 master EC2 instance** + **3 worker EC2 instances** (Ubuntu 22.04, t3.medium)
- Every instance automatically runs `kubeadm-scripts/common-setup.sh` on boot (via `user_data`), which installs containerd, kubeadm, kubelet, kubectl and disables swap — so nodes are pre-provisioned and ready.

At the end, note the outputs: `master_public_ip`, `master_private_ip`, `worker_public_ips`.

## Step 2 — Initialize the Kubernetes cluster (kubeadm)

SSH into the **master**:
```bash
ssh -i terraform/k8s-cluster-key.pem ubuntu@<master_public_ip>
```

Wait ~1-2 min for the boot script to finish (check with `cat /var/log/k8s-bootstrap.log`), then:
```bash
# copy kubeadm-scripts/master-init.sh onto the master first (scp), then:
./master-init.sh <master_private_ip>
```
This runs `kubeadm init`, configures `kubectl`, installs the Flannel CNI, and **prints a `kubeadm join` command** — copy it.

Now SSH into **each of the 3 workers** and run:
```bash
./worker-join.sh "kubeadm join 10.0.x.x:6443 --token ... --discovery-token-ca-cert-hash sha256:..."
```
(paste the exact command the master printed).

Back on the master, verify:
```bash
kubectl get nodes
# should show 1 master + 3 workers, all "Ready" after ~30s
```

## Step 3 — Build & push Docker images to DockerHub

From your local machine (or the master node, since Docker is installed there too):
```bash
chmod +x build-and-push.sh
./build-and-push.sh <your_dockerhub_username> http://<master_public_ip>:30800
```
This builds `myapp-backend`, `myapp-frontend`, `myapp-database` and pushes all three to `docker.io/<your_dockerhub_username>/...`.

> The second argument is the backend URL, baked into the React build at compile time (`REACT_APP_API_URL`), so the frontend knows where to call the API once it's exposed via NodePort 30800.

## Step 4 — Deploy to the Kubernetes cluster

Replace `DOCKERHUB_USERNAME` in the 3 image references inside `k8s/*.yaml`:
```bash
cd k8s
sed -i 's/DOCKERHUB_USERNAME/<your_dockerhub_username>/g' *.yaml
```

Apply everything (on the master, where `kubectl` is configured):
```bash
kubectl apply -f namespace.yaml
kubectl apply -f secrets.yaml
kubectl apply -f configmap.yaml
kubectl apply -f database-pvc.yaml
kubectl apply -f database-deployment.yaml
kubectl apply -f database-service.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml

kubectl get pods -n myapp -w
```

Once the Django backend pod is `Running`, run migrations once:
```bash
kubectl exec -n myapp -it deploy/backend -- python manage.py migrate
```

## Step 5 — Access the app

- **Frontend**: `http://<any_node_public_ip>:30080`
- **Backend health check**: `http://<any_node_public_ip>:30800/api/health/`

Any node's public IP works — Kubernetes NodePort routes the traffic to the right pod regardless of which node it lands on.

---

## Local testing (optional, before touching AWS at all)

```bash
docker compose up --build #
```
- Frontend → http://localhost:3000
- Backend → http://localhost:8000/api/health/
- Postgres → localhost:5432

---

## Notes / things to adjust for production

- `allowed_ssh_cidr` in `terraform.tfvars` is `0.0.0.0/0` by default — restrict it to your IP.
- `DJANGO_SECRET_KEY` and Postgres password are placeholders in `.env.example` / `k8s/secrets.yaml` — change them.
- The React app bakes `REACT_APP_API_URL` in **at build time**. If the master's public IP changes, rebuild & re-push the frontend image.
- For a real setup, put an Ingress controller (nginx-ingress) in front instead of raw NodePorts, and use `LoadBalancer`/ALB if you want a single stable domain.
- `terraform destroy` when done to avoid AWS charges.
