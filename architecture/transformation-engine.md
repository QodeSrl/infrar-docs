# Transformation Engine

## Overview

The Transformation Engine is the core innovation of Infrar. It converts provider-agnostic code (using infrar SDK) into provider-specific code (using native SDKs) at deployment time, eliminating runtime overhead while maintaining portability.

## Key Principle: Compile-Time Transformation

**Traditional Approach (Runtime Abstraction)**:
```python
# User writes this
from cloud_abstraction import storage

# Deployed code (same)
from cloud_abstraction import storage

# At runtime, library calls provider SDK
storage.upload(...)  # Internally: boto3.client('s3').upload()
                     # ⚠️ Adds latency, complexity, and overhead
```

**Infrar Approach (Compile-Time Transformation)**:
```python
# User writes this (stored in GitHub)
from infrar.storage import upload

upload(bucket='data', source='file.txt', dest='file.txt')

# Deployed code (transformed)
import boto3

s3 = boto3.client('s3')
s3.upload_file('file.txt', 'data', 'file.txt')

# ✅ Zero runtime overhead - direct SDK calls
```

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│              Transformation Pipeline                    │
│                                                         │
│  1. Parse        2. Analyze       3. Transform         │
│  ┌──────────┐   ┌──────────┐    ┌──────────┐         │
│  │   AST    │──>│  Infrar  │───>│ Provider │         │
│  │ Parser   │   │   Call   │    │   Code   │         │
│  │          │   │ Detector │    │Generator │         │
│  └──────────┘   └──────────┘    └──────────┘         │
│                                                         │
│  4. Validate     5. Output                             │
│  ┌──────────┐   ┌──────────┐                          │
│  │Generated │──>│  Write   │                          │
│  │   Code   │   │ Transformed│                         │
│  │ Validator│   │   Files  │                          │
│  └──────────┘   └──────────┘                          │
└─────────────────────────────────────────────────────────┘
```

## Implementation

### 1. AST Parsing

Parse user code into Abstract Syntax Tree for analysis.

**Python Example**:
```python
import ast

# User code
source_code = """
from infrar.storage import upload

upload(bucket='data', source='local.txt', destination='remote.txt')
"""

# Parse to AST
tree = ast.parse(source_code)

# Traverse AST
for node in ast.walk(tree):
    if isinstance(node, ast.ImportFrom):
        if node.module == 'infrar.storage':
            # Found infrar import
            imports = [alias.name for alias in node.names]

    if isinstance(node, ast.Call):
        if isinstance(node.func, ast.Name):
            if node.func.id == 'upload':
                # Found upload call
                # Extract arguments
                kwargs = {kw.arg: kw.value for kw in node.keywords}
```

**Node.js Example**:
```javascript
const parser = require('@babel/parser');
const traverse = require('@babel/traverse').default;

// User code
const sourceCode = `
import { upload } from 'infrar/storage';

upload({ bucket: 'data', source: 'local.txt', destination: 'remote.txt' });
`;

// Parse to AST
const ast = parser.parse(sourceCode, { sourceType: 'module' });

// Traverse AST
traverse(ast, {
  ImportDeclaration(path) {
    if (path.node.source.value === 'infrar/storage') {
      // Found infrar import
    }
  },
  CallExpression(path) {
    if (path.node.callee.name === 'upload') {
      // Found upload call
    }
  }
});
```

### 2. Infrar Call Detection

Identify all infrar SDK calls and their parameters.

**Detection Rules**:
```python
class InfrarCallDetector(ast.NodeVisitor):
    def __init__(self):
        self.infrar_calls = []

    def visit_Call(self, node):
        # Check if this is an infrar call
        if self.is_infrar_call(node):
            call_info = {
                'function': self.get_function_name(node),
                'args': self.extract_args(node),
                'kwargs': self.extract_kwargs(node),
                'lineno': node.lineno,
                'col_offset': node.col_offset
            }
            self.infrar_calls.append(call_info)

        self.generic_visit(node)

    def is_infrar_call(self, node):
        # Check if call is to infrar module
        # Example: infrar.storage.upload()
        if isinstance(node.func, ast.Attribute):
            if self.is_infrar_module(node.func):
                return True
        return False
```

### 3. Transformation Rules

Define how each infrar call maps to provider-specific code.

**Rule Structure**:
```python
# transformation_rules/storage.py

