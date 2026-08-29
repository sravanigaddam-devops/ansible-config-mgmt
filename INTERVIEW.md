# DevOps Portfolio — Interview Documentation
**Candidate:** Sravani Gaddam | **Stack:** AWS · Terraform · Ansible · Docker · Kubernetes · Jenkins  
**GitHub:** https://github.com/sravanigaddam-devops | **Docker Hub:** sravani81299 | **Region:** us-east-1 | **Date:** 2026-08-29

> **Elevator Pitch (30s):** “I built 3 linked DevOps projects: (1) Terraform-provisioned AWS VPC with S3 remote state, (2) Ansible that hardens and configures the EC2s from one control node, and (3) a gated CI/CD pipeline that tests a Python Flask app, builds versioned Docker images to Docker Hub, and rolling-deploys to Kubernetes. All 3 are live-verified — Terraform EC2 `3.231.160.92` served Ansible’s nginx, and `kind` K8s served `sravani81299/sravani-app:1.0.1` with zero-downtime.”

---

## Table of Contents
1. [At-a-Glance](#at-a-glance)
2. [Architecture](#architecture)
3. [Project 2 — Terraform (Foundation)](#project-2--terraform-aws-infra)
4. [Project 3 — Ansible (Config Mgmt)](#project-3--ansible-config-mgmt)
5. [Project 1 — CI/CD (Docker & K8s)](#project-1--cicd-docker--k8s)
6. [End-to-End Proof (Outputs)](#end-to-end-proof)
7. [Troubleshooting — What I Fixed](#troubleshooting)
8. [How to Explain in Interview](#how-to-explain-in-interview)
9. [Commands Cheat Sheet](#commands-cheat-sheet)
10. [Repos & Cleanup](#repos--cleanup)

---

## At-a-Glance

| # | Project | What It Proves | Key Tech | Live Output | Repo |
|---|---------|----------------|----------|-------------|------|
| 2 | **Terraform AWS Infra** | IaC, modules, remote state, safe plan/apply | Terraform 1.15.8, AWS VPC, S3 `use_lockfile` | `vpc-03b31ff28e41bc125`, `i-00f507eb0af08bb0e` `3.231.160.92` | [terraform-aws-infra](https://github.com/sravanigaddam-devops/terraform-aws-infra) |
| 3 | **Ansible Config Mgmt** | Idempotent config, roles, single control → many nodes | Ansible 2.21.3, Amazon Linux 2023, nginx, firewalld | `http://3.231.160.92/` `Configured by Ansible - web-1` + `/health` `ok` | [ansible-config-mgmt](https://github.com/sravanigaddam-devops/ansible-config-mgmt) |
| 1 | **CI/CD Pipeline** | Gated pipeline, versioning, rolling update, service exposure | Python Flask 3.1, Docker 29.7.2, kind K8s v1.37.0, Jenkins, Docker Hub | `sravani81299/sravani-app:1.0.1` 3/3 Running, `http://localhost:8080/health` | [cicd-python-app-k8s](https://github.com/sravanigaddam-devops/cicd-python-app-k8s) |

**Dependencies:** `2 → 3 → 1` — Terraform creates the network, Ansible configures the hosts, CI/CD deploys the app. Can be explained in that order in interview.

---

## Architecture

![Architecture](architecture.png)
*Generated 1400x900 PNG via `generate_diagram.py:1` — also as `architecture.drawio` editable at https://app.diagrams.net. Copies in each repo `architecture.png` + `screenshots/architecture.png`.*

**Flow:**
- **Infra:** `terraform apply` → VPC `10.0.0.0/16` (2 public `10.0.1/2.0/24` + 2 private `10.0.10/20.0/24` in `us-east-1a/b`) → IGW `igw-*` + NAT `nat-*` + RTs → SG `sg-*` (22,80,443,8080) → EC2 `t3.micro` `3.231.160.92` nginx `user_data` → S3 `sravani-terraform-demo-bucket1/terraform.tfstate` with `use_lockfile`
- **Config:** `ansible-playbook playbooks/site.yml:13` (roles: `hardening` → `webserver`) from WSL control → `inventory/hosts.ini:1` `3.231.160.92` → dnf-automatic + sshd harden + nginx `/health` + firewalld `http/https`
- **App:** `GitHub push` → Jenkins webhook → `app/test_app.py:1` pytest gate → `Dockerfile:1` `python:3.11-slim` `gunicorn` → `sravani81299/sravani-app:${BUILD} + latest` → `k8s/deployment.yaml:17` `3 replicas RollingUpdate maxSurge1 maxUnavailable1` → `k8s/service.yaml:1` `LoadBalancer 80→5000` → `kubectl port-forward` → `ok`

---

## Project 2 — Terraform AWS Infra

### Why
Repeatable, versioned infra. No click-ops. `plan` before `apply` prevents drift. Remote state in S3 lets team share lock.

### Structure
```
terraform-aws-infra/
  provider.tf:1        # aws ~>5.0, default_tags Project/Env/ManagedBy
  backend.tf:1         # S3 sravani-terraform-demo-bucket1 use_lockfile=true
  variables.tf:1       # vpc_cidr 10.0.0.0/16, subnets, t3.micro, key_name
  main.tf:1            # module vpc + web_sg + data aws_ami al2023 + module web_ec2
  outputs.tf:1         # vpc_id, public/private_subnet_ids, ec2_public_ip/dns
  modules/vpc/main.tf:1, modules/security-group/main.tf:1, modules/ec2/main.tf:1
  envs/dev/terraform.tfvars, terraform.tfvars.example, .gitignore, architecture.png
```

### Step-by-Step (Exactly as Executed)
```powershell
winget install --id Hashicorp.Terraform -e
terraform version # v1.15.8
aws sts get-caller-identity # terrafrom_user 968153778824 us-east-1
aws s3api get-bucket-versioning --bucket sravani-terraform-demo-bucket1 # Enabled
aws s3api put-bucket-versioning --bucket sravani-terraform-demo-bucket1 --versioning-configuration Status=Enabled
cd terraform-aws-infra
terraform fmt -recursive
terraform init -reconfigure # S3 backend
terraform validate # Success
terraform plan -out=tfplan # 16 to add, data aws_ami ami-0fe74bfcad4fd6bd2
terraform apply tfplan # First apply 15 ok, EC2 failed t2.micro not free-tier-eligible → changed variables.tf:42 t2.micro→t3.micro → apply 1 added i-00f507eb0af08bb0e 3.231.160.92
terraform output # vpc_id vpc-03b31ff28e41bc125, web_sg_id sg-063451d20929be4f5
curl http://3.231.160.92/ # <h1>sravani-devops - dev - ip-10-0-1-213.ec2.internal</h1>
# Create key for Ansible
aws ec2 create-key-pair --key-name sravani-key --query KeyMaterial --output text > ~/.ssh/sravani-key.pem
# set terraform.tfvars: key_name="sravani-key" → terraform apply → new EC2 i-00f507eb0af08bb0e 3.231.160.92
```

### Interview Talking Points (2 mins)
> “I used modules so VPC, SG, EC2 are reusable. Remote state in S3 with native locking avoids concurrent writes. I hit `412 PreconditionFailed` on `.tflock` — versioned bucket left delete markers, fixed by deleting all versions. I hit `InvalidParameterCombination t2.micro not free-tier` — `describe-instance-types --filters free-tier-eligible` showed `t3.micro` is, switched. I enforce `fmt`, `validate`, `plan -out` before `apply` for safe changes. Destroyed with `terraform destroy` to save cost.”

---

## Project 3 — Ansible Config Mgmt

### Why
One control node configures N EC2s idempotently. No SSH drift. Roles separate hardening vs app.

### Structure
```
ansible-config-mgmt/
  ansible.cfg:1                # inventory=inventory/hosts.ini, roles_path=./roles, yaml stdout
  inventory/hosts.ini:1        # web-1 ansible_host=3.231.160.92 ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/sravani-key.pem
  playbooks/site.yml:1         # pre_tasks dnf update, roles: hardening, webserver, post nginx started
  roles/hardening/tasks/main.yml:1 + handlers/main.yml # dnf-automatic, sshd PermitRootLogin no, fail2ban, postfix disable
  roles/webserver/tasks/main.yml:1 + handlers/main.yml # nginx, /etc/nginx/nginx.conf server { listen 80; /health }, index.html Configured by Ansible
  screenshots/ # ping, playbook, curl
```

### Step-by-Step
```bash
# WSL Ubuntu — pip via get-pip.py --break-system-packages (no apt pip)
python3 /tmp/get-pip.py --user --break-system-packages
~/.local/bin/pip install --user --break-system-packages ansible
~/.local/bin/ansible --version # 2.21.3
# Inventory already 3.231.160.92, key ~/.ssh/sravani-key.pem (chmod 400, copied from Windows C:/Users/srava/.ssh)
ANSIBLE_ROLES_PATH=./roles ~/.local/bin/ansible all -m ping -i inventory/hosts.ini # SUCCESS pong
ANSIBLE_ROLES_PATH=./roles ~/.local/bin/ansible-playbook playbooks/site.yml -i inventory/hosts.ini --check --diff # 7 changed (preview)
ANSIBLE_ROLES_PATH=./roles ~/.local/bin/ansible-playbook playbooks/site.yml -i inventory/hosts.ini # ok=12 changed=2 failed=0
# Fixes applied: handlers/missing Restart sshd, copy notify indent, firewalld not opening http
# Manual fix after first run: firewall-cmd --add-service=http --permanent; --reload (later added to role)
curl http://3.231.160.92/ # Configured by Ansible - web-1
curl http://3.231.160.92/health # ok
ssh -i ~/.ssh/sravani-key.pem ec2-user@3.231.160.92 "systemctl status nginx; cat /etc/nginx/nginx.conf"
```

### Interview Talking Points
> “I manage many EC2s from one control node via `inventory/hosts.ini`. Roles are idempotent — re-running changes nothing unless drift. I fixed world-writable `/mnt/c` ignoring `ansible.cfg` by `ANSIBLE_ROLES_PATH=./roles`, fixed missing `hardening/handlers/main.yml` for `notify: Restart sshd`, fixed `copy: notify` indent bug, and fixed firewalld blocking `80` by adding `firewalld: service=http immediate+permanent`. Verified with `curl` and `ss -tulpn`.”

---

## Project 1 — CI/CD Docker & K8s

### Why
Automated, gated, versioned, zero-downtime. Tests block broken deploys, `BUILD_NUMBER` tags allow rollback, `RollingUpdate` keeps 2/3 available.

### Structure
```
cicd-pipeline/ (remote cicd-python-app-k8s)
  app/app.py:1                 # Flask /, /health, /api/info
  app/requirements.txt:1       # flask 3.1, gunicorn 23, pytest 8.3
  app/test_app.py:1            # test_health, test_home gates pipeline
  Dockerfile:1                 # python:3.11-slim, pip install, gunicorn 0.0.0.0:5000
  k8s/deployment.yaml:1        # 3 replicas, RollingUpdate, sravani81299/sravani-app:1.0.1, liveness/readiness /health
  k8s/service.yaml:1           # LoadBalancer 80→5000
  jenkins/Jenkinsfile:1        # Checkout→Test→Build→Push→Deploy (sravani81299/sravani-app:${BUILD})
  screenshots/ # pytest, docker images, kubectl get pods/svc/deploy, curl
```

### Step-by-Step
```powershell
winget install --id Docker.DockerDesktop -e # 4.88.1 Engine 29.7.2 desktop-linux
docker --version # 29.7.2
py -m pip install -r app/requirements.txt
py -m pytest app/test_app.py -v # 2 passed
docker build -t sravani81299/sravani-app:1.0.0 -t sravani81299/sravani-app:latest . # 217MB
docker run -d -p 5000:5000 sravani81299/sravani-app:1.0.0; curl http://localhost:5000/health # {"status":"ok","version":"1.0.0"}
docker login -u sravani81299 # PAT dckr_pat_...
docker push sravani81299/sravani-app:1.0.0 # digest sha256:77eb...
# Bump app.py 1.0.0→1.0.1 Rolling Update Demo, pytest → 2 passed, docker build -t 1.0.1
winget install --id Kubernetes.kind -e # 0.33.0
kind create cluster --name sravani-kind --wait 3m # v1.37.0 Ready
kind load docker-image sravani81299/sravani-app:1.0.0 --name sravani-kind
kubectl apply -f k8s/deployment.yaml # sravani81299/sravani-app:1.0.0
kubectl apply -f k8s/service.yaml
kubectl rollout status deployment/sravani-app # 3/3 Running
kubectl port-forward svc/sravani-app-svc 8080:80 & curl http://localhost:8080/ # Version: 1.0.0
# Rolling update demo
kind load docker-image sravani81299/sravani-app:1.0.1 --name sravani-kind
kubectl set image deployment/sravani-app app=sravani81299/sravani-app:1.0.1
kubectl rollout status deployment/sravani-app # 2 updated, 1 old pending → 3 available 1.0.1
curl http://localhost:8080/ # Version: 1.0.1 - Rolling Update Demo
curl http://localhost:8080/health # {"status":"ok","version":"1.0.1"}
docker tag sravani81299/sravani-app:1.0.1 sravani81299/sravani-app:latest; docker push sravani81299/sravani-app:1.0.1; docker push latest
# Verify https://hub.docker.com/r/sravani81299/sravani-app/tags → 1.0.0, 1.0.1, latest
kind delete cluster --name sravani-kind # cleanup
```

### Jenkins (Interview Ready)
> “Jenkins triggers on GitHub webhook `/github-webhook/`. Pipeline: `credentials('dockerhub')` + `KUBECONFIG`, `sh 'pytest'`, `docker build -t $IMAGE`, `docker push $IMAGE + latest`, `sed -i s/sravani81299.*/$IMAGE/ k8s/deployment.yaml; kubectl apply; kubectl rollout status`. If `pytest` fails, deploy never runs. For local demo I used `kind` to avoid EKS cost; for AWS I’d use `aws ecr get-login-password` and EKS `LoadBalancer`.”

---

## End-to-End Proof

**Screenshots folder in each repo (proof beats prose):**
- `terraform-aws-infra/screenshots/` `terraform-output.txt` `terraform-state-list.txt` `aws-ec2-describe.txt` `architecture.png`
- `ansible-config-mgmt/screenshots/` `ansible-ping.txt` `ansible-playbook.txt` `curl-ec2.txt` `curl-ec2-health.txt`
- `cicd-pipeline/screenshots/` `pytest.txt` `docker-images.txt` `kubectl-get-pods.txt` `kubectl-get-svc.txt` `kubectl-get-deploy.txt` `kubectl-rollout-history.txt` `curl-k8s.txt`

**Live URLs (before `terraform destroy`):**
- `http://3.231.160.92/` + `/health` (Ansible)
- `http://localhost:8080/` via `kubectl port-forward` (K8s `1.0.1`)
- Docker Hub `https://hub.docker.com/r/sravani81299/sravani-app/tags`
- GitHub `https://github.com/sravanigaddam-devops/{terraform-aws-infra,ansible-config-mgmt,cicd-python-app-k8s}`

**.gitignore:** Python/Node template — no `node_modules/`, `__pycache__/`, `.venv/`, `*.pyc`, `.pytest_cache/`, `.env`, `*.pem`, `*.key`, `.terraform/`, `*.tfstate*` (keeps `terraform.tfvars.example`)

**Diagram:** `architecture.png` + `architecture.drawio` editable at draw.io

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `412 PreconditionFailed` S3 PutObject `.tflock` | Versioned bucket left delete markers → `aws s3api delete-object --version-id` all versions + `terraform force-unlock <ID>` |
| `InvalidParameterCombination t2.micro not free-tier` | `describe-instance-types --filters free-tier-eligible` → `t3.micro` in `variables.tf:42` |
| `ansible.cfg` ignored world-writable `/mnt/c` | `ANSIBLE_ROLES_PATH=./roles` + `cd` into repo root |
| `handler 'Restart sshd' not found` | Added `roles/hardening/handlers/main.yml` |
| `copy: notify unsupported` | Moved `notify:` from inside `copy:` to task level `  notify:` |
| `firewalld` blocks 80 after `service: started` | Added `firewalld: service=http/https` `port=8080/tcp` `permanent+immediate` |
| `pip: No module named pip` WSL Ubuntu 24.04 | `curl get-pip.py --user --break-system-packages` then `~/.local/bin/pip install --break-system-packages` |
| `docker: command not found` | `winget install Docker.DockerDesktop` → `npipe:////./pipe/docker_engine` + add to Path |
| `kubectl current-context not set` | `kind create cluster --name sravani-kind` → `kind-sravani-kind` |

---

## How to Explain in Interview

**2-Minute Version:**
> “I started with Terraform — modular VPC with S3 remote state and native locking. That gave me a public EC2. Then Ansible from one control node hardened SSH and deployed nginx with `/health`. Finally I built a Flask app with pytest gating, Dockerized it with versioned tags to Docker Hub, and rolling-deployed 3 replicas to Kubernetes — verified with `port-forward` and `curl`. All infra is destroyed to save cost but repos and images remain.”

**5-Minute Version:** Add tech versions, commands above, and one troubleshooting story (pick S3 lock or firewalld).

**10-Minute Deep Dive:** Walk `provider.tf:1` → `modules/vpc/main.tf:1` → `inventory/hosts.ini:1` → `roles/webserver/tasks/main.yml:7` → `Dockerfile:1` → `k8s/deployment.yaml:7` → `jenkins/Jenkinsfile:10`

**STAR for ‘State Lock’:**
> S: `terraform plan` got `412` lock. T: Unblock without `-lock=false`. A: Listed `s3api list-object-versions` found 3 versions + 2 delete markers, deleted all versions, `force-unlock`. R: `plan` succeeded, added to docs to clean `.tflock` if reoccurs.

**Questions You Can Ask Interviewer:**
- Do you use EKS or self-managed K8s? I used `kind` locally to save cost, can switch to EKS.
- Do you prefer GitOps (ArgoCD) vs Jenkins for CD? My Jenkins pipeline maps 1:1 to Argo.

---

## Commands Cheat Sheet

```powershell
# Terraform
terraform init; terraform validate; terraform plan -out=tfplan; terraform apply tfplan; terraform output; terraform destroy

# Ansible (WSL)
ANSIBLE_ROLES_PATH=./roles ~/.local/bin/ansible all -m ping -i inventory/hosts.ini
ANSIBLE_ROLES_PATH=./roles ~/.local/bin/ansible-playbook playbooks/site.yml -i inventory/hosts.ini

# CI/CD
py -m pytest app/test_app.py -v
docker build -t sravani81299/sravani-app:1.0.1 .; docker push sravani81299/sravani-app:1.0.1
kind create cluster --name sravani-kind; kind load docker-image sravani81299/sravani-app:1.0.1 --name sravani-kind
kubectl apply -f k8s/deployment.yaml; kubectl apply -f k8s/service.yaml; kubectl rollout status deployment/sravani-app
kubectl port-forward svc/sravani-app-svc 8080:80; curl http://localhost:8080/health
```

---

## Repos & Cleanup

```powershell
git log --oneline -3 # each repo cf6a68b, 728d0b8, 99bd54a
terraform destroy -auto-approve
kind delete cluster --name sravani-kind
aws ec2 delete-key-pair --key-name sravani-key # optional
# Keep: GitHub repos, Docker Hub tags, ~/.ssh/sravani-key.pem
```

*Generated 2026-08-29 — all outputs from live runs in `screenshots/`.*
