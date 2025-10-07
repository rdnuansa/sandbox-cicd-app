# Sandbox CI/CD Application

A minimal Node.js web application demonstrating container-based CI/CD deployment using Docker, Docker Compose, and GitHub Actions. This project serves as a hands-on learning sandbox for modern DevOps practices with a clear migration path to Kubernetes.

## 🎯 Project Purpose

This sandbox project demonstrates:
- **Trunk-based development** workflow using `master` branch
- **Containerization** with Docker and multi-stage builds
- **Automated CI/CD** pipeline with GitHub Actions
- **Container orchestration** with Docker Compose
- **Cloud deployment** to Alibaba Cloud
- **Production-ready** setup with health checks and graceful shutdown

## 📋 Prerequisites

- Docker (v20.10+)
- Docker Compose (v2.0+)
- Node.js (v18+) for local development
- GitHub account
- Alibaba Cloud instance with Docker installed

## 🚀 Quick Start

### Local Development

1. **Clone the repository**
   ```bash
   git clone https://github.com/YOUR_USERNAME/sandbox-cicd-app.git
   cd sandbox-cicd-app
   ```

2. **Install dependencies**
   ```bash
   npm install
   ```

3. **Run locally (without Docker)**
   ```bash
   npm start
   # Visit http://localhost:3000
   ```

### Docker Development

1. **Build the Docker image**
   ```bash
   docker build -t sandbox-cicd-app:dev .
   ```

2. **Run with Docker Compose**
   ```bash
   # Copy environment file
   cp .env.example .env
   
   # Update .env with your settings
   # For local development:
   REGISTRY_URL=localhost
   IMAGE_NAME=sandbox-cicd-app
   IMAGE_TAG=dev
   
   # Start the application
   docker compose up -d
   
   # View logs
   docker compose logs -f web
   
   # Stop the application
   docker compose down
   ```

3. **Access the application**
   - Main page: http://localhost
   - Health check: http://localhost/health

## 🌿 Development Workflow (Trunk-Based Development)

This project follows trunk-based development using `master` as the main branch.

### Branch Naming Conventions

- `master` - Main branch (always deployable)
- `feature/feature-name` - New features
- `fix/bug-description` - Bug fixes
- `hotfix/critical-fix` - Production hotfixes

### Workflow

1. **Create a feature branch**
   ```bash
   git checkout -b feature/my-feature
   ```

2. **Make changes and commit**
   ```bash
   git add .
   git commit -m "feat: add new feature"
   ```

3. **Push to GitHub**
   ```bash
   git push origin feature/my-feature
   ```

4. **Create Pull Request**
   - Open PR against `master`
   - Ensure CI checks pass
   - Request code review
   - Merge when approved

5. **Auto-deployment**
   - Once merged to `master`, GitHub Actions automatically:
     - Runs tests
     - Builds Docker image
     - Pushes to GitHub Container Registry
     - Deploys to Alibaba Cloud

### Commit Message Convention

Follow conventional commits:
- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation changes
- `refactor:` - Code refactoring
- `test:` - Adding tests
- `chore:` - Maintenance tasks

## 🔧 CI/CD Configuration

### GitHub Secrets Setup

Configure the following secrets in your GitHub repository (Settings → Secrets and variables → Actions):

| Secret Name | Description | Example |
|------------|-------------|---------|
| `ALIBABA_HOST` | Your Alibaba Cloud instance IP | `123.456.789.012` |
| `SSH_USERNAME` | SSH username for deployment | `root` or `ubuntu` |
| `SSH_PRIVATE_KEY` | Private SSH key for authentication | `-----BEGIN RSA PRIVATE KEY-----...` |
| `SSH_PORT` | SSH port (optional, defaults to 22) | `22` |

**Note:** The workflow uses `GITHUB_TOKEN` automatically for GitHub Container Registry authentication.

### Generate SSH Key

```bash
# On your local machine
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github_actions

# Copy public key to Alibaba Cloud
ssh-copy-id -i ~/.ssh/github_actions.pub user@your-alibaba-ip

# Copy private key content to GitHub Secret
cat ~/.ssh/github_actions
```

## 🏗️ CI/CD Pipeline Breakdown

The GitHub Actions workflow (`.github/workflows/deploy.yml`) consists of three jobs:

### 1. Build & Test
- Checks out code
- Sets up Node.js environment
- Installs dependencies
- Runs health check test
- **Trigger:** Push to `master`

### 2. Build & Push Docker Image
- Sets up Docker Buildx
- Logs into GitHub Container Registry
- Generates metadata (tags, labels)
- Builds multi-stage Docker image
- Pushes image with tags:
  - `master-{short-sha}` (e.g., `master-a1b2c3d`)
  - `latest` (for master branch)
- Uses layer caching for faster builds

### 3. Deploy to Alibaba Cloud
- SSHs into Alibaba Cloud instance
- Updates `.env` with new image tag
- Pulls latest Docker image
- Restarts container with `docker compose`
- Waits for health check
- Cleans up old images
- Shows deployment status

