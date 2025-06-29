# Azure Deployment Guide for Open WebUI

This guide walks you through deploying your Open WebUI fork on Azure with full CI/CD integration for syncing with the upstream repository.

## Architecture Overview

**Recommended Azure Services:**
- **Azure Container Apps** - Docker-based app hosting with WebSocket support
- **Azure Database for PostgreSQL** - Managed database service
- **Azure Storage Account** - File uploads and persistent data storage
- **Azure Container Registry (ACR)** - Custom Docker images
- **GitHub Actions** - CI/CD pipeline for automated deployments

## Prerequisites

- Azure CLI installed and logged in
- GitHub repository with your Open WebUI fork
- Docker installed locally (for testing)
- OpenSSL for generating secrets

## Step 1: Infrastructure Setup

### 1.1 Create Resource Group
```bash
az group create --name rg-openwebui --location eastus
```

### 1.2 Create Azure Container Registry
```bash
az acr create \
  --resource-group rg-openwebui \
  --name acrOpenWebUI \
  --sku Basic \
  --admin-enabled true
```

### 1.3 Create PostgreSQL Database
```bash
az postgres flexible-server create \
  --resource-group rg-openwebui \
  --name pg-openwebui \
  --location eastus \
  --admin-user openwebuiadmin \
  --admin-password "YourSecurePassword123!" \
  --version 15 \
  --sku-name Standard_B1ms \
  --storage-size 32 \
  --public-access 0.0.0.0
```

### 1.4 Create Database for Open WebUI
```bash
az postgres flexible-server db create \
  --resource-group rg-openwebui \
  --server-name pg-openwebui \
  --database-name openwebui
```

### 1.5 Create Storage Account
```bash
az storage account create \
  --name storageopenwebui \
  --resource-group rg-openwebui \
  --location eastus \
  --sku Standard_LRS
```

### 1.6 Create Container Apps Environment
```bash
az containerapp env create \
  --name env-openwebui \
  --resource-group rg-openwebui \
  --location eastus
```

## Step 2: GitHub Repository Configuration

### 2.1 Fork and Upstream Setup
Since you already have a fork, add the upstream remote:
```bash
git remote add upstream https://github.com/open-webui/open-webui.git
git fetch upstream
```

### 2.2 Create Service Principal for GitHub Actions
```bash
az ad sp create-for-rbac \
  --name sp-openwebui \
  --role contributor \
  --scopes /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/rg-openwebui \
  --sdk-auth
```

**Save the output JSON - you'll need it for GitHub secrets.**

### 2.3 Get Container Registry Credentials
```bash
# Get ACR username
az acr credential show --name acrOpenWebUI --query username -o tsv

# Get ACR password
az acr credential show --name acrOpenWebUI --query passwords[0].value -o tsv
```

### 2.4 Get Database Connection String
```bash
echo "postgresql://openwebuiadmin:YourSecurePassword123!@pg-openwebui.postgres.database.azure.com/openwebui?sslmode=require"
```

### 2.5 Get Storage Connection String
```bash
az storage account show-connection-string \
  --name storageopenwebui \
  --resource-group rg-openwebui \
  --query connectionString -o tsv
```

### 2.6 GitHub Secrets Configuration
Add these secrets to your GitHub repository settings (Settings > Secrets and variables > Actions):

| Secret Name | Value | Description |
|-------------|-------|-------------|
| `AZURE_CREDENTIALS` | Service principal JSON from step 2.2 | Azure authentication |
| `ACR_USERNAME` | ACR username from step 2.3 | Container registry login |
| `ACR_PASSWORD` | ACR password from step 2.3 | Container registry password |
| `DATABASE_URL` | Connection string from step 2.4 | PostgreSQL database |
| `STORAGE_CONNECTION_STRING` | Connection string from step 2.5 | Azure Storage |

## Step 3: CI/CD Pipeline Setup

### 3.1 Create GitHub Workflow
Create `.github/workflows/azure-deploy.yml`:

