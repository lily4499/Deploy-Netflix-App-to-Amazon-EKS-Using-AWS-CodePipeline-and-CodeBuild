Of course! Here's your entire **project properly rewritten as a `README.md`** file — ready for **GitHub**, including **file structure**, **all files content**, and **clear steps**.

---

# 📦 AWS CI/CD: Deploy Netflix App to Amazon EKS Using AWS CodePipeline and CodeBuild

---

## 🎯 Objective

Build a complete **CI/CD pipeline** on AWS that:

- Pulls code from GitHub
- Builds and pushes a Docker image to Amazon ECR
- Deploys the updated app to Amazon EKS automatically

---

## 🧠 Architecture Diagram

```plaintext
GitHub (Source Code)
   ↓
AWS CodePipeline
   ↓
AWS CodeBuild #1 (Build & Push Docker image to ECR)
   ↓
AWS CodeBuild #2 (Deploy updated image to Amazon EKS)
   ↓
Amazon EKS (Netflix App Live)
```

---

## 📂 Project Structure

```bash
aws-cicd-eks/
├── buildspec-ecr.yml        # Build & push Docker image to ECR
├── buildspec-eks.yml        # Build, push image, and deploy to EKS
├── deployment.yml           # Kubernetes deployment manifest
├── scripts/
│   └── create-codebuild-role.sh  # Script to create IAM role for kubectl
└── README.md
```

---

# 🛠 Step-by-Step Instructions

---

## 1️⃣ Create and Prepare EKS Cluster

✅ Create an EKS cluster if you haven't already:

```bash
eksctl create cluster --name my-cluster --region us-east-1 --nodes 2 --node-type t3.medium
```

---

## 2️⃣ Prepare Source Code