## 🖥️ Server Setup (Alibaba Cloud)

### Initial Server Configuration

1. **Install Docker and Docker Compose**
   ```bash
   # For Ubuntu
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh get-docker.sh
   sudo usermod -aG docker $USER
   
   # Install Docker Compose
   sudo apt-get update
   sudo apt-get install docker-compose-plugin
   ```

2. **Create deployment directory**
   ```bash
   sudo mkdir -p /opt/sandbox-app
   sudo chown $USER:$USER /opt/sandbox-app
   cd /opt/sandbox-app
   ```

3. **Copy docker-compose.yml and .env**
   ```bash
   # Create docker-compose.yml
   vim docker-compose.yml
   # Paste content from repository
   
   # Create .env
   vim .env
   # Paste content from .env.example and update values
   ```

4. **Optional: Enable auto-start on boot**
   ```bash
   # Create systemd service
   sudo vim /etc/systemd/system/sandbox-app.service
   ```
   
   Add the following content:
   ```ini
   [Unit]
   Description=Sandbox CI/CD Application
   Requires=docker.service
   After=docker.service
   
   [Service]
   Type=oneshot
   RemainAfterExit=yes
   WorkingDirectory=/opt/sandbox-app
   ExecStart=/usr/bin/docker compose up -d
   ExecStop=/usr/bin/docker compose down
   
   [Install]
   WantedBy=multi-user.target
   ```
   
   Enable the service:
   ```bash
   sudo systemctl enable sandbox-app.service
   sudo systemctl start sandbox-app.service
   ```

## 🐳 Docker Image Details

### Multi-stage Build
- **Builder stage:** Installs production dependencies
- **Production stage:** 
  - Based on Node.js 18 Alpine (lightweight)
  - Runs as non-root user (`nodejs`)
  - Includes health check
  - Optimized layer caching

### Image Size
- Approximately 150-200MB (thanks to Alpine and multi-stage build)

### Security Features
- Non-root user execution
- Minimal attack surface (Alpine)
- No unnecessary packages
- Read-only filesystem compatible

## 📊 Monitoring & Debugging

### View Logs
```bash
# Real-time logs
docker compose logs -f web

# Last 100 lines
docker compose logs --tail 100 web
```

### Check Container Health
```bash
docker inspect sandbox-cicd-app | grep -A 10 Health
```

### Manual Health Check
```bash
curl http://localhost:80/health
```

### Restart Container
```bash
docker compose restart web
```

## 🚢 Migration Path to Kubernetes

This project is designed for easy migration to Kubernetes:

### Current State (Docker Compose)
- Single-container deployment
- Docker Compose orchestration
- Manual scaling

### Future State (Kubernetes)
- Multi-replica deployments
- Auto-scaling with HPA
- Rolling updates
- Service mesh ready

### Migration Steps

1. **Create Kubernetes manifests**
   ```yaml
   # deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: sandbox-app
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: sandbox-app
     template:
       metadata:
         labels:
           app: sandbox-app
       spec:
         containers:
         - name: web
           image: ghcr.io/YOUR_USERNAME/sandbox-cicd-app:latest
           ports:
           - containerPort: 3000
           livenessProbe:
             httpGet:
               path: /health
               port: 3000
           readinessProbe:
             httpGet:
               path: /health
               port: 3000
   ```

2. **Add Service and Ingress**
   ```yaml
   # service.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: sandbox-app
   spec:
     selector:
       app: sandbox-app
     ports:
     - port: 80
       targetPort: 3000
   ```

3. **Update CI/CD pipeline**
   - Replace `docker compose` with `kubectl apply`
   - Add Helm charts for complex deployments
   - Implement GitOps with ArgoCD/Flux

## 📁 Project Structure

```
sandbox-cicd-app/
├── .github/
│   └── workflows/
│       └── deploy.yml          # CI/CD pipeline
├── src/
│   └── server.js               # Express server
├── public/
│   └── index.html              # Static HTML bio page
├── .dockerignore               # Docker build exclusions
├── .env.example                # Environment template
├── .gitignore                  # Git exclusions
├── Dockerfile                  # Multi-stage Docker build
├── docker-compose.yml          # Container orchestration
├── package.json                # Node.js dependencies
└── README.md                   # This file
```

## 🔐 Security Best Practices

### Container Security
- ✅ Non-root user execution
- ✅ Minimal base image (Alpine)
- ✅ Multi-stage builds (no build tools in production)
- ✅ Health checks enabled
- ✅ Resource limits (configurable in Compose)

### Secrets Management
- ✅ GitHub Secrets for sensitive data
- ✅ Environment variables (never commit `.env`)
- ✅ SSH key authentication
- ⚠️ Consider using secret managers (AWS Secrets Manager, HashiCorp Vault) for production