TRANSFORMATION_RULES = {
    'infrar.storage.upload': {
        'aws': {
            'imports': ['import boto3'],
            'code_template': """
s3_client = boto3.client('s3')
s3_client.upload_file(
    Filename={source},
    Bucket={bucket},
    Key={destination}
)
            """,
            'parameters': {
                'source': 'required',
                'bucket': 'required',
                'destination': 'required'
            }
        },
        'gcp': {
            'imports': ['from google.cloud import storage'],
            'code_template': """
storage_client = storage.Client()
bucket = storage_client.bucket({bucket})
blob = bucket.blob({destination})
blob.upload_from_filename({source})
            """,
            'parameters': {
                'source': 'required',
                'bucket': 'required',
                'destination': 'required'
            }
        },
        'azure': {
            'imports': ['from azure.storage.blob import BlobServiceClient'],
            'code_template': """
blob_service_client = BlobServiceClient.from_connection_string(
    os.environ['AZURE_STORAGE_CONNECTION_STRING']
)
blob_client = blob_service_client.get_blob_client(
    container={bucket},
    blob={destination}
)
with open({source}, 'rb') as data:
    blob_client.upload_blob(data)
            """,
            'parameters': {
                'source': 'required',
                'bucket': 'required',
                'destination': 'required'
            }
        }
    },

    'infrar.storage.download': {
        # Similar structure for download
    },

    'infrar.storage.list': {
        # Similar structure for list
    }
}
```

### 4. Code Generation

Generate provider-specific code using transformation rules.

**Generator Implementation**:
```python
class CodeGenerator:
    def __init__(self, provider, transformation_rules):
        self.provider = provider
        self.rules = transformation_rules
        self.imports_needed = set()

    def transform_call(self, call_info):
        """Transform a single infrar call to provider code."""
        function_name = call_info['function']

        # Get transformation rule
        if function_name not in self.rules:
            raise TransformationError(
                f"No transformation rule for {function_name}"
            )

        rule = self.rules[function_name][self.provider]

        # Collect imports
        self.imports_needed.update(rule['imports'])

        # Validate parameters
        self.validate_parameters(call_info, rule['parameters'])

        # Generate code from template
        code = self.generate_from_template(
            rule['code_template'],
            call_info['kwargs']
        )

        return code

    def generate_from_template(self, template, params):
        """Fill in template with actual parameter values."""
        code = template
        for param_name, param_value in params.items():
            placeholder = f"{{{param_name}}}"
            code = code.replace(placeholder, repr(param_value))
        return code

    def generate_imports(self):
        """Generate import statements."""
        return '\n'.join(sorted(self.imports_needed))
```

### 5. Full Transformation Example

**Input Code** (user's GitHub repo):
```python
# app.py
from infrar.storage import upload, download, list_objects

def process_data():
    # Download input file
    download(
        bucket='input-data',
        source='data.csv',
        destination='/tmp/data.csv'
    )

    # Process data
    result = analyze('/tmp/data.csv')

    # Upload result
    upload(
        bucket='output-data',
        source='/tmp/result.json',
        destination='results/result.json'
    )

    return result

def list_results():
    files = list_objects(
        bucket='output-data',
        prefix='results/'
    )
    return files
```

**Transformed Code for AWS** (deployed to cloud):
```python
# app.py (transformed)
import boto3

s3_client = boto3.client('s3')

def process_data():
    # Download input file
    s3_client.download_file(
        Bucket='input-data',
        Key='data.csv',
        Filename='/tmp/data.csv'
    )

    # Process data
    result = analyze('/tmp/data.csv')

    # Upload result
    s3_client.upload_file(
        Filename='/tmp/result.json',
        Bucket='output-data',
        Key='results/result.json'
    )

    return result

def list_results():
    response = s3_client.list_objects_v2(
        Bucket='output-data',
        Prefix='results/'
    )
    files = [obj['Key'] for obj in response.get('Contents', [])]
    return files
```

**Transformed Code for GCP** (deployed to cloud):
```python
# app.py (transformed)
from google.cloud import storage

storage_client = storage.Client()

def process_data():
    # Download input file
    bucket = storage_client.bucket('input-data')
    blob = bucket.blob('data.csv')
    blob.download_to_filename('/tmp/data.csv')

    # Process data
    result = analyze('/tmp/data.csv')

    # Upload result
    bucket = storage_client.bucket('output-data')
    blob = bucket.blob('results/result.json')
    blob.upload_from_filename('/tmp/result.json')

    return result

def list_results():
    bucket = storage_client.bucket('output-data')
    blobs = bucket.list_blobs(prefix='results/')
    files = [blob.name for blob in blobs]
    return files
