# BookMyTicket on Rancher (RKE2) with Flux: Setup Runbook

This file explains, step by step, how to build everything from nothing and end with the BookMyTicket application running:

1. AWS EC2 servers and firewall
2. A 3-node RKE2 cluster called `local` that runs Rancher
3. Rancher
4. A downstream cluster called `dev-cluster` (where the apps run)
5. Flux, connected to ONE Git repository
6. A Rancher catalog and the BookMyTicket Helm chart, both stored in that same repository
7. BookMyTicket (app + MySQL + Redis, MongoDB optional) deployed by Flux

Every file is given in full. Copy the files exactly. Spaces at the start of a line matter in YAML.

---

## 1. The big picture

```
GitHub repo: platform-gitops  (one repo, branch main)
   |
   |  Flux (inside dev-cluster) checks the repo about every minute
   v
+------------------------------+        +--------------------------------------+
| local cluster (3 EC2 nodes)  |        | dev-cluster (1 EC2 node)             |
|  rke2-server (control plane) | manages|  Flux + BookMyTicket + MySQL + Redis |
|  rke2-agent-1 (worker)       |------->|  (+ MongoDB, optional)               |
|  rke2-agent-2 (worker)       |        +--------------------------------------+
|  runs ONLY Rancher           |
+------------------------------+
```

- `local` runs Rancher only. Never deploy your apps there.
- `dev-cluster` is the downstream cluster. Flux and all your apps run here.
- Flux reads the Git repo. To change what runs, you change files in Git (the only exceptions are the ConfigMaps and Secrets described in section 10).

### Final repository structure

```
platform-gitops/
├── README.md
├── clusters/
│   └── dev-cluster/                  <- Flux watches this folder (the bootstrap --path)
│       ├── flux-system/              <- created by "flux bootstrap". Never edit.
│       ├── platform-catalog.yaml     <- registers this repo as a Rancher catalog
│       └── bookmyticket.yaml         <- tells Flux to read projects/bookmyticket/dev
├── catalogs/                         <- Helm charts (what Rancher lists under Apps > Charts)
│   └── bookmyticket/
│       └── 0.1.0/
│           ├── Chart.yaml
│           ├── values.yaml
│           └── templates/
│               ├── app-deployment.yaml
│               ├── app-service.yaml
│               ├── mysql-service.yaml
│               ├── mysql-statefulset.yaml
│               ├── redis-service.yaml
│               ├── redis-deployment.yaml
│               ├── mongo-service.yaml        (optional)
│               └── mongo-statefulset.yaml    (optional)
└── projects/                         <- what is deployed, per project and environment
    └── bookmyticket/
        └── dev/
            ├── namespace.yaml
            ├── helmrelease.yaml
            ├── configmap.yaml        <- holds the image name
            └── app-config.yaml       <- holds app settings (key=value)
```

Rules:
- `catalogs/` and `projects/` sit at the TOP level, next to `clusters/`. Never put them inside `clusters/`.
- Only Helm charts go in `catalogs/`. Rancher scans the repo for `Chart.yaml` files.
- The folder name `flux-system` is exact, and Flux creates it. You never make it by hand.

---

## 2. Values you need (fill this table in as you go)

| Placeholder | What it is | Value in the first build (for reference only) |
|---|---|---|
| `<RANCHER_EIP>` | Elastic IP of `rke2-server` | 15.252.167.72 |
| `<SERVER_PRIVATE_IP>` | Private IP of `rke2-server` | 172.31.45.126 |
| `<TOKEN>` | RKE2 join token (from the server) | (secret, never share) |
| `<DOWNSTREAM_EIP>` | Elastic IP of `downstream-node` | 13.235.145.239 |
| `<DOWNSTREAM_PRIVATE_IP>` | Private IP of `downstream-node` | 172.31.42.24 |
| `<GITHUB_USER>` | Your GitHub username | kusuma123-19 |
| `<GITHUB_TOKEN>` | GitHub classic token with the `repo` scope | (secret, never share) |
| `<YOUR_GMAIL_ADDRESS>` | Gmail address the app uses for mail | (yours) |

