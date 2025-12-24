I’implemeneted completes end-to-end CI/CD on GCP in a beginner-friendly, step-by-step way, keeping Free Trial limits in mind.
We’ll use GitHub + Google Cloud Build + GKE + Artifact Registry
Final Architecture (Simple & Real-World)
Developer → GitHub (code push)
          → Cloud Build (CI)
              → Build Docker image
              → Push to Artifact Registry
              → Deploy to DEV GKE
              → Manual Approval
              → Deploy to PROD GKE
Prerequisites (Do This First)
1️⃣ GCP Free Trial Account
•	$300 credits
•	Billing must be enabled
2️⃣ Install tools on your system
gcloud --version
kubectl version --client
docker --version

STEP 1️⃣ Create a GCP Project (Container)
Think of project = folder for everything
gcloud projects create cicd-demo
gcloud config set project cicd-demo
Enable billing (must do in console).
1.create service account CI/CD service account
Cloud Build permissions
gcloud projects add-iam-policy-binding cicd-demo-project-481705 ^
--member=serviceAccount:service-account-cicd-pipline@cicd-demo-project-481705.iam.gserviceaccount.com ^
--role=roles/cloudbuild.builds.builder
Artifact Registry push
gcloud projects add-iam-policy-binding cicd-demo-project-481705 ^
--member=serviceAccount:service-account-cicd-pipline@cicd-demo-project-481705.iam.gserviceaccount.com ^
--role=roles/artifactregistry.writer
GKE deploy
gcloud projects add-iam-policy-binding cicd-demo-project-481705 ^
--member=serviceAccount:service-account-cicd-pipline@cicd-demo-project-481705.iam.gserviceaccount.com ^
--role=roles/container.developer
Cloud Logging
gcloud projects add-iam-policy-binding cicd-demo-project-481705 ^
--member=serviceAccount:service-account-cicd-pipline@cicd-demo-project-481705.iam.gserviceaccount.com ^
--role=roles/logging.logWriter
TEP 2️⃣ Enable Required Services
This allows GCP to create clusters and pipelines.
gcloud services enable container.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com
 

STEP 3️⃣ Create a Place to Store Docker Images
👉 Docker images cannot be pushed directly to GKE
👉 First, store them in Artifact Registry
gcloud artifacts repositories create my-docker-test-repo \
  --repository-format=docker \
  --location=asia-south1
 
Now tell Docker to trust GCP:
gcloud auth configure-docker asia-south1-docker.pkg.dev
 

STEP 3️⃣ Create 2 GKE Clusters (DEV & PROD)
gcloud components install kubectl
 
 
After that kubectl is installed successfully.
To check kubectl installed or not kindly verified

kubectl version –client
 

DEV Cluster
gcloud container clusters create dev-cluster \
  --zone asia-south1-a \
  --num-nodes=1 \
  --machine-type=e2-small
PROD Cluster
gcloud container clusters create prod-cluster2\
  --zone asia-south1-a \
  --num-nodes=1 \
  --machine-type=e2-small
 

Get credentials:
gcloud container clusters get-credentials dev-cluster --zone asia-south1-a
 
gcloud container clusters get-credentials prod-cluster --zone asia-south1-a

STEP 4️⃣ Create GitHub Repository
gcp-cicd-demo/
├── app.py
├── Dockerfile
├── deployment-dev.yaml
├── deployment-prod.yaml
└── cloudbuild.yaml
STEP 5️⃣ Sample Application (Python)
FROM python:3.9-slim
WORKDIR /app
COPY app.py .
RUN pip install flask
CMD ["python", "app.py"]


STEP 6️⃣ Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY app.py .
RUN pip install flask
CMD ["python", "app.py"]


STEP 7️⃣ Kubernetes Deployment Files
deployment-dev.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-dev
  template:
    metadata:
      labels:
        app: app-dev
    spec:
      containers:
      - name: app
        image: IMAGE_NAME
        ports:
        - containerPort: 8080

deployment-prod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-prod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-prod
  template:
    metadata:
      labels:
        app: app-prod
    spec:
      containers:
      - name: app
        image: IMAGE_NAME
        ports:
        - containerPort: 8080

STEP 8️⃣ Cloud Build CI/CD Pipeline
cloudbuild.yaml
steps:
# Build Docker Image
- name: 'gcr.io/cloud-builders/docker'
  args:
    - build
    - '-t'
    - 'asia-south1-docker.pkg.dev/$cicd-demo-project-481705/my-test-docker-repo/app:$SHORT_SHA'
    - '.'

# Push Image
- name: 'gcr.io/cloud-builders/docker'
  args:
    - push
    - 'asia-south1-docker.pkg.dev/$cicd-demo-project-481705/my-test-docker-repo/app:$SHORT_SHA'

# Deploy to DEV
- name: 'gcr.io/cloud-builders/kubectl'
  args:
    - apply
    - '-f'
    - deployment-dev.yaml
  env:
    - 'CLOUDSDK_COMPUTE_ZONE=asia-south1-a'
    - 'CLOUDSDK_CONTAINER_CLUSTER=dev-cluster'

# Manual Approval
- name: 'gcr.io/cloud-builders/gcloud'
  args:
    - builds
    - submit
  waitFor: ['-']

# Deploy to PROD
- name: 'gcr.io/cloud-builders/kubectl'
  args:
    - apply
    - '-f'
    - deployment-prod.yaml
  env:
    - 'CLOUDSDK_COMPUTE_ZONE=asia-south1-a'
    - 'CLOUDSDK_CONTAINER_CLUSTER=prod-cluster2'

images:
- asia-south1-docker.pkg.dev/$cicd-demo-project-481705/my-test-docker-repo/app:$SHORT_SHA



STEP 9️⃣ Connect GitHub to Cloud Build (Auto Trigger)
1.	Go to Cloud Build → Triggers
2.	Create Trigger
3.	Select GitHub
4.	Repo: gcp-cicd-demo
5.	Event: Push to branch
6.	Branch: main
7.	Build config: cloudbuild.yaml
✅ Now pipeline triggers automatically on code push

Pipeline run successfully now how to access application on URL
1.check cluster nodes status
 
2.check your pods is running or not
 
3.how to get external ip to access application
 

STEP 🔟 Approval Before PROD Deployment
1.	Cloud Build → Settings
2.	Enable Manual Approval
3.	PROD step will pause
4.	Click Approve → Deploys to PROD
STEP 1️⃣1️⃣ Test Deployment
kubectl get pods
kubectl get svc
Expose service:
kubectl expose deployment app-dev \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8080
Open External IP 🎉
________________________________________
✅ Final Result
✔ Code push → CI/CD auto triggered
✔ Docker image built & pushed
✔ Auto deploy to DEV
✔ Manual approval
✔ Deploy to PROD






