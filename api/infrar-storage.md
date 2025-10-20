# infrar-storage API Reference

## Overview

The `infrar-storage` plugin provides provider-agnostic object storage operations. Use this for uploading, downloading, listing, and deleting files in cloud object storage.

**Supported Providers**:
- AWS S3
- GCP Cloud Storage
- Azure Blob Storage

## Installation

```bash
pip install infrar-storage
```

## Basic Usage

```python
from infrar.storage import upload, download, list_objects, delete

# Upload a file
upload(
    bucket='my-bucket',
    source='local-file.txt',
    destination='remote-file.txt'
)

# Download a file
download(
    bucket='my-bucket',
    source='remote-file.txt',
    destination='local-file.txt'
)

# List objects
files = list_objects(bucket='my-bucket', prefix='data/')

# Delete an object
delete(bucket='my-bucket', path='old-file.txt')
```

## API Reference

### upload()

Upload a file to object storage.

```python
def upload(
    bucket: str,
    source: str,
    destination: str,
    content_type: Optional[str] = None,
    metadata: Optional[Dict[str, str]] = None
) -> None
```

**Parameters**:
- `bucket` (str, required): Name of the storage bucket
- `source` (str, required): Local file path to upload
- `destination` (str, required): Remote object key/path
- `content_type` (str, optional): MIME type of the file
- `metadata` (dict, optional): Custom metadata key-value pairs

**Returns**: None

**Raises**:
- `StorageError`: If upload fails
- `FileNotFoundError`: If source file doesn't exist
- `PermissionError`: If insufficient permissions

**Example**:
```python
from infrar.storage import upload

# Simple upload
upload(
    bucket='images',
    source='/tmp/photo.jpg',
    destination='photos/2024/photo.jpg'
)

# With content type and metadata
upload(
    bucket='images',
    source='/tmp/photo.jpg',
    destination='photos/2024/photo.jpg',
    content_type='image/jpeg',
    metadata={'photographer': 'John Doe', 'year': '2024'}
)
```

**Provider Transformations**:

AWS:
```python
import boto3

s3_client = boto3.client('s3')
s3_client.upload_file(
    Filename='/tmp/photo.jpg',
    Bucket='images',
    Key='photos/2024/photo.jpg',
    ExtraArgs={
        'ContentType': 'image/jpeg',
        'Metadata': {'photographer': 'John Doe', 'year': '2024'}
    }
)
```

GCP:
```python
from google.cloud import storage

storage_client = storage.Client()
bucket = storage_client.bucket('images')
blob = bucket.blob('photos/2024/photo.jpg')
blob.metadata = {'photographer': 'John Doe', 'year': '2024'}
blob.upload_from_filename('/tmp/photo.jpg', content_type='image/jpeg')
```

Azure:
```python
from azure.storage.blob import BlobServiceClient
import os

blob_service_client = BlobServiceClient.from_connection_string(
    os.environ['AZURE_STORAGE_CONNECTION_STRING']
)
blob_client = blob_service_client.get_blob_client(
    container='images',
    blob='photos/2024/photo.jpg'
)
with open('/tmp/photo.jpg', 'rb') as data:
    blob_client.upload_blob(
        data,
        content_settings={'content_type': 'image/jpeg'},
        metadata={'photographer': 'John Doe', 'year': '2024'}
    )
```

---

### download()

Download a file from object storage.

```python
def download(
    bucket: str,
    source: str,
    destination: str,
    if_modified_since: Optional[datetime] = None
) -> None
```

**Parameters**:
- `bucket` (str, required): Name of the storage bucket
- `source` (str, required): Remote object key/path
- `destination` (str, required): Local file path to save
- `if_modified_since` (datetime, optional): Only download if modified after this time

**Returns**: None

**Raises**:
- `StorageError`: If download fails
- `ObjectNotFoundError`: If object doesn't exist
- `PermissionError`: If insufficient permissions

**Example**:
```python
from infrar.storage import download
from datetime import datetime, timedelta

# Simple download
download(
    bucket='images',
    source='photos/2024/photo.jpg',
    destination='/tmp/photo.jpg'
)

# Conditional download (only if modified in last 24 hours)
download(
    bucket='images',
    source='photos/2024/photo.jpg',
    destination='/tmp/photo.jpg',
    if_modified_since=datetime.now() - timedelta(days=1)
)
```

---

### list_objects()

List objects in a bucket with optional prefix filtering.

```python
def list_objects(
    bucket: str,
    prefix: str = '',
    max_results: Optional[int] = None,
    delimiter: Optional[str] = None
) -> List[ObjectInfo]
```