Versions used in the first build: Ubuntu 22.04, RKE2 v1.36.5+rke2r1, Rancher v2.15.2 (chart from the `rancher-stable` repo), Flux v2.x (controllers helm v1.6.5, kustomize v1.9.6), local-path-provisioner v0.0.31.

Never write passwords or tokens into Git or into chats. Secrets are created with `kubectl` (section 8).

---

## 3. AWS: firewall and servers

All steps are in the AWS Console, region **Asia Pacific (Mumbai) ap-south-1**.

### 3.1 Security group (create it first)

EC2 > Network & Security > Security Groups > **Create security group**
- Name: `rke2-sg`, Description: `rke2 cluster`, VPC: default

Inbound rules:

| Type | Port | Source |
|---|---|---|
| SSH | 22 | My IP |
| HTTP | 80 | Anywhere-IPv4 |
| HTTPS | 443 | Anywhere-IPv4 |
| Custom TCP | 6443 | My IP |
| Custom TCP | 30000-32767 | My IP |
| All traffic | all | the same security group `rke2-sg` (self-reference) |

How to add the last rule: after creating the group, open it, **Edit inbound rules > Add rule**, Type `All traffic`, Source `Custom`, search `rke2-sg`, choose the group itself, **Save rules**. This lets the servers talk to each other.

Notes:
- The `30000-32767` rule is how you open apps (NodePort) from your laptop. Your home IP can change. If the app or Studio 3T stops opening, set this rule's source to **My IP** again.
- The EC2 browser terminal ("EC2 Instance Connect") works with the SSH rule. If Connect fails, temporarily set SSH to Anywhere or add the Instance Connect range for ap-south-1 (`13.233.177.0/29`).

### 3.2 Instances (4 servers)

For each: **Ubuntu Server 22.04 LTS (64-bit x86)**, key pair `rke2-key` (.pem, create once), security group `rke2-sg`, same VPC, auto-assign public IP enabled.

| Name | Cluster | Size | Disk |
|---|---|---|---|
| `rke2-server` | local | 2 vCPU, 8 GB (for example t3.large or m7i-flex.large) | 40 GiB gp3 |
| `rke2-agent-1` | local | 2 vCPU, 4 GB or more (t3.medium or larger) | 30 GiB gp3 |
| `rke2-agent-2` | local | same as agent-1 | 30 GiB gp3 |
| `downstream-node` | dev-cluster | 2 vCPU, 8 GB works but is tight. 4 vCPU is better. | 40 GiB gp3 |

Tip: the two agents can be launched together with **Number of instances = 2**, then renamed.

### 3.3 Elastic IPs

EC2 > Elastic IPs > **Allocate Elastic IP address**, then **Actions > Associate Elastic IP address**.
- Associate one with `rke2-server` (this is `<RANCHER_EIP>`)
- Associate one with `downstream-node` (this is `<DOWNSTREAM_EIP>`)
- Release any unused Elastic IPs (they cost money when not attached).

### 3.4 Connecting

Tick the instance > **Connect > EC2 Instance Connect > Connect**. Use one browser tab per server. To paste in that terminal, use Ctrl+Shift+V.

On every server, first run:
```bash
sudo -i
apt update && apt upgrade -y
```
If a purple screen asks about restarting services, press Enter. Do NOT reboot during the setup.

Find each server's private IP (the first number):
```bash
hostname -I
```

---

## 4. The `local` cluster with RKE2 (runs Rancher)

### 4.1 `rke2-server`

```bash
mkdir -p /etc/rancher/rke2
cat > /etc/rancher/rke2/config.yaml <<EOF
tls-san:
  - <RANCHER_EIP>
  - <SERVER_PRIVATE_IP>
write-kubeconfig-mode: "0644"
EOF

curl -sfL https://get.rke2.io | sh -
systemctl enable --now rke2-server
```
This takes 3 to 5 minutes and the command may print nothing for a while. To pin the exact version used in the first build, install with:
`curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.36.5+rke2r1 sh -`

Set up kubectl:
```bash
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc
echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml' >> ~/.bashrc
source ~/.bashrc
kubectl get nodes
kubectl get pods -A
```
The node must be `Ready`. All pods must become `Running` or `Completed` (wait 2 to 3 minutes).