```

## Transformation Pipeline

### Step-by-Step Process

```python
def transform_code(source_code, target_provider):
    """
    Complete transformation pipeline.

    Args:
        source_code: Original code with infrar SDK
        target_provider: 'aws', 'gcp', or 'azure'

    Returns:
        Transformed code with provider SDK
    """

    # Step 1: Parse source code to AST
    tree = ast.parse(source_code)

    # Step 2: Detect infrar calls
    detector = InfrarCallDetector()
    detector.visit(tree)
    infrar_calls = detector.infrar_calls

    # Step 3: Validate all calls can be transformed
    validator = TransformationValidator(target_provider)
    validator.validate(infrar_calls)

    # Step 4: Generate provider-specific code
    generator = CodeGenerator(target_provider, TRANSFORMATION_RULES)

    # Transform each call
    transformed_calls = []
    for call in infrar_calls:
        transformed = generator.transform_call(call)
        transformed_calls.append({
            'original_line': call['lineno'],
            'transformed_code': transformed
        })

    # Step 5: Replace infrar calls with transformed code
    rewriter = CodeRewriter(tree)
    new_tree = rewriter.replace_calls(transformed_calls)

    # Step 6: Generate imports
    imports = generator.generate_imports()

    # Step 7: Convert AST back to source code
    transformed_code = ast.unparse(new_tree)

    # Step 8: Add necessary imports
    final_code = imports + '\n\n' + transformed_code

    # Step 9: Format code (black, autopep8, etc.)
    formatted_code = format_code(final_code)

    return formatted_code
```

## Handling Edge Cases

### 1. Client Instantiation

**Problem**: Some providers need client setup, others don't.

**Solution**: Smart client management

```python
# AWS pattern (clients need instantiation)
if any_aws_calls:
    generate_code("""
s3_client = boto3.client('s3')
# ... reuse s3_client for all S3 operations
    """)

# GCP pattern (can instantiate per call or reuse)
if any_gcp_calls:
    generate_code("""
storage_client = storage.Client()
# ... reuse for all storage operations
    """)
```

### 2. Error Handling

**User Code**:
```python
from infrar.storage import upload
from infrar.exceptions import StorageError

try:
    upload(bucket='data', source='file.txt', destination='file.txt')
except StorageError as e:
    print(f"Upload failed: {e}")
```

**Challenge**: Different providers have different exceptions.

**Solution**: Map exceptions in transformation

```python
# AWS transformation includes exception mapping
try:
    s3_client.upload_file(...)
except boto3.exceptions.S3UploadFailedError as e:
    # Map to infrar exception for consistency
    raise StorageError(str(e))
```

### 3. Async Operations

**User Code**:
```python
from infrar.storage import upload_async

await upload_async(bucket='data', source='file.txt', destination='file.txt')
```

**AWS Transformation**:
```python
import aioboto3

async with aioboto3.client('s3') as s3_client:
    await s3_client.upload_file(...)
```

**GCP Transformation**:
```python
from google.cloud import storage

# GCP doesn't have native async, use thread pool
import asyncio
from concurrent.futures import ThreadPoolExecutor

loop = asyncio.get_event_loop()
await loop.run_in_executor(
    ThreadPoolExecutor(),
    lambda: storage_client.bucket('data').blob('file.txt').upload_from_filename('file.txt')
)
```

### 4. Provider-Specific Features

**User Code**:
```python
# Core feature (works everywhere)
from infrar.storage import upload

# Provider-specific feature (AWS only)
from infrar.aws.s3 import enable_versioning

upload(bucket='data', source='file.txt', destination='file.txt')

# Only works on AWS
enable_versioning(bucket='data')
```

**Behavior**:
- Core features: Transform for any provider
- Provider-specific features: Only available on that provider
- Infrar warns if user tries to migrate code with provider-specific features

## Local Development Support

For local development, infrar SDK actually works:

```python
# infrar/storage/__init__.py (SDK implementation)

import os

def upload(bucket, source, destination):
    """
    Upload file to cloud storage.

    In local development, this actually executes based on configured provider.
    In production, this code is transformed away.
    """
    provider = os.environ.get('INFRAR_PROVIDER', 'aws')

    if provider == 'aws':
        import boto3
        s3 = boto3.client('s3')
        s3.upload_file(source, bucket, destination)

    elif provider == 'gcp':
        from google.cloud import storage
        client = storage.Client()
        bucket = client.bucket(bucket)
        blob = bucket.blob(destination)
        blob.upload_from_filename(source)

    elif provider == 'azure':
        from azure.storage.blob import BlobServiceClient
        client = BlobServiceClient.from_connection_string(
            os.environ['AZURE_STORAGE_CONNECTION_STRING']
        )
        blob_client = client.get_blob_client(bucket, destination)
        with open(source, 'rb') as data:
            blob_client.upload_blob(data)

    elif provider == 'mock':
        # For testing without cloud access
        print(f"[MOCK] Upload {source} to {bucket}/{destination}")
