# CI/CD Pipeline with GitHub Actions & Docker Setup Guide

## Prerequisites Setup

### 1. Multipass Ubuntu Environment
```bash
# On macOS, ensure Multipass is running
multipass list
multipass shell <your-ubuntu-instance>
```

### 2. Install Required Tools in Ubuntu
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER

# Install Docker Compose
sudo apt install docker-compose -y

# Install kubectl and Minikube
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/

# Install Git
sudo apt install git -y

# Logout and login again for docker group changes
exit
multipass shell <your-ubuntu-instance>
```

## Project Structure

Create your project directory:
```bash
mkdir cicd-demo && cd cicd-demo
git init
```

Create the following structure:
```
cicd-demo/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── app/
│   ├── app.py
│   ├── requirements.txt
│   └── test_app.py
├── Dockerfile
├── docker-compose.yml
├── output_images
├── k8s/
│   ├── deployment.yml
│   └── service.yml
└── README.md
```

## 1. Sample Application Files

### app/app.py

### app/requirements.txt

### app/test_app.py

## 2. Docker Configuration

### Dockerfile
```dockerfile
FROM python:3.9-slim

WORKDIR /app

# Copy requirements first for better caching
COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app/ .

# Create non-root user
RUN useradd --create-home --shell /bin/bash appuser && \
    chown -R appuser:appuser /app
USER appuser

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

CMD ["python", "app.py"]
```

### docker-compose.yml

## 3. GitHub Actions Workflow

### .github/workflows/ci-cd.yml
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  DOCKER_HUB_USERNAME: ${{ secrets.DOCKER_HUB_USERNAME }}
  DOCKER_HUB_TOKEN: ${{ secrets.DOCKER_HUB_TOKEN }}
  IMAGE_NAME: cicd-demo-app

jobs:
  test:
    runs-on: ubuntu-latest
    name: Test Application
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
        
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r app/requirements.txt
        
    - name: Run tests
      run: |
        cd app
        pytest test_app.py -v --tb=short

  build-and-push:
    runs-on: ubuntu-latest
    needs: test
    name: Build and Push Docker Image
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
      
    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ env.DOCKER_HUB_USERNAME }}
        password: ${{ env.DOCKER_HUB_TOKEN }}
        
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
          
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        file: ./Dockerfile
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy:
    runs-on: ubuntu-latest
    needs: build-and-push
    name: Deploy to Local Environment
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Deploy notification
      run: |
        echo "Deployment would happen here"
        echo "Image: ${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}:latest"
        echo "Ready for local deployment!"
```

## 4. Kubernetes Configuration

### k8s/deployment.yml

### k8s/service.yml

## 5. Setup Instructions

### Step 1: Create Docker Hub Account
1. Go to https://hub.docker.com and create an account
2. Create an access token: Settings > Security > New Access Token

### Step 2: Setup GitHub Repository
```bash
# Initialize git repository
git add .
git commit -m "Initial commit"

# Create GitHub repository and push
gh repo create cicd-demo --public --source=. --remote=origin --push
```

### Step 3: Configure GitHub Secrets
1. Go to your GitHub repository
2. Settings > Secrets and variables > Actions
3. Add secrets:
   - `DOCKER_HUB_USERNAME`: Your Docker Hub username
   - `DOCKER_HUB_TOKEN`: Your Docker Hub access token

### Step 4: Test the Pipeline
```bash
# Make a change and push
echo "# CI/CD Demo" > README.md
git add .
git commit -m "Add README"
git push origin main
```

## 6. Local Deployment with Minikube

### Start Minikube
```bash
# Start Minikube
minikube start --driver=docker

# Enable ingress addon
minikube addons enable ingress

# Check status
minikube status
kubectl get nodes
```

### Deploy Application
```bash
# Update deployment.yml with your Docker Hub username
sed -i 's/YOUR_DOCKERHUB_USERNAME/your-actual-username/g' k8s/deployment.yml

# Deploy to Kubernetes
kubectl apply -f k8s/deployment.yml
kubectl apply -f k8s/service.yml

# Check deployment
kubectl get pods
kubectl get services

# Get service URL
minikube service cicd-demo-service --url
```

### Access the Application
```bash
# Port forward for easy access
kubectl port-forward service/cicd-demo-service 5000:5000

# Test the application
curl http://localhost:5000
curl http://localhost:5000/health
curl http://localhost:5000/api/data
```

## 7. Testing and Validation

### Local Testing
```bash
# Build and test locally
docker-compose up --build

# Run tests
docker-compose run test

# Test API endpoints
curl http://localhost:5000
curl http://localhost:5000/health
```

### Monitor CI/CD Pipeline
1. Check GitHub Actions tab in your repository
2. Monitor Docker Hub for image builds
3. Verify Kubernetes deployment: `kubectl get all`

## 8. Cleanup Commands

```bash
# Stop and clean up Minikube
kubectl delete -f k8s/
minikube stop
minikube delete

# Clean up Docker
docker-compose down
docker system prune -a
```

## Troubleshooting

### Common Issues:
1. **Docker permission denied**: Run `sudo usermod -aG docker $USER` and logout/login
2. **Minikube won't start**: Try `minikube delete` then `minikube start`
3. **GitHub Actions failing**: Check secrets are properly configured
4. **Port conflicts**: Use different ports in docker-compose.yml

### Useful Commands:
```bash
# Check logs
kubectl logs -f deployment/cicd-demo-app
docker-compose logs

# Debug
kubectl describe pod <pod-name>
docker exec -it <container-id> /bin/bash
```