Get the join token for the agents:
```bash
cat /var/lib/rancher/rke2/server/node-token
```
It starts with `K10...`. Keep it secret.

### 4.2 `rke2-agent-1` and `rke2-agent-2` (repeat on each)

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

mkdir -p /etc/rancher/rke2
cat > /etc/rancher/rke2/config.yaml <<EOF
server: https://<SERVER_PRIVATE_IP>:9345
token: <TOKEN>
EOF

systemctl enable --now rke2-agent
```
Check on `rke2-server` after 1 to 3 minutes:
```bash
kubectl get nodes
```
You need 3 nodes, all `Ready`. If an agent does not join, run `journalctl -u rke2-agent --no-pager | tail -30` on that agent. The usual causes are a wrong server IP or token, or the self-reference rule missing in `rke2-sg`.

---

## 5. Rancher on the `local` cluster

Run on `rke2-server`.

### 5.1 Helm and cert-manager
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

kubectl get pods -n cert-manager
```
Wait until the 3 cert-manager pods are `1/1 Running`.

### 5.2 Rancher
```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

# Check the Rancher version and which Kubernetes versions it accepts
helm show chart rancher-stable/rancher | grep -E "^(version|appVersion|kubeVersion)"
```
In the first build this printed `appVersion: v2.15.2` and `kubeVersion: < 1.37.0-0`, which accepts Kubernetes 1.36. If the RKE2 version is newer than `kubeVersion` allows, reinstall RKE2 with a supported version before continuing.

```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.<RANCHER_EIP>.sslip.io \
  --set bootstrapPassword=ChangeMe12345 \
  --set replicas=1

kubectl -n cattle-system rollout status deploy/rancher
kubectl get pods -n cattle-system
```
Do NOT press Ctrl+C during the rollout; it takes 3 to 8 minutes.

### 5.3 First login

Open `https://rancher.<RANCHER_EIP>.sslip.io` in a browser.
1. Click **Advanced > Proceed** on the certificate warning (Rancher uses a self-signed certificate).
2. Log in with the bootstrap password `ChangeMe12345`.
3. Choose **Set a specific password to use** and enter your own password (12+ characters). Write it down.
4. Accept the license and click **Continue**. Leave the Server URL as it is and click **Save URL**.
5. The Home page shows the `local` cluster with 3 nodes.

If the page says `no available server` or shows a 502, Rancher is still starting. Wait 5 to 10 minutes and refresh.

---

## 6. The downstream cluster `dev-cluster`

### 6.1 Create it in Rancher
1. Home > **Create** > **Custom**.
2. Cluster Name: `dev-cluster` (this name is used later in the Flux path).
3. Leave the Kubernetes version as the default, then click **Create**.
4. On the **Registration** tab:
   - Tick **etcd**, **Control Plane** and **Worker**.
   - Click **Show Advanced**: set **Node Public IP** = `<DOWNSTREAM_EIP>` and **Node Internal IP** = `<DOWNSTREAM_PRIVATE_IP>`.
   - Tick **Skip TLS verification** if you see it.
   - Click the copy icon on the command.

### 6.2 Run the command on `downstream-node`
Open the browser terminal for `downstream-node`, run `sudo -i`, and paste the command.

Important: the command must start with `curl --insecure -fL ...`. If your copy starts with plain `curl -fL`, add `--insecure` after `curl`, otherwise you get "SSL certificate ... unable to get local issuer certificate". The shape is:
```bash
curl --insecure -fL https://rancher.<RANCHER_EIP>.sslip.io/system-agent-install.sh | sudo sh -s - \
  --server https://rancher.<RANCHER_EIP>.sslip.io \
  --label 'cattle.io/os=linux' \
  --token <REGISTRATION_TOKEN_FROM_RANCHER> \
  --ca-checksum <CHECKSUM_FROM_RANCHER> \
  --etcd --controlplane --worker \
  --address <DOWNSTREAM_EIP> \
  --internal-address <DOWNSTREAM_PRIVATE_IP>
```
Always copy the real command (with its token and checksum) from the Rancher screen. Do not type it by hand, and do not share it.

