# Infrar MVP - Testing from Scratch

**Purpose**: Complete guide for testing the Infrar MVP from a clean state
**Target**: Fresh installation using Test PyPI
**Duration**: ~30 minutes

---

## 🎯 What We're Testing

This guide proves the complete MVP workflow:

1. ✅ Install Infrar SDK from Test PyPI
2. ✅ Write cloud-agnostic code
3. ✅ Transform to AWS (boto3)
4. ✅ Transform to GCP (google-cloud-storage)
5. ✅ Verify zero runtime overhead

---

## 📋 Prerequisites

### Required
- Python 3.8 or higher
- Go 1.21 or higher (for transformation engine)
- Git
- Internet connection

### Verify Prerequisites
```bash
python3 --version  # Should be 3.8+
go version         # Should be 1.21+
git --version      # Any recent version
```

---

## 🚀 Step 1: Install Infrar SDK

### Option A: From Test PyPI (Recommended for testing)

```bash
# Create fresh virtual environment
python3 -m venv infrar-test-env
source infrar-test-env/bin/activate  # On Windows: infrar-test-env\Scripts\activate

# Install from Test PyPI
pip install -i https://test.pypi.org/simple/ infrar

# Verify installation
python3 -c "import infrar; print(f'Infrar SDK v{infrar.__version__} installed!')"
```

### Option B: From Source (If not yet published to Test PyPI)

```bash
# Clone and install
git clone https://github.com/QodeSrl/infrar-sdk-python.git
cd infrar-sdk-python
pip install -e .

# Verify
python3 -c "import infrar; print(f'Infrar SDK v{infrar.__version__} installed!')"
cd ..
```

---

## 🔧 Step 2: Install Transformation Engine

```bash
# Clone engine repository
git clone https://github.com/QodeSrl/infrar-engine.git
cd infrar-engine

# Build the transformation tool
go build -o bin/transform ./cmd/transform

# Verify build
./bin/transform --help

cd ..
```

---

## 📦 Step 3: Get Transformation Rules

```bash
# Clone plugins repository
git clone https://github.com/QodeSrl/infrar-plugins.git

# Verify plugins exist
ls infrar-plugins/packages/storage/aws/rules.yaml
ls infrar-plugins/packages/storage/gcp/rules.yaml
```

Your directory structure should now look like:
```
./
├── infrar-engine/
│   └── bin/transform
├── infrar-plugins/
│   └── packages/storage/{aws,gcp}/rules.yaml
└── infrar-test-env/ (if using virtualenv)
```

---

## ✍️ Step 4: Write Your First Cloud-Agnostic App

Create a simple data processing app:

```bash
cat > my_app.py << 'EOF'
from infrar.storage import upload, download, delete

def data_pipeline():
    """Process data using cloud-agnostic storage operations."""

    # Upload raw data
    upload(
        bucket='analytics-data',
        source='raw_data.csv',
        destination='data/raw/input.csv'
    )

    # Download processed results
    download(
        bucket='analytics-data',
        source='data/processed/results.csv',
        destination='output_results.csv'
    )

    # Clean up temp files
    delete(
        bucket='analytics-data',
        path='data/temp/staging.csv'
    )

    print("Pipeline completed successfully!")

if __name__ == '__main__':
    data_pipeline()
EOF
```

**Verify the code is valid Python**:
```bash
python3 -m py_compile my_app.py
python3 -c "from my_app import data_pipeline; print('✅ App imports successfully')"
```

---

## 🔄 Step 5: Transform to AWS

```bash
# Transform to AWS/boto3
cd infrar-engine
./bin/transform \
    -provider aws \
    -plugins ../infrar-plugins/packages \
    -input ../my_app.py \
    -output ../my_app_aws.py

cd ..
```

**Check the output**:
```bash
cat my_app_aws.py
```