### Network Security
- Configure firewall rules on Alibaba Cloud
- Use HTTPS with reverse proxy (Nginx/Caddy)
- Implement rate limiting
- Enable Docker network isolation

## 🧪 Testing

### Manual Testing
```bash
# Run the app
npm start

# Test health endpoint
curl http://localhost:3000/health

# Test main page
curl http://localhost:3000
```

### Automated Testing (in CI)
The pipeline automatically:
1. Installs dependencies
2. Starts the server
3. Performs health check
4. Validates HTTP 200 response

### Adding More Tests
```bash
# Install testing framework
npm install --save-dev jest supertest

# Create test file: src/server.test.js
# Update package.json scripts
"test": "jest --coverage"
```

## 🐛 Troubleshooting

### Common Issues

**Issue: Container won't start**
```bash
# Check logs
docker compose logs web

# Inspect container
docker inspect sandbox-cicd-app

# Check port conflicts
sudo netstat -tlnp | grep :80
```

**Issue: Deployment fails in CI/CD**
```bash
# SSH into server manually
ssh user@your-alibaba-ip

# Check Docker status
docker ps -a
docker compose ps

# Check disk space
df -h

# Check Docker logs
journalctl -u docker.service
```

**Issue: Can't pull image from registry**
```bash
# Login manually
echo "YOUR_GITHUB_TOKEN" | docker login ghcr.io -u YOUR_USERNAME --password-stdin

# Check image exists
docker manifest inspect ghcr.io/YOUR_USERNAME/sandbox-cicd-app:latest
```

**Issue: Health check failing**
```bash
# Check if app is responding
docker exec sandbox-cicd-app curl -f http://localhost:3000/health

# Check container logs
docker logs sandbox-cicd-app

# Restart container
docker compose restart web
```

## 📈 Performance Optimization

### Docker Image
- Using Alpine (smaller size)
- Multi-stage builds (no dev dependencies)
- Layer caching in CI/CD
- `.dockerignore` to exclude unnecessary files

### Application
- Production Node.js optimizations
- Graceful shutdown handling
- Health check endpoint
- Proper logging

### CI/CD
- Docker layer caching with GitHub Actions
- Parallel job execution
- Conditional deployments

## 🔄 Updating the Application

### Making Changes

1. **Update code**
   ```bash
   git checkout -b feature/update-bio
   # Make your changes
   git add .
   git commit -m "feat: update bio information"
   git push origin feature/update-bio
   ```

2. **Create PR and merge to master**
   - Automatic deployment will trigger

3. **Monitor deployment**
   - Check GitHub Actions workflow
   - Verify deployment on server
   - Test the live application

### Rollback Strategy

If deployment fails or issues occur:

```bash
# SSH into server
ssh user@your-alibaba-ip
cd /opt/sandbox-app

# Check previous image tags
docker images | grep sandbox-cicd-app

# Update .env to previous tag
vim .env
# Change IMAGE_TAG to previous version

# Restart with previous version
docker compose up -d web
```

## 📚 Learning Resources

### Docker & Containers
- [Docker Documentation](https://docs.docker.com/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Security](https://docs.docker.com/engine/security/)

### CI/CD
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Trunk-Based Development](https://trunkbaseddevelopment.com/)

### Kubernetes Migration
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kompose (Compose to K8s)](https://kompose.io/)
- [Helm Charts](https://helm.sh/)

## 🤝 Contributing

This is a personal learning sandbox, but contributions are welcome!

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'feat: add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📝 License

This project is licensed under the MIT License - see the LICENSE file for details.

## 🎓 What You'll Learn

By working with this project, you'll gain hands-on experience with:

- ✅ **Docker**: Building efficient, secure container images
- ✅ **Docker Compose**: Orchestrating multi-container applications
- ✅ **CI/CD**: Automating builds, tests, and deployments
- ✅ **GitHub Actions**: Creating workflow pipelines
- ✅ **Cloud Deployment**: Deploying to production servers
- ✅ **DevOps Practices**: Trunk-based development, GitOps principles
- ✅ **Container Registry**: Working with GitHub Container Registry
- ✅ **SSH Deployment**: Secure remote deployments
- ✅ **Health Checks**: Implementing application monitoring
- ✅ **Infrastructure as Code**: Managing infrastructure with code

## 🚀 Next Steps

After mastering this setup, consider:

1. **Add a database** (PostgreSQL/MongoDB with Docker Compose)
2. **Implement monitoring** (Prometheus + Grafana)
3. **Add logging** (ELK stack or Loki)
4. **Set up staging environment**
5. **Migrate to Kubernetes** (EKS, AKS, or Alibaba ACK)
6. **Implement blue-green deployments**
7. **Add automated testing** (integration tests, e2e tests)
8. **Set up Terraform** for infrastructure provisioning

## 📞 Support

For issues or questions:
- Open an issue on GitHub
- Check the troubleshooting section
- Review GitHub Actions logs

---

**Happy Learning! 🎉**

Built with ❤️ for learning CI/CD and DevOps practices.