### 6.3 Wait
Cluster Management shows `dev-cluster` as `Provisioning`, then **Active**. This takes 5 to 15 minutes. The Provisioning Log tab shows the progress (it waits on `calico` for a while, which is normal).

### 6.4 kubectl on `downstream-node`
```bash
sudo -i
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc
echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml' >> ~/.bashrc
source ~/.bashrc
kubectl get nodes
```
One node, `Ready`. From now on, every command in sections 7 to 9 runs on `downstream-node`, unless stated otherwise.

---

## 7. Git repository and Flux

### 7.1 GitHub repository and token
1. GitHub > **New repository** > name `platform-gitops`, private, and do NOT add a README, .gitignore or license (it must be empty).
2. GitHub > Settings > Developer settings > Personal access tokens > **Tokens (classic)** > Generate new token (classic). Scope: **repo**. Expiration: 90 days or more. Copy the token immediately.

### 7.2 Install the Flux CLI and bootstrap (on `downstream-node`)

```bash
curl -s https://fluxcd.io/install.sh | bash
flux --version
flux check --pre
```
`flux check --pre` must end with "prerequisites checks passed".

Bootstrap (this is the same format used in the company project: git URL, username, password, token auth, path, branch):
```bash
read -s -p "GitHub token: " GIT_TOKEN; echo

flux bootstrap git \
  --url=https://github.com/<GITHUB_USER>/platform-gitops \
  --username='<GITHUB_USER>' \
  --password="$GIT_TOKEN" \
  --token-auth=true \
  --path=clusters/dev-cluster \
  --branch=main
```
Notes:
- Paste the token when prompted (nothing is shown on screen).
- At work the branch may be `master`; just change `--branch`.
- The optional flag `--registry=<your registry>` makes Flux pull its images from your own registry. It is not needed here.
- The command creates `clusters/dev-cluster/flux-system/` in the repo and installs the 4 controllers. It ends with "all components are healthy".

Verify:
```bash
flux check
kubectl get pods -n flux-system
flux get kustomizations
```
4 controllers `Running`, and `flux-system` with READY `True`.

### 7.3 Get the repo onto your laptop

On Windows, in VS Code's terminal (or PowerShell):
```powershell
git --version
git config --global user.name "<GITHUB_USER>"
git config --global user.email "your-email@example.com"
cd $HOME
git clone https://github.com/<GITHUB_USER>/platform-gitops.git
cd platform-gitops
git pull
code .
```
A browser window may ask you to sign in to GitHub the first time. The `clusters/dev-cluster/flux-system` folder is already there. You create the other files in section 9.

---

## 8. Prepare the cluster (storage and secrets)

Run on `downstream-node`, as root.