**Parameters**:
- `bucket` (str, required): Name of the storage bucket
- `prefix` (str, optional): Filter objects by prefix (default: '')
- `max_results` (int, optional): Maximum number of results to return
- `delimiter` (str, optional): Delimiter for grouping (typically '/')

**Returns**: List of `ObjectInfo` objects

**ObjectInfo** dataclass:
```python
@dataclass
class ObjectInfo:
    key: str                    # Object key/path
    size: int                   # Size in bytes
    last_modified: datetime     # Last modification time
    etag: str                   # Entity tag
    storage_class: str          # Storage class
    metadata: Dict[str, str]    # Custom metadata
```

**Example**:
```python
from infrar.storage import list_objects

# List all objects
objects = list_objects(bucket='images')
for obj in objects:
    print(f"{obj.key} - {obj.size} bytes")

# List with prefix (only photos from 2024)
objects = list_objects(
    bucket='images',
    prefix='photos/2024/',
    max_results=100
)

# List "directories" (using delimiter)
objects = list_objects(
    bucket='images',
    prefix='photos/',
    delimiter='/'
)
# Returns: ['photos/2023/', 'photos/2024/', 'photos/2025/']
```

---

### delete()

Delete an object from storage.

```python
def delete(
    bucket: str,
    path: str,
    if_exists: bool = True
) -> bool
```

**Parameters**:
- `bucket` (str, required): Name of the storage bucket
- `path` (str, required): Object key/path to delete
- `if_exists` (bool, optional): If False, raise error if object doesn't exist

**Returns**: True if deleted, False if object didn't exist (when `if_exists=True`)

**Raises**:
- `StorageError`: If deletion fails
- `ObjectNotFoundError`: If object doesn't exist and `if_exists=False`
- `PermissionError`: If insufficient permissions

**Example**:
```python
from infrar.storage import delete

# Simple delete
delete(bucket='images', path='photos/old-photo.jpg')

# Delete with error if not exists
try:
    delete(bucket='images', path='photos/important.jpg', if_exists=False)
except ObjectNotFoundError:
    print("File was already deleted!")
```

---

### delete_many()

Delete multiple objects in a single operation (more efficient than individual deletes).

```python
def delete_many(
    bucket: str,
    paths: List[str]
) -> Dict[str, bool]
```

**Parameters**:
- `bucket` (str, required): Name of the storage bucket
- `paths` (list, required): List of object keys/paths to delete

**Returns**: Dictionary mapping each path to deletion status (True if deleted, False if failed)

**Example**:
```python
from infrar.storage import delete_many

# Delete multiple files
results = delete_many(
    bucket='images',
    paths=[
        'photos/2020/old1.jpg',
        'photos/2020/old2.jpg',
        'photos/2020/old3.jpg'
    ]
)

for path, success in results.items():
    if success:
        print(f"Deleted {path}")
    else:
        print(f"Failed to delete {path}")
```

---

### copy()

Copy an object within the same bucket or to another bucket.

```python
def copy(
    source_bucket: str,
    source_path: str,
    destination_bucket: str,
    destination_path: str,
    metadata: Optional[Dict[str, str]] = None
) -> None
```

**Parameters**:
- `source_bucket` (str, required): Source bucket name
- `source_path` (str, required): Source object key/path
- `destination_bucket` (str, required): Destination bucket name
- `destination_path` (str, required): Destination object key/path
- `metadata` (dict, optional): Metadata for destination object

**Returns**: None

**Example**:
```python
from infrar.storage import copy

# Copy within same bucket
copy(
    source_bucket='images',
    source_path='photos/2024/photo.jpg',
    destination_bucket='images',
    destination_path='photos/archive/photo.jpg'
)

# Copy to different bucket
copy(
    source_bucket='images',
    source_path='photos/2024/photo.jpg',
    destination_bucket='backups',
    destination_path='2024/photo.jpg',
    metadata={'backup_date': '2024-01-15'}
)
```

---

### get_metadata()

Get metadata for an object without downloading it.

```python
def get_metadata(
    bucket: str,
    path: str
) -> ObjectInfo
```

**Parameters**:
- `bucket` (str, required): Name of the storage bucket
- `path` (str, required): Object key/path

**Returns**: `ObjectInfo` object with metadata

**Example**:
```python
from infrar.storage import get_metadata

# Get object info
info = get_metadata(bucket='images', path='photos/2024/photo.jpg')

print(f"Size: {info.size} bytes")
print(f"Last modified: {info.last_modified}")
print(f"Content type: {info.metadata.get('content_type')}")
```

---

### generate_signed_url()

Generate a signed URL for temporary access to an object.

```python
def generate_signed_url(
    bucket: str,
    path: str,
    expiration_seconds: int = 3600,
    method: str = 'GET'
) -> str
```