```

**Local Development Flow**:
1. Developer installs `pip install infrar-storage`
2. Sets `INFRAR_PROVIDER=aws` (or gcp/azure/mock)
3. Runs code locally - it works!
4. Pushes to GitHub
5. Infrar transforms for deployment - no infrar SDK in production

## Performance Considerations

### Transformation Time

- **Parsing**: Fast (milliseconds for typical file)
- **Detection**: Fast (single AST traversal)
- **Generation**: Fast (template substitution)
- **Total**: < 1 second for typical application

### Caching

```python
# Cache transformation results
cache_key = hash(source_code + target_provider)

if cache_key in transformation_cache:
    return transformation_cache[cache_key]

# Perform transformation
result = transform_code(source_code, target_provider)

# Cache result
transformation_cache[cache_key] = result
return result
```

### Parallelization

Transform multiple files concurrently:
```python
import concurrent.futures

files_to_transform = ['app.py', 'worker.py', 'utils.py']

with concurrent.futures.ThreadPoolExecutor() as executor:
    futures = {
        executor.submit(transform_file, file, provider): file
        for file in files_to_transform
    }

    results = {}
    for future in concurrent.futures.as_completed(futures):
        file = futures[future]
        results[file] = future.result()
```

## Testing Strategy

### Unit Tests

```python
def test_upload_transformation_aws():
    source = """
from infrar.storage import upload
upload(bucket='data', source='file.txt', destination='file.txt')
    """

    result = transform_code(source, 'aws')

    assert 'import boto3' in result
    assert 's3_client.upload_file' in result
    assert 'infrar' not in result  # Completely removed

def test_upload_transformation_gcp():
    source = """
from infrar.storage import upload
upload(bucket='data', source='file.txt', destination='file.txt')
    """

    result = transform_code(source, 'gcp')

    assert 'from google.cloud import storage' in result
    assert 'storage_client' in result
    assert 'upload_from_filename' in result
```

### Integration Tests

```python
@pytest.mark.integration
def test_transform_and_execute_aws():
    # Transform code
    transformed = transform_code(sample_code, 'aws')

    # Write to file
    with open('/tmp/test_app.py', 'w') as f:
        f.write(transformed)

    # Execute transformed code
    exec(compile(transformed, '/tmp/test_app.py', 'exec'))

    # Verify file was uploaded to AWS
    s3 = boto3.client('s3')
    objects = s3.list_objects_v2(Bucket='test-bucket')
    assert 'file.txt' in [obj['Key'] for obj in objects['Contents']]
```

### End-to-End Tests

```python
def test_deploy_and_run():
    # Full pipeline test
    project = create_test_project(
        repo_url='https://github.com/test/app',
        provider='aws'
    )

    # Trigger deployment
    deployment = trigger_deployment(project.id)

    # Wait for completion
    wait_for_deployment(deployment.id, timeout=300)

    # Verify app is running
    response = requests.get(deployment.url)
    assert response.status_code == 200

    # Verify transformed code is deployed (not infrar SDK)
    container_code = get_deployed_code(deployment.id)
    assert 'boto3' in container_code
    assert 'infrar' not in container_code
```

## Future Enhancements

### 1. Optimization

Optimize generated code for performance:
```python
# Before
for file in files:
    s3_client = boto3.client('s3')  # Inefficient!
    s3_client.upload_file(file, 'bucket', file)

# After (optimized transformation)
s3_client = boto3.client('s3')  # Reuse client
for file in files:
    s3_client.upload_file(file, 'bucket', file)
```

### 2. Type Checking

Validate parameters at transformation time:
```python
# Catch errors before deployment
upload(bucket=123, source='file.txt')  # ❌ bucket must be string
# Transformation fails with helpful error message
```

### 3. Cost Estimation

Analyze code to estimate costs:
```python
# Detect patterns
if frequent_small_uploads:
    warn("Many small uploads detected. Consider batching for cost optimization.")
```

### 4. Provider-Specific Optimization

Use provider-specific features for better performance:
```python
# AWS: Use multipart upload for large files automatically
if file_size > 100_mb:
    generate_multipart_upload_code()
```

## Conclusion

The Transformation Engine is the key differentiator for Infrar. By transforming code at deployment time rather than runtime, we achieve:

- ✅ **Zero performance overhead**
- ✅ **Full provider feature access**
- ✅ **True portability**
- ✅ **Simple debugging** (native stack traces)
- ✅ **No vendor lock-in** (source of truth is infrar code)

Next: [Capability Catalog](capability-catalog.md) - How capabilities and implementations are defined