```yaml
name: Deploy to Azure Container Apps

on:
  push:
    branches: [ main ]
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * *'  # Daily sync check at 2 AM UTC

env:
  REGISTRY_NAME: acropenwebui.azurecr.io
  IMAGE_NAME: open-webui
  CONTAINER_APP_NAME: ca-openwebui
  RESOURCE_GROUP: rg-openwebui
  CONTAINER_APP_ENV: env-openwebui

jobs:
  sync-upstream:
    runs-on: ubuntu-latest
    outputs:
      has-changes: ${{ steps.sync.outputs.has-changes }}
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0
      
      - name: Configure Git
        run: |
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'
      
      - name: Sync upstream changes
        id: sync
        run: |
          git remote add upstream https://github.com/open-webui/open-webui.git
          git fetch upstream
          
          # Check if there are new commits
          BEHIND=$(git rev-list --count HEAD..upstream/main)
          echo "Commits behind upstream: $BEHIND"
          
          if [ "$BEHIND" -gt 0 ]; then
            echo "has-changes=true" >> $GITHUB_OUTPUT
            git merge upstream/main --no-edit
            git push origin main
          else
            echo "has-changes=false" >> $GITHUB_OUTPUT
            echo "Already up to date with upstream"
          fi

  build-and-deploy:
    needs: sync-upstream
    if: needs.sync-upstream.outputs.has-changes == 'true' || github.event_name == 'workflow_dispatch' || github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: main  # Use updated main branch after sync
      
      - name: Log in to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Log in to Container Registry
        run: |
          echo ${{ secrets.ACR_PASSWORD }} | docker login ${{ env.REGISTRY_NAME }} -u ${{ secrets.ACR_USERNAME }} --password-stdin
      
      - name: Build and push Docker image
        run: |
          docker build -t ${{ env.REGISTRY_NAME }}/${{ env.IMAGE_NAME }}:${{ github.sha }} .
          docker tag ${{ env.REGISTRY_NAME }}/${{ env.IMAGE_NAME }}:${{ github.sha }} ${{ env.REGISTRY_NAME }}/${{ env.IMAGE_NAME }}:latest
          docker push ${{ env.REGISTRY_NAME }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          docker push ${{ env.REGISTRY_NAME }}/${{ env.IMAGE_NAME }}:latest
      
      - name: Deploy to Container Apps
        run: |
          az containerapp update \
            --name ${{ env.CONTAINER_APP_NAME }} \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --image ${{ env.REGISTRY_NAME }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

## Step 4: Deploy Container App

### 4.1 Generate Secret Key
```bash
WEBUI_SECRET_KEY=$(openssl rand -base64 32)
echo $WEBUI_SECRET_KEY
```

### 4.2 Create Initial Container App
```bash
az containerapp create \
  --name ca-openwebui \
  --resource-group rg-openwebui \
  --environment env-openwebui \
  --image acrOpenWebUI.azurecr.io/open-webui:latest \
  --target-port 8080 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 3 \
  --cpu 1.0 \
  --memory 2Gi \
  --registry-server acrOpenWebUI.azurecr.io \
  --registry-username $(az acr credential show --name acrOpenWebUI --query username -o tsv) \
  --registry-password $(az acr credential show --name acrOpenWebUI --query passwords[0].value -o tsv) \
  --env-vars \
    "DATABASE_URL=postgresql://openwebuiadmin:YourSecurePassword123!@pg-openwebui.postgres.database.azure.com/openwebui?sslmode=require" \
    "WEBUI_SECRET_KEY=$WEBUI_SECRET_KEY" \
    "SCARF_NO_ANALYTICS=true" \
    "DO_NOT_TRACK=true" \
    "ANONYMIZED_TELEMETRY=false" \
    "DATA_DIR=/app/backend/data" \
    "CORS_ALLOW_ORIGIN=*"
```

### 4.3 Get Application URL
```bash
az containerapp show \
  --name ca-openwebui \
  --resource-group rg-openwebui \
  --query properties.configuration.ingress.fqdn \
  -o tsv
```

## Step 5: Initial Deployment

### 5.1 Build and Push Initial Image
```bash
# Log in to ACR
az acr login --name acrOpenWebUI

# Build and push image
docker build -t acrOpenWebUI.azurecr.io/open-webui:latest .
docker push acrOpenWebUI.azurecr.io/open-webui:latest
```

### 5.2 Update Container App with New Image
```bash
az containerapp update \
  --name ca-openwebui \
  --resource-group rg-openwebui \
  --image acrOpenWebUI.azurecr.io/open-webui:latest
```

## Step 6: Verification and Testing

### 6.1 Check Deployment Status
```bash
az containerapp show \
  --name ca-openwebui \
  --resource-group rg-openwebui \
  --query properties.runningStatus