**Parameters**:
- `bucket` (str, required): Name of the storage bucket
- `path` (str, required): Object key/path
- `expiration_seconds` (int, optional): URL validity in seconds (default: 3600 = 1 hour)
- `method` (str, optional): HTTP method ('GET' or 'PUT')

**Returns**: Signed URL string

**Example**:
```python
from infrar.storage import generate_signed_url

# Generate download URL (valid for 1 hour)
url = generate_signed_url(
    bucket='images',
    path='photos/2024/photo.jpg',
    expiration_seconds=3600
)

# Share URL with user
print(f"Download link: {url}")

# Generate upload URL (valid for 15 minutes)
upload_url = generate_signed_url(
    bucket='images',
    path='photos/2024/new-photo.jpg',
    expiration_seconds=900,
    method='PUT'
)

# User can upload directly to this URL
# curl -X PUT --upload-file photo.jpg "{upload_url}"
```

---

## Exceptions

### StorageError

Base exception for all storage errors.

```python
from infrar.storage.exceptions import StorageError

try:
    upload(bucket='images', source='file.txt', destination='file.txt')
except StorageError as e:
    print(f"Storage operation failed: {e}")
```

### ObjectNotFoundError

Raised when trying to access a non-existent object.

```python
from infrar.storage.exceptions import ObjectNotFoundError

try:
    download(bucket='images', source='missing.jpg', destination='local.jpg')
except ObjectNotFoundError:
    print("File doesn't exist")
```

### BucketNotFoundError

Raised when trying to access a non-existent bucket.

```python
from infrar.storage.exceptions import BucketNotFoundError

try:
    list_objects(bucket='non-existent-bucket')
except BucketNotFoundError:
    print("Bucket doesn't exist")
```

### PermissionError

Raised when operation is denied due to insufficient permissions.

```python
from infrar.storage.exceptions import PermissionError

try:
    delete(bucket='protected-bucket', path='file.txt')
except PermissionError:
    print("Not authorized to delete this file")
```

---

## Configuration

### Local Development

For local development, set the provider and credentials:

```bash
# AWS
export INFRAR_PROVIDER=aws
export AWS_PROFILE=dev
# or
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...

# GCP
export INFRAR_PROVIDER=gcp
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json

# Azure
export INFRAR_PROVIDER=azure
export AZURE_STORAGE_CONNECTION_STRING=...
```

### Testing with Mocks

For testing without cloud access:

```bash
export INFRAR_PROVIDER=mock
```

```python
from infrar.storage import upload

# This won't actually upload, just logs
upload(bucket='test', source='file.txt', destination='file.txt')
# Output: [MOCK] Upload file.txt to test/file.txt
```

---

## Best Practices

### 1. Use Consistent Naming

```python
# Good: Consistent, organized structure
upload(bucket='data', source='file.csv', destination='raw/2024/01/15/file.csv')

# Bad: Inconsistent structure
upload(bucket='data', source='file.csv', destination='file_20240115.csv')
```

### 2. Handle Errors Gracefully

```python
from infrar.storage import upload
from infrar.storage.exceptions import StorageError
import logging

try:
    upload(bucket='data', source='file.txt', destination='file.txt')
except StorageError as e:
    logging.error(f"Failed to upload: {e}")
    # Implement retry logic or fallback
```

### 3. Use Metadata for Tracking

```python
upload(
    bucket='data',
    source='report.pdf',
    destination='reports/2024/report.pdf',
    metadata={
        'generated_by': 'analytics_service',
        'generated_at': '2024-01-15T10:00:00Z',
        'version': '1.0'
    }
)
```

### 4. Clean Up Temporary Files

```python
import os
from infrar.storage import download

download(bucket='data', source='data.csv', destination='/tmp/data.csv')

try:
    process_data('/tmp/data.csv')
finally:
    os.remove('/tmp/data.csv')  # Clean up
```

### 5. Use Signed URLs for Direct Access

```python
from infrar.storage import generate_signed_url

# Instead of downloading and serving
# download(bucket='images', source='photo.jpg', destination='/tmp/photo.jpg')
# serve_file('/tmp/photo.jpg')

# Generate signed URL and redirect
url = generate_signed_url(bucket='images', path='photo.jpg', expiration_seconds=300)
return redirect(url)  # Client downloads directly from storage
```

---

## Performance Considerations

### Batch Operations

```python
from infrar.storage import delete_many

# Bad: Individual deletes (slow)
for path in paths:
    delete(bucket='data', path=path)

# Good: Batch delete (fast)
delete_many(bucket='data', paths=paths)
```

### Streaming Large Files

For very large files, consider streaming:

```python
# Future enhancement (not in MVP)
from infrar.storage import upload_stream

with open('large-file.zip', 'rb') as f:
    upload_stream(bucket='data', destination='large-file.zip', stream=f)
```