### 8.1 Storage for MySQL (and MongoDB)
An RKE2 cluster created by Rancher has no storage provisioner. Without one, MySQL stays `Pending` forever.
```bash
kubectl get storageclass

kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.31/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl get storageclass
kubectl get pods -n local-path-storage
```
You want `local-path (default)` and one `Running` pod. (Data is stored on the node's disk, which is fine for practice. If the node is lost, the data is lost.)

### 8.2 Login for the catalog (so Rancher can read the private repo)
```bash
read -s -p "GitHub token: " GIT_TOKEN; echo
kubectl create secret generic catalog-git-auth -n cattle-system \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=<GITHUB_USER> \
  --from-literal=password="$GIT_TOKEN"
```
If the answer is `AlreadyExists`, it is already there. (To replace it later, delete it first with `kubectl delete secret catalog-git-auth -n cattle-system`.)

### 8.3 Namespace and application secret
The secret must exist in the namespace before the app can start. Choose your own passwords (letters and digits are safest).
```bash
kubectl create namespace bookmyticket-dev

read -s -p "DB password: " DBPASS; echo
read -s -p "Gmail app password (type dummy to skip mail): " MAILPASS; echo
read -s -p "Admin password: " ADMINPASS; echo

kubectl create secret generic app-secret -n bookmyticket-dev \
  --from-literal=db-url='jdbc:mysql://mysql-service:3306/book-my-ticket?createDatabaseIfNotExist=true' \
  --from-literal=db-username='root' \
  --from-literal=db-password="$DBPASS" \
  --from-literal=redis-url='redis://redis-service:6379' \
  --from-literal=mail-username='<YOUR_GMAIL_ADDRESS>' \
  --from-literal=mail-password="$MAILPASS" \
  --from-literal=admin-password="$ADMINPASS" \
  --from-literal=cloudinary-url='cloudinary://api_key:api_secret@cloud_name'
```
What each key is for:

| Key | Meaning |
|---|---|
| `db-url`, `db-username`, `db-password` | How the app connects to MySQL. `mysql-service` is the in-cluster name of MySQL. The same DB password is also MySQL's root password. |
| `redis-url` | How the app connects to Redis |
| `mail-username`, `mail-password` | Gmail address and a Gmail **app password** (Google Account > Security > App passwords) |
| `admin-password` | Password of the app's admin login (the admin email comes from `app-config`) |
| `cloudinary-url` | Placeholder value. Image uploads work only with real Cloudinary credentials |

Important:
- MySQL saves its root password on its disk the first time it starts. If you later change `db-password` in the secret, MySQL and the app disagree. Keep it the same, or delete the MySQL disk (`kubectl delete pvc mysql-data-mysql-0 -n bookmyticket-dev` after scaling the StatefulSet to 0) to start clean.
- Never put real passwords in the application repo or in Git.

### 8.4 Optional: MongoDB secret (only if you deploy MongoDB)
```bash
read -s -p "Mongo password: " MPASS; echo
kubectl create secret generic mongo-secret -n bookmyticket-dev \
  --from-literal=username=admin \
  --from-literal=password="$MPASS"
```
MongoDB runs in the cluster but the application does NOT use it (the app is written for MySQL). It is only for learning. Skip this and delete the two `mongo-*` template files if you do not want it.

---

## 9. The repository files (create all of them in VS Code)

For each file: in VS Code Explorer, hover over `platform-gitops`, click **New File**, type the full path (slashes create the folders), paste the content, press Ctrl+S.

Replace `<GITHUB_USER>` where it appears.

### 9.1 Flux entry files: `clusters/dev-cluster/`

**`clusters/dev-cluster/platform-catalog.yaml`** (registers this repo as a Rancher catalog; Flux applies it)
```yaml
apiVersion: catalog.cattle.io/v1
kind: ClusterRepo
metadata:
  name: platform-catalog
spec:
  gitRepo: https://github.com/<GITHUB_USER>/platform-gitops
  gitBranch: main
  clientSecret:
    name: catalog-git-auth
    namespace: cattle-system
```

**`clusters/dev-cluster/bookmyticket.yaml`** (tells Flux to read the project folder)
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: bookmyticket-dev
  namespace: flux-system
spec:
  interval: 1m
  path: ./projects/bookmyticket/dev
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

### 9.2 The Helm chart: `catalogs/bookmyticket/0.1.0/`

**`catalogs/bookmyticket/0.1.0/Chart.yaml`**
```yaml
apiVersion: v2
name: bookmyticket
description: BookMyTicket app with MySQL and Redis
type: application
version: 0.1.0
appVersion: "latest"
```

**`catalogs/bookmyticket/0.1.0/values.yaml`**
```yaml
replicaCount: 1
image: kusumacr21/book-my-ticket:latest
service:
  nodePort: 30081
mysql:
  storage: 5Gi
mongo:
  image: mongo:7.0
  storage: 2Gi
  nodePort: 30017
```
(The `mongo:` block is read by the optional MongoDB templates. Keep it even if you skip MongoDB.)

**`catalogs/bookmyticket/0.1.0/templates/app-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: book-my-ticket-app
  labels:
    app: book-my-ticket
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  selector:
    matchLabels:
      app: book-my-ticket
  template:
    metadata:
      labels:
        app: book-my-ticket
    spec:
      containers:
        - name: book-my-ticket
          image: "{{ .Values.image }}"
          ports:
            - containerPort: 8085
          envFrom:
            - configMapRef:
                name: app-config
          env:
            - name: DB_URL
              valueFrom:
                secretKeyRef: { name: app-secret, key: db-url }
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef: { name: app-secret, key: db-username }
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef: { name: app-secret, key: db-password }
            - name: REDIS_URL
              valueFrom:
                secretKeyRef: { name: app-secret, key: redis-url }
            - name: MAIL_USERNAME
              valueFrom:
                secretKeyRef: { name: app-secret, key: mail-username }
            - name: MAIL_PASSWORD
              valueFrom:
                secretKeyRef: { name: app-secret, key: mail-password }
            - name: ADMIN_PASSWORD
              valueFrom:
                secretKeyRef: { name: app-secret, key: admin-password }
            - name: CLOUDINARY_URL
              valueFrom:
                secretKeyRef: { name: app-secret, key: cloudinary-url }
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```
Why `maxSurge: 0`: on a small node there is no spare CPU to start a new pod before the old one stops. With this setting the old pod stops first, so updates do not get stuck in `Pending` (the app is unavailable for 1 to 2 minutes during an update).

**`catalogs/bookmyticket/0.1.0/templates/app-service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: book-my-ticket-service
spec:
  type: NodePort
  selector:
    app: book-my-ticket
  ports:
    - port: 8085
      targetPort: 8085
      nodePort: {{ .Values.service.nodePort }}
```

**`catalogs/bookmyticket/0.1.0/templates/mysql-service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

**`catalogs/bookmyticket/0.1.0/templates/mysql-statefulset.yaml`**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-service
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef: { name: app-secret, key: db-password }
            - name: MYSQL_DATABASE
              value: book-my-ticket
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: {{ .Values.mysql.storage }}
```

**`catalogs/bookmyticket/0.1.0/templates/redis-service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
```

**`catalogs/bookmyticket/0.1.0/templates/redis-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
```

**Optional MongoDB: `catalogs/bookmyticket/0.1.0/templates/mongo-service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  type: NodePort
  selector:
    app: mongo
  ports:
    - port: 27017
      targetPort: 27017
      nodePort: {{ .Values.mongo.nodePort }}
```

**Optional MongoDB: `catalogs/bookmyticket/0.1.0/templates/mongo-statefulset.yaml`**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: mongo-service
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongo
          image: "{{ .Values.mongo.image }}"
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef: { name: mongo-secret, key: username }
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef: { name: mongo-secret, key: password }
          resources:
            requests:
              cpu: "50m"
              memory: "256Mi"
            limits:
              memory: "512Mi"
          volumeMounts:
            - name: mongo-data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongo-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: {{ .Values.mongo.storage }}
```
VS Code shows red warnings on the files with `{{ }}`. Ignore them; it reads them as plain YAML.

### 9.3 The deployment: `projects/bookmyticket/dev/`

**`projects/bookmyticket/dev/namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bookmyticket-dev
```

**`projects/bookmyticket/dev/configmap.yaml`** (the image the app runs; one single line)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: bookmyticket-values
  namespace: bookmyticket-dev
  annotations:
    kustomize.toolkit.fluxcd.io/ssa: IfNotPresent
data:
  values.yaml: |
    image: kusumacr21/book-my-ticket:latest
```

**`projects/bookmyticket/dev/app-config.yaml`** (app settings as key=value; every key becomes an environment variable)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: bookmyticket-dev
  annotations:
    kustomize.toolkit.fluxcd.io/ssa: IfNotPresent
data:
  ADMIN_EMAIL: admin@gmail.com
```

About the annotation `kustomize.toolkit.fluxcd.io/ssa: IfNotPresent`: Flux creates the ConfigMap from Git once and never overwrites it afterwards. After that, you change its values in Rancher (Storage > ConfigMaps) and Git edits are ignored. Remove the annotation if you prefer Git to be the only source of truth (then edit the files in Git and push).

**`projects/bookmyticket/dev/helmrelease.yaml`**
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: bookmyticket
  namespace: bookmyticket-dev
spec:
  interval: 5m
  chart:
    spec:
      chart: ./catalogs/bookmyticket/0.1.0
      reconcileStrategy: Revision
      sourceRef:
        kind: GitRepository
        name: flux-system
        namespace: flux-system
  valuesFrom:
    - kind: ConfigMap
      name: bookmyticket-values
```
`valuesFrom` makes Helm read the image from the ConfigMap. `reconcileStrategy: Revision` makes Flux rebuild the chart on every Git change. Without it, Flux ignores changes to chart files unless the chart version number changes.

### 9.4 Check the folder layout before pushing

In VS Code's Explorer you must see exactly the structure from section 1. In particular `projects` must be at the top level (not inside `clusters`). In the terminal:
```powershell
Get-ChildItem -Recurse -File | Where-Object { $_.FullName -notmatch "\\.git\\" } | Select-Object -ExpandProperty FullName
```

### 9.5 Push (this starts the deployment)
```powershell
git add -A
git commit -m "bookmyticket chart, catalog and deployment"
git push
```
The message "This repository moved" with a capitalised username is harmless.

---

## 10. Verify the deployment

Flux checks Git about every minute and deploys by itself. Wait 2 to 5 minutes after the push (you do not have to run anything). To speed it up, on `downstream-node`:
```bash
flux reconcile source git flux-system
flux reconcile kustomization flux-system
flux reconcile helmrelease bookmyticket -n bookmyticket-dev --timeout=10m
```
Do NOT press Ctrl+C while a reconcile is waiting. It can take several minutes. "context deadline exceeded" only means it stopped waiting; check the real state with the commands below.

```bash
flux get kustomizations
flux get helmreleases -A
kubectl get pods -n bookmyticket-dev
kubectl get svc -n bookmyticket-dev
kubectl get pvc -n bookmyticket-dev
kubectl get configmap -n bookmyticket-dev
kubectl get clusterrepo
```
Expected result:
- `flux-system` and `bookmyticket-dev` kustomizations: READY `True`
- HelmRelease `bookmyticket`: READY `True` ("Helm install succeeded")
- Pods `book-my-ticket-app-...`, `mysql-0`, `redis-...` (and `mongo-0` if used): `Running`. The app pod may restart once or twice while MySQL starts, which is normal.
- Service `book-my-ticket-service` showing `8085:30081/TCP`
- PVC `mysql-data-mysql-0`: `Bound`
- ConfigMaps `bookmyticket-values` and `app-config` exist
- `platform-catalog` listed by `kubectl get clusterrepo`

In Rancher (Explore `dev-cluster`):
- **Apps > Repositories**: `platform-catalog` is **Active**
- **Apps > Charts**: search `bookmyticket` and see version 0.1.0
- **Workloads > Pods** (namespace `bookmyticket-dev`): the pods are `Running`

Open the application in a browser:
```
http://<DOWNSTREAM_EIP>:30081
```
It is `http`, not `https`. Log in as admin with email `admin@gmail.com` and the admin password you chose in section 8.3.

---

## 11. Stopping and starting the servers (to save cost)

Stop (never Terminate; Terminate deletes everything): EC2 > Instances > tick all four > **Instance state > Stop**. Elastic IPs stay yours (keep them, they hold Rancher's address).

Start in this order:
1. `rke2-server`, wait about 3 minutes
2. `rke2-agent-1` and `rke2-agent-2`, and `downstream-node`
3. Wait 5 to 10 minutes. Rancher shows `no available server` until its pod is ready. Check on `rke2-server` with `kubectl get nodes` and `kubectl get pods -n cattle-system`.
4. `dev-cluster` returns to Active by itself. If the app pod stays `Pending`, see the table below.

---

## 12. Common problems and fixes

| Problem | Cause | Fix |
|---|---|---|
| Rancher page says `no available server` or 502 | Rancher is still starting after a start or install | Wait 5 to 10 minutes. Check `kubectl get pods -n cattle-system` |
| `curl: (60) SSL certificate ... unable to get local issuer certificate` when registering the node | Rancher's certificate is self-signed | Add `--insecure` after `curl` in the registration command |
| `dev-cluster` shows Machines (0) | The registration command was not run on the node | Run it on `downstream-node` as root |
| `flux get sources git` says `authentication required: Invalid username or token` | The GitHub token expired or was deleted | Create a new token with `repo` scope, then on `downstream-node`: `read -s -p "Token: " GIT_TOKEN; echo`, `kubectl delete secret flux-system -n flux-system`, `flux create secret git flux-system --namespace=flux-system --url=https://github.com/<GITHUB_USER>/platform-gitops --username=<GITHUB_USER> --password="$GIT_TOKEN"`. Also replace `catalog-git-auth` (delete it and recreate it as in 8.2) |
| `kubectl get clusterrepo` does not list `platform-catalog` | Flux has not applied the file (wrong folder, not pushed, or Flux cannot read Git) | The file must be directly in `clusters/dev-cluster/`. Run `flux get sources git` and read the message |
| Catalog `platform-catalog` is not Active in Rancher | Wrong token in `catalog-git-auth`, or the repo URL is wrong | Recreate the secret (8.2) and check the URL in the file |
| HelmRelease error `can't evaluate field repository in type interface {}` | A template still uses `.Values.image.repository` while `values.yaml` has `image:` as one line | Make the image line in `app-deployment.yaml` exactly `image: "{{ .Values.image }}"` |
| HelmRelease error `nil pointer evaluating interface {}.image` (mongo) | The `mongo:` block is missing from `values.yaml` | Add the `mongo:` block from 9.2 |
| HelmRelease error `MalformedYAMLError ... mapping values are not allowed` | An indentation mistake in a template | Replace the whole file with the version in this README |
| HelmRelease cannot find ConfigMap `bookmyticket-values` | Flux applies the ConfigMap and the HelmRelease at the same time | Wait a minute and run `flux reconcile helmrelease bookmyticket -n bookmyticket-dev --timeout=10m` |
| Flux keeps using an old copy of the chart | `reconcileStrategy: Revision` is missing | Add it to `helmrelease.yaml` (9.3) and push |
| App pod `CreateContainerConfigError` | `app-secret` or `app-config` is missing a key or does not exist | Check `kubectl get secret,configmap -n bookmyticket-dev` and create what is missing |
| App pod `CrashLoopBackOff` | The app starts and crashes (often MySQL not ready yet, or a wrong DB setting) | Read `kubectl logs deploy/book-my-ticket-app -n bookmyticket-dev --previous --tail=40`. If MySQL was just starting, it fixes itself after a restart or two |
| `mysql-0` is `Pending` | No storage provisioner | Do 8.1 |
| App or Mongo pod `Pending` with `Insufficient cpu` | The single node has no spare CPU | `kubectl scale deploy book-my-ticket-app -n bookmyticket-dev --replicas=0`, wait until it is gone, then `--replicas=1`. A bigger node (4 vCPU) fixes it for good |
| `ImagePullBackOff` | The image name or tag does not exist on Docker Hub | Check the exact name and tag on Docker Hub (the repository must be public) |
| App does not open in the browser | Your IP changed, so the `30000-32767` rule blocks you | EC2 > Security Groups > `rke2-sg` > set that rule's source to **My IP** |
| App opens but login fails | Wrong admin password or secret values | Check `app-secret` (Storage > Secrets in Rancher) |
| MySQL password mismatch after changing `db-password` | MySQL saved the first password on its disk | See the important note in 8.3 |

---

## 13. Quick checklist (to repeat the setup)

- [ ] Security group `rke2-sg` with the self-reference rule
- [ ] 4 instances (server, 2 agents, downstream node) and 2 Elastic IPs
- [ ] RKE2 server, 2 agents joined, 3 nodes `Ready`
- [ ] cert-manager and Rancher installed, logged in
- [ ] `dev-cluster` created with Custom, node registered, cluster `Active`
- [ ] Empty GitHub repo `platform-gitops` and a token with `repo` scope
- [ ] Flux bootstrapped on `downstream-node` with `--path=clusters/dev-cluster`
- [ ] Storage provisioner installed (`local-path (default)`)
- [ ] Secrets created: `catalog-git-auth` (cattle-system), `app-secret` (bookmyticket-dev), optional `mongo-secret`
- [ ] All repo files from section 9 created and pushed
- [ ] HelmRelease `True`, pods `Running`, catalog `Active`
- [ ] App opens at `http://<DOWNSTREAM_EIP>:30081`