```

### 6.2 View Logs
```bash
az containerapp logs show \
  --name ca-openwebui \
  --resource-group rg-openwebui \
  --follow
```

### 6.3 Test Application
Visit the URL from step 4.3 to access your deployed Open WebUI application.

## Upstream Sync Strategies

### Automatic Sync (Recommended)
The GitHub workflow automatically:
- Checks for upstream changes daily
- Merges changes automatically
- Triggers deployment if there are updates
- Handles basic merge conflicts

### Manual Sync Process
```bash
# Fetch upstream changes
git fetch upstream

# Create sync branch for review
git checkout -b sync-upstream
git merge upstream/main

# Review changes, resolve conflicts if needed
git push origin sync-upstream

# Create PR for review, then merge to main
```

### Handling Merge Conflicts
If automatic sync fails due to conflicts:
1. Check GitHub Actions logs
2. Manually sync using the process above
3. Resolve conflicts in your preferred editor
4. Push resolved changes to trigger deployment

## Configuration Management

### Environment Variables
Key environment variables for production:

```bash
# Required
DATABASE_URL="postgresql://user:pass@host/db?sslmode=require"
WEBUI_SECRET_KEY="your-secret-key"

# Optional
AZURE_STORAGE_CONNECTION_STRING="storage-connection-string"
OLLAMA_BASE_URL="http://your-ollama-instance:11434"
OPENAI_API_KEY="your-openai-key"
CORS_ALLOW_ORIGIN="*"

# Telemetry (recommended to disable)
SCARF_NO_ANALYTICS=true
DO_NOT_TRACK=true
ANONYMIZED_TELEMETRY=false
```

### Scaling Configuration
```bash
# Update scaling settings
az containerapp update \
  --name ca-openwebui \
  --resource-group rg-openwebui \
  --min-replicas 1 \
  --max-replicas 5 \
  --cpu 2.0 \
  --memory 4Gi
```

## Monitoring and Maintenance

### Cost Monitoring
```bash
# Check current usage and costs
az consumption usage list \
  --start-date 2024-01-01 \
  --end-date 2024-01-31 \
  --scope /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/rg-openwebui
```

### Health Checks
```bash
# Check container app health
az containerapp show \
  --name ca-openwebui \
  --resource-group rg-openwebui \
  --query properties.runningStatus

# Check database connectivity
az postgres flexible-server show \
  --name pg-openwebui \
  --resource-group rg-openwebui \
  --query state
```

### Backup Strategy
1. **Database**: Enable automated backups on PostgreSQL
2. **Storage**: Use Azure Storage redundancy
3. **Configuration**: Keep environment variables in Azure Key Vault
4. **Code**: Your fork serves as the code backup

## Troubleshooting

### Common Issues

**Container fails to start:**
```bash
# Check logs
az containerapp logs show --name ca-openwebui --resource-group rg-openwebui --tail 100

# Check revision status
az containerapp revision list --name ca-openwebui --resource-group rg-openwebui
```

**Database connection issues:**
```bash
# Test database connectivity
az postgres flexible-server connect \
  --name pg-openwebui \
  --admin-user openwebuiadmin \
  --admin-password "YourSecurePassword123!" \
  --database-name openwebui
```

**Sync conflicts:**
1. Check GitHub Actions logs
2. Manually resolve conflicts locally
3. Push resolved changes

### Performance Optimization
- Monitor CPU/Memory usage in Azure Portal
- Adjust replica counts based on traffic
- Consider using Azure CDN for static assets
- Enable database query optimization

## Security Considerations

1. **Use Azure Key Vault** for sensitive configurations
2. **Enable Private Endpoints** for database connections
3. **Configure Network Security Groups** for additional protection
4. **Regular Security Updates** via automated syncing
5. **Monitor Access Logs** in Azure Portal

## Cost Optimization

- Use Azure Cost Management to track spending
- Consider Reserved Instances for predictable workloads
- Implement auto-scaling to reduce idle costs
- Regular cleanup of unused resources

## Next Steps

1. Set up custom domain with SSL certificate
2. Configure Azure CDN for better performance
3. Implement Azure Key Vault for secrets management
4. Set up monitoring and alerting
5. Configure backup and disaster recovery procedures

---

**Need Help?**
- Azure documentation: https://docs.microsoft.com/azure/
- Open WebUI documentation: https://docs.openwebui.com/
- GitHub Actions documentation: https://docs.github.com/actions/ 