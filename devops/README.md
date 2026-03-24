# BloggerApp DevOps: Environment Setup & CI/CD Flow

This guide walks you through the complete process of creating your AWS EKS infrastructure for the first time, deploying your application, and understanding how the continuous deployment (CI/CD) automated pipeline works.

## Phase 1: First-Time Initialization (The Complete Setup)

### Step 1: Create AWS Infrastructure with Terraform
The very first thing you need to do is provision your cloud infrastructure on AWS. This will physically create your Virtual Private Cloud, EC2 worker nodes, and the EKS cluster.
```bash
# Navigate to the terraform directory
cd devops/terraform

# Initialize Terraform plugins
terraform init

# Apply the infrastructure config (takes ~15 minutes to run)
terraform apply -auto-approve
```

### Step 2: Connect your local terminal to the AWS EKS Cluster
Once Terraform finishes, your AWS resources are built, but your local computer doesn't know how to talk to the EKS cluster yet! You must pull the correct authentication keys into your terminal to use `kubectl`:
```bash
# Update kubeconfig to connect to your EKS cluster
aws eks update-kubeconfig --region ap-south-1 --name bloggerapp-eks
```

### Step 3: Deploy the Kubernetes (k8s) YAML Manifests
Now that you are connected to the cluster, you must submit the configuration rules and deploy the actual application pods inside it.
```bash
# Navigate back to the root of the project
cd ../..

# 3.1: Create isolated namespaces for frontend, backend, and monitoring
kubectl apply -f devops/k8s/namespaces.yaml

# 3.2: Manually inject the Backend secrets
# Note: First, you encrypt your Mongo Database URL to base64:
# echo -n "real_mongo_db_url" | base64
# Then place that scrambled string inside backend-secret.yaml, and deploy it:
kubectl apply -f devops/k8s/backend-secret.yaml

# 3.3: Deploy the Backend APIs and Frontend React App 
kubectl apply -f devops/k8s/backend-deployment.yaml
kubectl apply -f devops/k8s/frontend-deployment.yaml

# 3.4: Install Prometheus and Grafana for monitoring health
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f devops/k8s/monitoring-values.yaml
```

### Step 4: Extract the Live Service URLs
Everything is deployed gracefully. AWS will provision LoadBalancers to give you real internet-facing URLs. Run these commands to grab the exact URL of your endpoints:
```bash
# Get the Frontend Web App URL:
kubectl get svc -n frontend frontend-service -o jsonpath='{"http://"}{.status.loadBalancer.ingress[0].hostname}'

# Get the Grafana Dashboard URL (Login: admin / admin):
kubectl get svc -n monitoring monitoring-grafana -o jsonpath='{"http://"}{.status.loadBalancer.ingress[0].hostname}:3000'
```
*At this point — your live AWS environment is completely set up and functional!*

---

## Phase 2: Updating the Application (Continuous Deployment)

The infrastructure is already up and running. From now on, you **do NOT need to run the setup Steps 1 to 4** again just to push new code to your application!

### What happens when you change the frontend or backend code?
When you modify your code locally on your laptop (either the React UI or Node.js Backend API), you don't have to manually upload it to the Kubernetes Pods. The system acts autonomously via CI/CD pipelines.

### The Developer Workflow steps:
1. Make code changes in your local `client/` or `server/` directory.
2. Add and commit those changes using Git.
3. Push the changes to GitHub.

```bash
# Push your local code updates
git add .
git commit -m "Updated the frontend homepage design"
git push origin main
```

### What happens automatically under the hood?
1. **GitHub Actions is triggered**: Pushing to the `main` branch immediately alerts the GitHub Actions Runner.
2. **Autobuild Dockerization**: GitHub Actions builds a brand-new Docker image for your updated `frontend` or `backend` code inside their cloud servers.
3. **Pushed to DockerHub**: The newly built Docker image is uploaded securely to your DockerHub container repository.
4. **Pulled to EKS Pods**: Finally, GitHub Actions remotely logs into your AWS EKS cluster and instructs the backend/frontend deployments to terminate the old containers and automatically pull down the newly updated image.

*(You can literally watch this pipeline run under the `Actions` tab on your GitHub repository. Your EKS pods will update dynamically with zero downtime).*

---

## Phase 3: Cost Management (Tear Down)
Always shut down the cluster when you finish developing to avoid being billed endlessly by AWS (~$150/month).

```bash
# 1. Clean out the LoadBalancers from Kubernetes first 
# (Required before Terraform destroy, so AWS routing is clear)
kubectl delete namespace frontend backend monitoring

# 2. Destroy the AWS Servers permanently
cd devops/terraform
terraform destroy -auto-approve
```
