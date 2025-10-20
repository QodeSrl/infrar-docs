# Greenfield: Deploy New Projects with Infrar

## Overview

This guide shows how to deploy a brand new application using Infrar. "Greenfield" means starting from scratch - you'll build your application with Infrar from day one, ensuring maximum portability and cost optimization.

## Prerequisites

- GitHub account
- Docker knowledge (basic)
- Python 3.8+ (for MVP) or Node.js 16+ (Phase 2)
- Cloud provider account (AWS, GCP, or Azure)

## Quick Start (10 minutes)

### Step 1: Create Your Application

Create a simple Python web application:

```python
# app.py
from flask import Flask, request, jsonify
from infrar.storage import upload, list_objects
import os

app = Flask(__name__)

@app.route('/')
def home():
    return 'Hello from Infrar!'

@app.route('/upload', methods=['POST'])
def upload_file():
    """Upload a file to storage."""
    file = request.files['file']
    filename = file.filename

    # Save locally first
    local_path = f'/tmp/{filename}'
    file.save(local_path)

    # Upload to cloud storage
    upload(
        bucket='my-app-uploads',
        source=local_path,
        destination=f'uploads/{filename}'
    )

    os.remove(local_path)

    return jsonify({'message': 'File uploaded', 'filename': filename})

@app.route('/files', methods=['GET'])
def list_files():
    """List all uploaded files."""
    objects = list_objects(bucket='my-app-uploads', prefix='uploads/')
    files = [{'name': obj.key, 'size': obj.size} for obj in objects]
    return jsonify({'files': files})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

Create `requirements.txt`:
```
flask==3.0.0
infrar-storage==1.0.0
```

Create `Dockerfile`:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8080

CMD ["python", "app.py"]
```

### Step 2: Push to GitHub

```bash
# Initialize git repository
git init
git add .
git commit -m "Initial commit"

# Create GitHub repository (via UI or gh CLI)
gh repo create my-app --public --source=. --push
```

### Step 3: Sign Up for Infrar

1. Go to https://app.infrar.io
2. Click "Sign Up"
3. Authenticate with GitHub

### Step 4: Create Project in Infrar

1. Click "New Project"
2. Fill in details:
   - **Name**: My App
   - **GitHub Repo**: your-username/my-app
   - **Dockerfile Path**: ./Dockerfile (default)

3. Click "Continue to Provider Selection"

### Step 5: Select Cloud Provider

Infrar will show cost comparison:

```
Cost Comparison for Your Configuration:
├─ AWS (us-east-1)
│  ├─ ECS Fargate: $45/month
│  ├─ S3 Storage: $5/month
│  ├─ Data Transfer: $10/month
│  └─ Total: $60/month
│
├─ GCP (us-central1) ⭐ RECOMMENDED
│  ├─ Cloud Run: $32/month
│  ├─ Cloud Storage: $5/month
│  ├─ Data Transfer: $12/month
│  └─ Total: $49/month (18% savings)
│
└─ Azure (eastus)
   ├─ Container Apps: $38/month
   ├─ Blob Storage: $5/month
   ├─ Data Transfer: $9/month
   └─ Total: $52/month
```

Select **GCP** (cheapest option).

### Step 6: Configure Resources

Specify your resource requirements:

- **CPU**: 1 vCPU (or use slider)
- **Memory**: 2048 MB
- **Region**: us-central1
- **Min Instances**: 0 (scale to zero)
- **Max Instances**: 10

Infrar calculates updated cost: **$49/month**

### Step 7: Add Cloud Credentials

#### For GCP:

1. Go to GCP Console: https://console.cloud.google.com
2. Create service account:
   - IAM & Admin → Service Accounts → Create
   - Name: `infrar-deployer`
   - Grant roles:
     - Cloud Run Admin
     - Storage Admin
     - Artifact Registry Administrator
   - Create key (JSON)
   - Download `infrar-key.json`

3. In Infrar UI:
   - Click "Add GCP Credentials"
   - Upload `infrar-key.json`
   - Click "Validate" (Infrar tests permissions)

### Step 8: Deploy!

1. Click "Deploy"
2. Watch the magic happen:

```
[1/7] Cloning repository... ✓
[2/7] Analyzing code... ✓
      Found 3 infrar SDK calls
[3/7] Transforming code for GCP... ✓
      infrar.storage.upload → google.cloud.storage
      infrar.storage.list_objects → google.cloud.storage
[4/7] Building Docker image... ✓
      Image: gcr.io/project/my-app:abc123
[5/7] Pushing to Artifact Registry... ✓
[6/7] Provisioning infrastructure... ✓
      Cloud Run service: my-app
      Cloud Storage bucket: my-app-uploads
[7/7] Deploying application... ✓

🎉 Deployment successful!

Your application is live at:
https://my-app-abc123-uc.a.run.app

Infrar transformed your code:
- infrar.storage → google.cloud.storage
- Zero runtime overhead
- Full GCP performance
```

### Step 9: Test Your Application

```bash
# Test home page
curl https://my-app-abc123-uc.a.run.app/
# Output: Hello from Infrar!

# Upload a file
curl -X POST -F "file=@test.txt" \
  https://my-app-abc123-uc.a.run.app/upload
# Output: {"message":"File uploaded","filename":"test.txt"}

# List files
curl https://my-app-abc123-uc.a.run.app/files
# Output: {"files":[{"name":"uploads/test.txt","size":1234}]}
```

**Congratulations!** Your app is running on GCP, using native GCP SDKs, with zero runtime overhead.

## What Just Happened?

### Code Transformation

Your original code:
```python
from infrar.storage import upload
upload(bucket='my-app-uploads', source=local_path, destination=f'uploads/{filename}')
```

Was transformed to GCP-specific code:
```python
from google.cloud import storage

storage_client = storage.Client()
bucket = storage_client.bucket('my-app-uploads')
blob = bucket.blob(f'uploads/{filename}')
blob.upload_from_filename(local_path)
```

**No runtime abstraction layer** - just pure GCP SDK calls.

### Infrastructure Provisioning

Infrar created:
1. **Cloud Run Service**: Runs your container
2. **Cloud Storage Bucket**: Stores uploaded files
3. **IAM Bindings**: Proper permissions for Cloud Run to access Storage
4. **Load Balancer**: HTTPS endpoint (automatic with Cloud Run)

All configured with best practices.

## Next Steps

### Add More Features

#### 1. Add Database

```python
# app.py
from infrar.database import query, execute

@app.route('/users', methods=['GET'])
def get_users():
    users = query(database='mydb', sql='SELECT * FROM users')
    return jsonify({'users': users})

@app.route('/users', methods=['POST'])
def create_user():
    data = request.json
    execute(
        database='mydb',
        sql='INSERT INTO users (name, email) VALUES (?, ?)',
        params=[data['name'], data['email']]
    )
    return jsonify({'message': 'User created'})
```

Update `requirements.txt`:
```
flask==3.0.0
infrar-storage==1.0.0
infrar-database==1.0.0  # Add database plugin
```

Deploy again - Infrar will:
- Transform database calls to GCP Cloud SQL
- Provision Cloud SQL instance
- Configure networking

#### 2. Add Background Jobs

```python
# worker.py
from infrar.messaging import subscribe

def process_job(message):
    print(f"Processing job: {message}")
    # Do work...

# Subscribe to job queue
subscribe(queue='jobs', handler=process_job)
```

Deploy as separate service - Infrar will:
- Create separate Cloud Run Job (or Cloud Functions)
- Set up Pub/Sub topic
- Configure subscriptions

### Switch Cloud Providers

Realized GCP isn't optimal for your workload? Switch to AWS:

1. Go to Project → Migrate
2. Select AWS, us-east-1
3. Infrar shows updated cost: **$60/month** (20% more expensive)
4. Click "Migrate"

Infrar will:
- Retrieve your original infrar SDK code
- Transform for AWS (boto3 instead of google-cloud-storage)
- Build new Docker image
- Provision AWS infrastructure (ECS, S3)
- Deploy in parallel (no downtime)
- Switch DNS
- Decommission GCP resources

**Your code doesn't change** - same GitHub repo, different deployment.

## Best Practices

### 1. Use Environment Variables

```python
# Good
UPLOAD_BUCKET = os.environ.get('UPLOAD_BUCKET', 'my-app-uploads')
upload(bucket=UPLOAD_BUCKET, source=file, destination=path)

# Bad - hardcoded
upload(bucket='my-app-uploads', source=file, destination=path)
```

