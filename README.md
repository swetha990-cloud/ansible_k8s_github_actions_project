# devops-demo-Real Time Project
# training batch demo Real Time Project


# 🚀 CI/CD Deployment to AWS EKS with GitHub Actions + Ansible

This repository implements a **complete CI/CD pipeline** for deploying a containerised application to an **AWS EKS (Kubernetes) cluster**.

## Jenkins → GitHub Actions mapping

* **Checkout Code** → `actions/checkout`
* **Build Jar** → `mvn clean package`
* **SonarQube Scan** → `sonar-maven-plugin` with `SONAR_TOKEN`
* **Build Docker Image** → `docker build`
* **Scan Docker Image** → `aquasecurity/trivy-action`
* **Push Image to DockerHub** → `docker/login-action` + `docker push`
* **Update Kubernetes Deployment Manifest in GitHub repo** → `git commit` & `push` with `GITHUB_TOKEN`
* **K8s Deployment using Ansible** → `ansible-playbook` step (runner must have kube access)

---

## 📂 Repository Structure

```
.
├── Application/
│   ├── pom.xml                # Java/Maven project
│   └── src/                   # Application source code
│
├── Ansible/
│   ├── ansible_k8s_deploy_playbook.yaml  # Ansible playbook to deploy to EKS
│   └── k8s_deployment.yaml               # Kubernetes Deployment manifest
│
└── .github/
    └── workflows/deploy.yaml   # GitHub Actions pipeline
```

---

## 🔑 Prerequisites

* An existing **AWS EKS cluster** with worker nodes running
* **Docker Hub account** for hosting images
* GitHub repository with the following **Secrets** configured:

| Secret Name             | Purpose                                           |
| ----------------------- | ------------------------------------------------- |
| `AWS_ACCESS_KEY_ID`     | AWS IAM Access Key with EKS + ECR permissions     |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM Secret Key                                |
| `KUBECONFIG_DATA`       | Base64-encoded kubeconfig of your EKS cluster     |
| `GITHUB_TOKEN`          | Provided automatically by GitHub (no need to set) |
| `DOCKER_USERNAME`       | DockerHub username                                |
| `DOCKER_PASSWORD`       | DockerHub password/token                          |
| `SONAR_HOST_URL`        | e.g. `http://3.88.43.182:9000`                    |
| `SONAR_TOKEN`           | SonarQube token                                   |

---

## ⚙️ Pipeline Flow

1. **Checkout code**
   GitHub Actions checks out the repo.

2. **Configure AWS credentials**
   Needed to talk to EKS.

3. **Configure kubeconfig**
   Decoded from `KUBECONFIG_DATA` and stored at `$HOME/.kube/config`.

4. **Build and scan Docker image**
   Maven builds JAR; Docker builds image; Trivy scans it.

5. **Push Docker image**
   Push to Docker Hub.

6. **Update Kubernetes manifest**
   `sed` replaces the image tag in `k8s_deployment.yaml`.

7. **Run Ansible playbook**
   Ansible runs `kubectl apply` to deploy to EKS.

---

##  GitHub Actions Workflow

`.github/workflows/ci-cd.yaml`:

```
https://github.com/jadalaramani/Ansible_k8s_gha/blob/main/.github/workflows/ci-cd.yaml
```

---

## 📝 Ansible Playbook

`Ansible/ansible_k8s_deploy_playbook.yaml`:

```
https://github.com/jadalaramani/Ansible_k8s_gha/blob/main/Ansible/ansible_k8s_deploy_playbook.yaml
```
---

## 📝 Kubernetes Deployment Manifest

`Ansible/k8s_deployment.yaml`:
```
https://github.com/jadalaramani/Ansible_k8s_gha/blob/main/Ansible/k8s_deployment.yaml
```

## Notes

* `github.run_number` acts like Jenkins `${BUILD_NUMBER}`.
* The `Update YAML + Push` step commits back to the same repo.
* The Ansible step assumes your `playbook` uses the updated YAML file.
---
Implementation:

## Step1: Install kubectl & eksctl

```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client


curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```
```
aws configure
```
```
eksctl create cluster --name my-cluster 
```

## Docker && Sonarqube
```
yum install docker 
systemctl start docker
systemctl enable docker
docker run -itd --name sonar -p 9000:9000 sonarqube:lts
```
## Base64 kubeconfig
```
cat ~/.kube/config | base64 -w 0
```
## 🛠 Troubleshooting

* **`No inventory was parsed` warning:**
  Safe to ignore since we’re running on localhost with `connection: local`.

* **`localhost:8080` connection refused:**
  Kubeconfig or AWS credentials not set. Check that the `KUBECONFIG_DATA` secret is correct and AWS creds are configured.

* **Push rejected on GitHub:**
  Add `fetch-depth: 0` to checkout step and do a `git pull --rebase` before pushing.

* **Manifest path not found:**
  Use relative path `{{ playbook_dir }}/k8s_deployment.yaml` instead of hardcoding `/home/ec2-user`.

---

## ✅ Success Criteria

* Docker image built, scanned, and pushed to Docker Hub.
* Kubernetes manifest updated with the new image tag.
* Ansible applies the manifest to EKS.
* GitHub Actions job completes successfully.