✅ Use this repo:  
👉 [https://github.com/lily4499/netflix-app.git](https://github.com/lily4499/netflix-app.git)

✅ Add a Kubernetes deployment file: `deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netflix-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: netflix
  template:
    metadata:
      labels:
        app: netflix
    spec:
      containers:
      - name: netflix
        image: 637423529262.dkr.ecr.us-east-1.amazonaws.com/netflix-app:latest
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: netflix-service
spec:
  selector:
    app: netflix
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: LoadBalancer
```

---

## 3️⃣ Create Amazon ECR Repository

✅ Create ECR repository to host your Docker images:

```bash
aws ecr create-repository --repository-name netflix-app --region us-east-1
```

---

## 4️⃣ Create Buildspec Files

---

### 🔹 `buildspec-ecr.yml`

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 637423529262.dkr.ecr.us-east-1.amazonaws.com
      - IMAGE_REPO_NAME=637423529262.dkr.ecr.us-east-1.amazonaws.com/netflix-app
      - IMAGE_TAG=latest
  build:
    commands:
      - echo Build started on `date`
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
  post_build:
    commands:
      - echo Build completed on `date`
      - docker push $IMAGE_REPO_NAME:$IMAGE_TAG
```

---

### 🔹 `buildspec-eks.yml`

```yaml
version: 0.2

phases:
  install:
    commands:
      - echo Installing kubectl...
      - curl -LO https://dl.k8s.io/release/v1.27.2/bin/linux/amd64/kubectl
      - chmod +x kubectl
      - mv kubectl /usr/local/bin/
  pre_build:
    commands:
      - echo Logging into ECR and configuring kubeconfig...
      - aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 637423529262.dkr.ecr.us-east-1.amazonaws.com
      - aws eks update-kubeconfig --name my-cluster --region us-east-1
  build:
    commands:
      - echo Building Docker image...
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
  post_build:
    commands:
      - echo Pushing Docker image and deploying to EKS...
      - docker push $IMAGE_REPO_NAME:$IMAGE_TAG
      - kubectl apply -f deployment.yml
```

---

## 5️⃣ Create IAM Role for CodeBuild (Kubectl Access)

---

### 🔹 Script: `scripts/create-codebuild-role.sh`

```bash
#!/usr/bin/env bash
TRUST="{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"codebuild.amazonaws.com\"},\"Action\":\"sts:AssumeRole\"}]}"
echo '{ "Version": "2012-10-17", "Statement": [{ "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" }] }' > /tmp/iam-role-policy

aws iam create-role --role-name CodeBuildKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'

aws iam put-role-policy --role-name CodeBuildKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy

aws iam attach-role-policy --role-name CodeBuildKubectlRole --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
aws iam attach-role-policy --role-name CodeBuildKubectlRole --policy-arn arn:aws:iam::aws:policy/AWSCodeBuildAdminAccess
aws iam attach-role-policy --role-name CodeBuildKubectlRole --policy-arn arn:aws:iam::aws:policy/AWSCodeCommitFullAccess
aws iam attach-role-policy --role-name CodeBuildKubectlRole --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam attach-role-policy --role-name CodeBuildKubectlRole --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess
```

✅ Run the script:

```bash
bash scripts/create-codebuild-role.sh
```

✅ Update the AWS Auth config in EKS:

```bash
eksctl create iamidentitymapping \
  --cluster my-cluster \
  --region us-east-1 \
  --arn arn:aws:iam::637423529262:role/CodeBuildKubectlRole \
  --group system:masters \
  --username CodeBuildKubectlRole
```

---

## 6️⃣ Create CodeBuild Projects

✅ Create **two** AWS CodeBuild projects:

---

### Project 1: Build and Push to ECR

| Setting | Value |
|---|---|
| Project Name | `netflix-build-ecr` |
| Source | GitHub (your repo) |
| Environment | Ubuntu, Standard runtime |
| Privileged Mode | ✅ Enabled |
| Service Role | `CodeBuildKubectlRole` |
| Buildspec File | `buildspec-ecr.yml` |
| Artifacts | None |

✅ After creation → **Start Build**.

---

### Project 2: Build, Push & Deploy to EKS

| Setting | Value |
|---|---|
| Project Name | `netflix-build-eks` |
| Source | GitHub (same repo) |
| Environment | Ubuntu, Standard runtime |
| Privileged Mode | ✅ Enabled |
| Service Role | `CodeBuildKubectlRole` |
| Buildspec File | `buildspec-eks.yml` |
| Artifacts | None |

✅ After creation → **Start Build**.

---

## 7️⃣ (Optional) Automate with AWS CodePipeline

✅ Create a CodePipeline with these stages:

- **Source:** GitHub (netflix-app repo)
- **Build:** Run `netflix-build-ecr`
- **Deploy:** Run `netflix-build-eks`

✅ Now every push to GitHub will **automatically build, push, and deploy**.

---

# 🎯 Final Checklist

| Step | Action | Purpose |
|:---|:---|:---|
| 1 | Create EKS Cluster | Host app |
| 2 | Prepare GitHub Repo | Source code |
| 3 | Create ECR Repo | Store Docker images |
| 4 | Create Buildspec Files | Define build and deployment |
| 5 | Create IAM Role | Allow kubectl access |
| 6 | Create CodeBuild Projects | Build & Deploy |
| 7 | Run Builds | Push Image, Deploy App |
| 8 | (Optional) Set up CodePipeline | Automate everything |

---

# 🏆 Output

- Docker image is stored in Amazon ECR ✅
- Netflix app is deployed in Amazon EKS ✅
- GitHub → CodePipeline → ECR → EKS full CI/CD ✅
- Application available through a public LoadBalancer URL ✅

---

# 🚀 Notes

- Remember to **set IAM permissions properly** or builds will fail at `kubectl apply`.
- Always enable **Privileged Mode** for Docker builds inside CodeBuild.
- Use `kubectl get svc` to find the LoadBalancer URL.

---

---

---

## 🛡️ (Optional) Add Manual Approval Stage in CodePipeline

### Why Manual Approval?

- Prevents automatic deployments into production.
- Allows QA, Manager, or Engineer to manually validate the build.
- Useful for higher safety and control.

---

### How to Add:

1. Open **AWS Console** → **CodePipeline** → Select **Your Pipeline**.
2. Click **Edit**.
3. Click **+ Add Stage** between the **Build** and **Deploy** stages:
   - **Name it:** `ManualApproval`
4. Inside the new stage:
   - Click **+ Add Action Group**
   - **Action Name:** `ApproveDeployment`
   - **Action Provider:** `Manual approval`
   - *(Optional)* Enable **SNS Notifications** to send emails when approval is needed.
5. Click **Save** to update the pipeline.

---
## CodePipeline Flow with Manual Approval

```plaintext
GitHub (Source Code)
   ↓
AWS CodeBuild (Build Docker Image + Push to ECR)
   ↓
🛡️ Manual Approval (Human Approval Required)
   ↓
AWS CodeBuild (Deploy Updated Image to Amazon EKS)
   ↓
Amazon EKS (Netflix App Live)


```