Configure in Infrar UI:
- Project Settings → Environment Variables
- Add: `UPLOAD_BUCKET=my-app-uploads`

### 2. Structure Your Code

```
my-app/
├── app/
│   ├── __init__.py
│   ├── routes.py      # Web routes
│   ├── storage.py     # Storage operations (using infrar.storage)
│   └── database.py    # Database operations (using infrar.database)
├── tests/
│   ├── test_routes.py
│   └── test_storage.py
├── Dockerfile
├── requirements.txt
└── README.md
```

### 3. Test Locally

```bash
# Set provider for local development
export INFRAR_PROVIDER=gcp
export GOOGLE_APPLICATION_CREDENTIALS=~/infrar-key.json

# Install dependencies
pip install -r requirements.txt

# Run locally
python app.py

# Infrar SDK works with real GCP
# Upload actually goes to Cloud Storage
```

### 4. Use Docker Multi-Stage Builds

```dockerfile
# Build stage
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Runtime stage
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY app.py .
ENV PATH=/root/.local/bin:$PATH
EXPOSE 8080
CMD ["python", "app.py"]
```

Smaller images = faster deployments.

### 5. Handle Errors

```python
from infrar.storage import upload
from infrar.storage.exceptions import StorageError
import logging

@app.route('/upload', methods=['POST'])
def upload_file():
    try:
        # ... upload logic
        upload(bucket='uploads', source=local_path, destination=dest_path)
        return jsonify({'message': 'Success'})

    except StorageError as e:
        logging.error(f"Upload failed: {e}")
        return jsonify({'error': 'Upload failed'}), 500

    finally:
        # Clean up temp files
        if os.path.exists(local_path):
            os.remove(local_path)
```

## Monitoring & Debugging

### View Logs

In Infrar UI:
- Project → Deployments → Latest → Logs
- Real-time log streaming
- Filter by level (INFO, ERROR, etc.)

### View Metrics

Dashboard shows:
- Request rate
- Response time (p50, p95, p99)
- Error rate
- Resource usage (CPU, memory)
- Costs (updated daily)

### Debug Issues

If deployment fails:

1. **Check Transformation Logs**:
   - Did code transformation succeed?
   - Any unsupported infrar calls?

2. **Check Build Logs**:
   - Docker build errors?
   - Missing dependencies?

3. **Check Deployment Logs**:
   - Infrastructure provisioning errors?
   - Permission issues?

4. **Check Application Logs**:
   - Runtime errors?
   - Storage/database connection issues?

## Common Issues

### "Transformation failed: Unsupported infrar call"

**Problem**: Used an infrar function not yet implemented.

**Solution**: Check [infrar-storage API docs](../api/infrar-storage.md) for supported functions.

### "Permission denied: Cannot create Cloud Run service"

**Problem**: Service account lacks necessary permissions.

**Solution**: Grant Cloud Run Admin role to service account.

### "Application not responding to health checks"

**Problem**: Application not listening on correct port.

**Solution**: Ensure app listens on port specified in Dockerfile EXPOSE (default: 8080).

```python
if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
```

## Cost Optimization

### Monitor Costs

Infrar Dashboard shows:
- Daily cost breakdown
- Cost by capability (compute, storage, networking)
- Cost trends
- Recommendations

### Optimize Settings

**Reduce costs by**:
1. **Scale to zero**: Set min instances = 0 (GCP Cloud Run, Azure Container Apps)
2. **Right-size**: Reduce CPU/memory if underutilized
3. **Use cheaper storage class**: Standard → Nearline (for infrequently accessed data)
4. **Enable compression**: Reduce data transfer costs

### Compare Providers

Click "Cost Comparison" to see:
- Current cost: GCP $49/month
- If on AWS: $60/month (22% more)
- If on Azure: $52/month (6% more)

Consider migrating if workload changes.

## Conclusion

Greenfield development with Infrar gives you:
- ✅ **Provider flexibility** from day one
- ✅ **Cost optimization** with automatic comparisons
- ✅ **Best practices** applied automatically
- ✅ **No vendor lock-in** - switch anytime

Next steps:
- Read [Brownfield Guide](brownfield.md) for migrating existing apps
- Read [Multi-Cloud Guide](multi-cloud.md) for running across multiple providers
- Explore [API Reference](../api/infrar-storage.md) for all available functions

Happy building! 🚀