---

## Examples

### Example 1: File Backup System

```python
from infrar.storage import upload, list_objects
import os

def backup_directory(local_dir, bucket, remote_prefix):
    """Backup all files from local directory to storage."""
    for root, dirs, files in os.walk(local_dir):
        for file in files:
            local_path = os.path.join(root, file)
            relative_path = os.path.relpath(local_path, local_dir)
            remote_path = f"{remote_prefix}/{relative_path}"

            print(f"Uploading {local_path} → {remote_path}")
            upload(bucket=bucket, source=local_path, destination=remote_path)

# Backup /var/log to storage
backup_directory('/var/log', 'backups', 'logs/2024-01-15')
```

### Example 2: Data Pipeline

```python
from infrar.storage import download, upload, list_objects
import pandas as pd

def process_data_pipeline(bucket, input_prefix, output_prefix):
    """Process CSV files from storage."""
    # List input files
    objects = list_objects(bucket=bucket, prefix=input_prefix)

    for obj in objects:
        if not obj.key.endswith('.csv'):
            continue

        # Download
        local_input = f'/tmp/input_{obj.key.split("/")[-1]}'
        download(bucket=bucket, source=obj.key, destination=local_input)

        # Process
        df = pd.read_csv(local_input)
        df_processed = df[df['value'] > 100]  # Filter

        # Upload result
        local_output = local_input.replace('input_', 'output_')
        df_processed.to_csv(local_output, index=False)

        output_key = obj.key.replace(input_prefix, output_prefix)
        upload(bucket=bucket, source=local_output, destination=output_key)

        # Clean up
        os.remove(local_input)
        os.remove(local_output)

# Process all CSV files in 'raw/' to 'processed/'
process_data_pipeline('data', 'raw/2024-01/', 'processed/2024-01/')
```

### Example 3: Image Gallery

```python
from infrar.storage import upload, list_objects, generate_signed_url

def upload_image(image_path, image_id):
    """Upload image and generate thumbnail."""
    # Upload original
    upload(
        bucket='images',
        source=image_path,
        destination=f'originals/{image_id}.jpg',
        content_type='image/jpeg',
        metadata={'uploaded_by': 'user123'}
    )

    # Create and upload thumbnail
    create_thumbnail(image_path, f'/tmp/thumb_{image_id}.jpg')
    upload(
        bucket='images',
        source=f'/tmp/thumb_{image_id}.jpg',
        destination=f'thumbnails/{image_id}.jpg',
        content_type='image/jpeg'
    )

def get_gallery_images():
    """Get signed URLs for all gallery images."""
    images = []
    thumbnails = list_objects(bucket='images', prefix='thumbnails/')

    for thumb in thumbnails:
        image_id = thumb.key.split('/')[-1].replace('.jpg', '')

        images.append({
            'id': image_id,
            'thumbnail_url': generate_signed_url(
                bucket='images',
                path=thumb.key,
                expiration_seconds=3600
            ),
            'full_url': generate_signed_url(
                bucket='images',
                path=f'originals/{image_id}.jpg',
                expiration_seconds=3600
            )
        })

    return images
```

---

## Troubleshooting

### Upload Fails with PermissionError

**Problem**: `PermissionError: Insufficient permissions to write to bucket`

**Solution**: Ensure your credentials have write permissions:

AWS:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:PutObjectAcl"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

### Large File Upload Times Out

**Problem**: Upload fails for files > 5GB

**Solution**: Use multipart upload (automatic for AWS/GCP, handled by SDK):

```python
# SDK handles this automatically
upload(bucket='data', source='large-file.zip', destination='large-file.zip')
# Uses multipart upload if file > 100MB
```

### List Returns Empty Despite Files Existing

**Problem**: `list_objects()` returns empty list but files exist

**Solution**: Check prefix syntax:

```python
# Wrong: Leading slash
objects = list_objects(bucket='data', prefix='/folder')  # Returns empty

# Correct: No leading slash
objects = list_objects(bucket='data', prefix='folder/')  # Works
```

---

## Changelog

### v1.0.0 (MVP)
- Initial release
- Basic operations: upload, download, list, delete
- AWS S3 support
- GCP Cloud Storage support
- Python 3.8+ support

### v1.1.0 (Planned)
- Azure Blob Storage support
- Streaming uploads for large files
- Progress callbacks
- Retry logic with exponential backoff

### v1.2.0 (Planned)
- Parallel uploads/downloads
- Local caching
- Compression support

---

## Support

For issues or questions:
- GitHub Issues: https://github.com/infrar/infrar-storage/issues
- Documentation: https://docs.infrar.io/plugins/storage
- Email: support@infrar.io