You should see pure boto3 code:
```python
import boto3

s3 = boto3.client('s3')

def data_pipeline():
    """Process data using cloud-agnostic storage operations."""

    s3.upload_file('raw_data.csv', 'analytics-data', 'data/raw/input.csv')
    s3.download_file('analytics-data', 'data/processed/results.csv', 'output_results.csv')
    s3.delete_object(Bucket='analytics-data', Key='data/temp/staging.csv')

    print("Pipeline completed successfully!")
# ... rest of code
```

**Verify it's valid Python**:
```bash
python3 -m py_compile my_app_aws.py
echo "✅ AWS version is valid Python!"
```

---

## 🌐 Step 6: Transform to GCP

```bash
# Transform to GCP/google-cloud-storage
cd infrar-engine
./bin/transform \
    -provider gcp \
    -plugins ../infrar-plugins/packages \
    -input ../my_app.py \
    -output ../my_app_gcp.py

cd ..
```

**Check the output**:
```bash
cat my_app_gcp.py
```

You should see pure google-cloud-storage code:
```python
from google.cloud import storage

storage_client = storage.Client()

def data_pipeline():
    """Process data using cloud-agnostic storage operations."""

    bucket = storage_client.bucket('analytics-data')
    blob = bucket.blob('data/raw/input.csv')
    blob.upload_from_filename('raw_data.csv')

    bucket = storage_client.bucket('analytics-data')
    blob = bucket.blob('data/processed/results.csv')
    blob.download_to_filename('output_results.csv')

    bucket = storage_client.bucket('analytics-data')
    blob = bucket.blob('data/temp/staging.csv')
    blob.delete()

    print("Pipeline completed successfully!")
# ... rest of code
```

**Verify it's valid Python**:
```bash
python3 -m py_compile my_app_gcp.py
echo "✅ GCP version is valid Python!"
```

---

## ✅ Step 7: Verify the Results

### Check: No Infrar Dependencies in Output

```bash
# AWS version should NOT import infrar
grep "import infrar" my_app_aws.py && echo "❌ FAIL" || echo "✅ PASS: No infrar imports"

# GCP version should NOT import infrar
grep "import infrar" my_app_gcp.py && echo "❌ FAIL" || echo "✅ PASS: No infrar imports"
```

### Check: Correct Provider SDKs

```bash
# AWS should use boto3
grep "import boto3" my_app_aws.py && echo "✅ PASS: Uses boto3" || echo "❌ FAIL"

# GCP should use google-cloud-storage
grep "from google.cloud import storage" my_app_gcp.py && echo "✅ PASS: Uses google-cloud-storage" || echo "❌ FAIL"
```

### Check: Function Structure Preserved

```bash
# Both should preserve the data_pipeline function
grep "def data_pipeline" my_app_aws.py && echo "✅ AWS: Function preserved"
grep "def data_pipeline" my_app_gcp.py && echo "✅ GCP: Function preserved"
```

---

## 🎉 Success Criteria

If all checks pass, you've proven:

✅ **Write Once**: Single source file (`my_app.py`)
✅ **Deploy Anywhere**: AWS (`my_app_aws.py`) and GCP (`my_app_gcp.py`)
✅ **Zero Runtime Overhead**: No infrar imports in production code
✅ **Native SDKs**: Pure boto3 and google-cloud-storage code
✅ **Structure Preserved**: Functions and logic intact

---

## 📊 Quick Test Summary

Run this script to verify everything:

```bash
#!/bin/bash
echo "🧪 Infrar MVP Test Summary"
echo "=========================="
echo

echo "1. SDK Installation"
python3 -c "import infrar; print(f'✅ Infrar SDK v{infrar.__version__}')" || echo "❌ SDK not installed"

echo
echo "2. Transformation Engine"
[[ -f infrar-engine/bin/transform ]] && echo "✅ Engine built" || echo "❌ Engine missing"

echo
echo "3. Plugins"
[[ -f infrar-plugins/packages/storage/aws/rules.yaml ]] && echo "✅ AWS plugin found" || echo "❌ AWS plugin missing"
[[ -f infrar-plugins/packages/storage/gcp/rules.yaml ]] && echo "✅ GCP plugin found" || echo "❌ GCP plugin missing"

echo
echo "4. Source App"
[[ -f my_app.py ]] && python3 -m py_compile my_app.py && echo "✅ Source app valid" || echo "❌ Source app invalid"

echo
echo "5. AWS Transformation"
[[ -f my_app_aws.py ]] && python3 -m py_compile my_app_aws.py && echo "✅ AWS version valid" || echo "❌ AWS version invalid"

echo
echo "6. GCP Transformation"
[[ -f my_app_gcp.py ]] && python3 -m py_compile my_app_gcp.py && echo "✅ GCP version valid" || echo "❌ GCP version invalid"

echo
echo "7. Zero Runtime Overhead"
! grep -q "import infrar" my_app_aws.py && echo "✅ AWS: No infrar dependency" || echo "❌ AWS: Still has infrar"
! grep -q "import infrar" my_app_gcp.py && echo "✅ GCP: No infrar dependency" || echo "❌ GCP: Still has infrar"

echo
echo "=========================="
echo "✅ MVP Test Complete!"
```

Save as `test_mvp.sh`, make executable, and run:
```bash
chmod +x test_mvp.sh
./test_mvp.sh
```

---

## 🐛 Troubleshooting

### Issue: SDK Import Fails

**Error**: `ModuleNotFoundError: No module named 'infrar'`

**Solution**:
```bash
# Ensure you're in the virtual environment
source infrar-test-env/bin/activate

# Reinstall
pip install -i https://test.pypi.org/simple/ infrar

# Or install from source
cd infrar-sdk-python && pip install -e .
```

### Issue: Transform Command Not Found

**Error**: `command not found: transform`

**Solution**:
```bash
cd infrar-engine
go build -o bin/transform ./cmd/transform
./bin/transform --help  # Should now work
```

### Issue: Plugins Not Found

**Error**: `failed to load rules: open ../infrar-plugins/packages: no such file`

**Solution**:
```bash
# Ensure plugins are cloned
ls infrar-plugins/packages/storage/aws/rules.yaml

# Or specify full path
./bin/transform -provider aws -plugins /full/path/to/infrar-plugins/packages -input my_app.py
```

### Issue: Transformation Syntax Error

**Error**: `Transformation error: validation error: invalid Python syntax`

**Cause**: Likely due to comments in functions (known limitation)

**Solution**: Remove inline comments from your source file:
```python
# BEFORE (may fail)
def process():
    # This comment causes issues
    upload(bucket='test', source='file.csv', destination='out.csv')

# AFTER (works)
def process():
    upload(bucket='test', source='file.csv', destination='out.csv')
```

See SDK README's "Known Limitations" section for more details.

---

## 📝 Next Steps

After successful testing:

1. **Try More Examples**
   - Test all 4 operations (upload, download, delete, list_objects)
   - Try more complex pipelines
   - Test error handling

2. **Real Cloud Testing**
   - Set up AWS credentials
   - Run `my_app_aws.py` with real S3 bucket
   - Set up GCP credentials
   - Run `my_app_gcp.py` with real GCS bucket

3. **Share Feedback**
   - Report issues on GitHub
   - Suggest improvements
   - Share your use cases

---

## 📚 Additional Resources

- **SDK Documentation**: https://github.com/QodeSrl/infrar-sdk-python
- **Engine Documentation**: https://github.com/QodeSrl/infrar-engine
- **Plugin Rules**: https://github.com/QodeSrl/infrar-plugins
- **Full Documentation**: https://github.com/QodeSrl/infrar-docs

---

## ✅ Test Checklist

- [ ] Prerequisites installed (Python 3.8+, Go 1.21+, Git)
- [ ] SDK installed from Test PyPI (or source)
- [ ] Engine built successfully
- [ ] Plugins cloned
- [ ] Sample app created
- [ ] AWS transformation successful
- [ ] GCP transformation successful
- [ ] All output files valid Python
- [ ] No infrar dependencies in output
- [ ] Function structure preserved
- [ ] Test script passes all checks

---

**Status**: Ready for Testing
**Last Updated**: October 20, 2025
**Version**: MVP v0.1.0
