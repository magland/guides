# DANDI Python API: Browsing and Exploring Data

This document provides a comprehensive guide for using the DANDI Python client to browse and retrieve information from the DANDI Archive. It focuses on **read-only operations** for exploring datasets programmatically.

## Table of Contents

- [Installation](#installation)
- [Core Concepts](#core-concepts)
- [Connecting to DANDI](#connecting-to-dandi)
- [Working with Dandisets](#working-with-dandisets)
- [Browsing Assets](#browsing-assets)
- [Accessing Metadata](#accessing-metadata)
- [Advanced Queries](#advanced-queries)
- [Error Handling](#error-handling)
- [Complete Examples](#complete-examples)

---

## Installation

```bash
pip install dandi
```

## Core Concepts

### Main Classes

- **`DandiAPIClient`**: The main entry point for interacting with a DANDI Archive instance
- **`RemoteDandiset`**: Represents a dataset (Dandiset) on the server
- **`RemoteAsset`**: Represents a file or Zarr archive within a Dandiset
  - `RemoteBlobAsset`: A regular file asset
  - `RemoteZarrAsset`: A Zarr archive asset
- **`Version`**: Information about a specific version of a Dandiset

### Key Terminology

- **Dandiset**: A dataset in the DANDI Archive (analogous to a repository)
- **Asset**: A file or data object within a Dandiset
- **Version**: Dandisets can have multiple versions; `"draft"` is the working version, while published versions have specific identifiers
- **Instance**: Different DANDI deployments (e.g., production "dandi", staging "dandi-staging")

---

## Connecting to DANDI

### Creating a Client

```python
from dandi.dandiapi import DandiAPIClient

# Connect to the main DANDI Archive (default)
client = DandiAPIClient()

# Or explicitly specify the instance
client = DandiAPIClient.for_dandi_instance("dandi")

# Connect to the staging server
client = DandiAPIClient.for_dandi_instance("dandi-staging")

# Or use a custom API URL
client = DandiAPIClient(api_url="https://api.dandiarchive.org/api")
```

### Using as a Context Manager

```python
# Recommended: automatically closes the session when done
with DandiAPIClient.for_dandi_instance("dandi") as client:
    # Your code here
    dandiset = client.get_dandiset("000027")
    # ...
```

---

## Working with Dandisets

### Getting a Specific Dandiset

```python
# Get a Dandiset by ID (defaults to most recent published version or draft)
dandiset = client.get_dandiset("000027")

# Get a specific version
dandiset = client.get_dandiset("000027", "0.210831.2033")

# Get the draft version explicitly
dandiset = client.get_dandiset("000027", "draft")

# Lazy loading (no API call until you access properties)
dandiset = client.get_dandiset("000027", lazy=True)

# Eager loading (makes API call immediately)
dandiset = client.get_dandiset("000027", lazy=False)
```

### Dandiset Properties

```python
# Basic information
print(dandiset.identifier)          # e.g., "000027"
print(dandiset.version_id)          # e.g., "0.210831.2033" or "draft"
print(dandiset.created)             # datetime object
print(dandiset.modified)            # datetime object
print(dandiset.contact_person)      # Contact person name

# Version information
print(dandiset.version)                         # Version object
print(dandiset.draft_version)                   # Always available
print(dandiset.most_recent_published_version)   # None if never published

# URLs for API access
print(dandiset.api_url)             # Full API URL for this Dandiset
print(dandiset.version_api_url)     # API URL for this specific version
```

### Listing All Dandisets

```python
# Get all Dandisets (returns iterator)
for dandiset in client.get_dandisets():
    print(f"{dandiset.identifier}: {dandiset.version.name}")

# Filter Dandisets
# Only draft-only Dandisets (not yet published)
for dandiset in client.get_dandisets(draft=True):
    print(dandiset.identifier)

# Only published Dandisets
for dandiset in client.get_dandisets(draft=False):
    print(dandiset.identifier)

# Include embargoed Dandisets (requires authentication)
for dandiset in client.get_dandisets(embargoed=True):
    print(dandiset.identifier)

# Search by text
for dandiset in client.get_dandisets(search="optogenetics"):
    print(f"{dandiset.identifier}: {dandiset.version.name}")

# Order results
for dandiset in client.get_dandisets(order="modified"):  # or "-modified" for descending
    print(f"{dandiset.identifier}: {dandiset.modified}")

# Get only Dandisets you own (requires authentication)
for dandiset in client.get_dandisets(mine=True):
    print(dandiset.identifier)
```

### Working with Versions

```python
# List all versions of a Dandiset
for version in dandiset.get_versions():
    print(f"{version.identifier}: {version.name}")
    print(f"  Assets: {version.asset_count}, Size: {version.size} bytes")
    print(f"  Created: {version.created}")

# Sort versions by creation date
for version in dandiset.get_versions(order="created"):
    print(version.identifier)

# Get detailed information about a specific version
version_info = dandiset.get_version("0.210831.2033")
print(f"Status: {version_info.status}")
print(f"Validation errors: {len(version_info.asset_validation_errors)}")

# Switch to a different version
older_dandiset = dandiset.for_version("0.210831.2033")
print(older_dandiset.version_id)  # "0.210831.2033"
```

---

## Browsing Assets

### Getting All Assets

```python
# Iterate through all assets in a Dandiset
for asset in dandiset.get_assets():
    print(f"{asset.path} - {asset.size} bytes")

# Order assets
for asset in dandiset.get_assets(order="path"):
    print(asset.path)

for asset in dandiset.get_assets(order="-modified"):  # Most recently modified first
    print(f"{asset.path}: {asset.modified}")
```

### Finding Assets by Path

```python
# Get a specific asset by exact path
asset = dandiset.get_asset_by_path("sub-RAT123/sub-RAT123.nwb")
print(f"Asset ID: {asset.identifier}")
print(f"Size: {asset.size} bytes")

# Get all assets under a path prefix
for asset in dandiset.get_assets_with_path_prefix("sub-RAT123/"):
    print(asset.path)

# Path prefix without trailing slash matches any path starting with that prefix
for asset in dandiset.get_assets_with_path_prefix("sub-RAT"):
    print(asset.path)  # Matches sub-RAT123, sub-RAT456, etc.
```

### Finding Assets by Pattern

```python
# Use glob patterns (added in version 0.44.0)
for asset in dandiset.get_assets_by_glob("*.nwb"):
    print(asset.path)

for asset in dandiset.get_assets_by_glob("sub-*/ses-*/data/*.nwb"):
    print(asset.path)

# Glob patterns can also be ordered
for asset in dandiset.get_assets_by_glob("*.nwb", order="size"):
    print(f"{asset.path}: {asset.size} bytes")
```

### Asset Properties

```python
asset = dandiset.get_asset_by_path("sub-RAT123/sub-RAT123.nwb")

# Basic properties
print(asset.identifier)       # Unique asset ID
print(asset.path)             # Path within the Dandiset
print(asset.size)             # Size in bytes
print(asset.created)          # Creation datetime
print(asset.modified)         # Modification datetime

# Asset type
print(asset.asset_type)       # AssetType.BLOB or AssetType.ZARR

# For blob assets
if isinstance(asset, RemoteBlobAsset):
    print(asset.blob)         # Blob ID

# For Zarr assets
if isinstance(asset, RemoteZarrAsset):
    print(asset.zarr)         # Zarr ID
    
# Download URL
print(asset.download_url)     # URL to download the asset
```

### Working with Zarr Assets

```python
# Get a Zarr asset
zarr_asset = dandiset.get_asset_by_path("data.zarr")

# Iterate through files within the Zarr
for entry in zarr_asset.iterfiles():
    print(f"{entry}: {entry.size} bytes")

# Filter Zarr entries by prefix
for entry in zarr_asset.iterfiles(prefix="data/arrays/"):
    print(entry)

# Get a specific entry within a Zarr
entry = zarr_asset.get_entry_by_path("data/arrays/voltage.npy")
print(f"Size: {entry.size}")
print(f"Modified: {entry.modified}")
print(f"Digest: {entry.digest.value}")

# Zarr entry properties
print(entry.name)              # Filename
print(entry.suffix)            # File extension
print(entry.stem)              # Filename without extension
```

---

## Accessing Metadata

### Dandiset Metadata

```python
# Get structured metadata (as dandischema.models.Dandiset)
metadata = dandiset.get_metadata()
print(metadata.name)
print(metadata.description)
print(metadata.contributor)

# Get raw metadata as a dictionary
raw_metadata = dandiset.get_raw_metadata()
print(raw_metadata["name"])
print(raw_metadata["description"])

# Convert to JSON
import json
print(json.dumps(raw_metadata, indent=2))
```

### Asset Metadata

```python
asset = dandiset.get_asset_by_path("sub-RAT123/sub-RAT123.nwb")

# Get structured metadata (as dandischema.models.Asset)
metadata = asset.get_metadata()
print(metadata.path)
print(metadata.contentSize)

# Get raw metadata as a dictionary
raw_metadata = asset.get_raw_metadata()
print(raw_metadata["path"])
print(raw_metadata["contentSize"])

# Access nested metadata fields
if "digest" in raw_metadata:
    digests = raw_metadata["digest"]
    for algo, value in digests.items():
        print(f"{algo}: {value}")
```

### Checksums and Digests

```python
# Get the primary digest for an asset
digest = asset.get_digest()
print(f"Algorithm: {digest.algorithm}")  # DigestType enum
print(f"Value: {digest.value}")

# Get a specific digest type as a string
etag = asset.get_raw_digest("dandi:dandi-etag")
print(etag)

# For Zarr assets, the primary digest is dandi-zarr-checksum
zarr_digest = zarr_asset.get_digest()
print(f"{zarr_digest.algorithm}: {zarr_digest.value}")
```

---

## Advanced Queries

### Getting an Asset by ID

```python
# Get an asset directly by its ID (without Dandiset context)
asset = client.get_asset("a1b2c3d4-5678-90ab-cdef-1234567890ab")
print(asset.path)
print(asset.size)

# Note: This asset won't have dandiset_id or version_id attributes
# Use dandiset.get_asset() if you need those
```

### Content URLs

```python
# Get a direct download URL from metadata
url = asset.get_content_url()
print(url)

# Get S3 URL specifically
s3_url = asset.get_content_url(regex=r"amazonaws.com")
print(s3_url)

# Follow redirects to get the final URL
final_url = asset.get_content_url(follow_redirects=True)

# Strip query parameters
clean_url = asset.get_content_url(strip_query=True)
```

### Downloading Asset Data

```python
from pathlib import Path

# Download a single asset
asset.download("local_file.nwb")

# Download with custom chunk size
asset.download("local_file.nwb", chunk_size=8192)

# Download all assets in a directory
dandiset.download_directory(
    assets_dirpath="sub-RAT123/",  # Remote path
    dirpath="./local_data/",        # Local destination
    chunk_size=1048576              # 1 MB chunks
)

# Get a file-like object for reading (for blob assets)
readable = asset.as_readable()
with readable.open() as fp:
    data = fp.read()
```

### Working with Multiple Dandisets

```python
# Compare assets across versions
draft = client.get_dandiset("000027", "draft")
published = client.get_dandiset("000027", "0.210831.2033")

draft_paths = {a.path for a in draft.get_assets()}
published_paths = {a.path for a in published.get_assets()}

new_files = draft_paths - published_paths
removed_files = published_paths - draft_paths

print(f"New files in draft: {len(new_files)}")
print(f"Removed from draft: {len(removed_files)}")
```

---

## Error Handling

### Common Exceptions

```python
from dandi.exceptions import NotFoundError

# Handle missing Dandisets
try:
    dandiset = client.get_dandiset("999999")
except NotFoundError:
    print("Dandiset not found")

# Handle missing assets
try:
    asset = dandiset.get_asset_by_path("nonexistent.nwb")
except NotFoundError:
    print("Asset not found")

# Handle missing versions
try:
    version = dandiset.get_version("0.999999.9999")
except NotFoundError:
    print("Version not found")
```

### HTTP Errors

```python
import requests

try:
    metadata = asset.get_metadata()
except requests.HTTPError as e:
    print(f"HTTP error: {e.response.status_code}")
    print(e.response.text)
```

---

## Complete Examples

### Example 1: Survey All Dandisets

```python
from dandi.dandiapi import DandiAPIClient

with DandiAPIClient.for_dandi_instance("dandi") as client:
    total_size = 0
    total_assets = 0
    
    for dandiset in client.get_dandisets():
        version = dandiset.version
        print(f"Dandiset {dandiset.identifier}: {version.name}")
        print(f"  Assets: {version.asset_count}")
        print(f"  Size: {version.size:,} bytes")
        
        total_size += version.size
        total_assets += version.asset_count
    
    print(f"\nTotal: {total_assets:,} assets, {total_size:,} bytes")
```

### Example 2: Find All NWB Files in a Dandiset

```python
from dandi.dandiapi import DandiAPIClient

with DandiAPIClient.for_dandi_instance("dandi") as client:
    dandiset = client.get_dandiset("000027")
    
    nwb_files = []
    total_size = 0
    
    for asset in dandiset.get_assets():
        if asset.path.endswith('.nwb'):
            nwb_files.append(asset)
            total_size += asset.size
    
    print(f"Found {len(nwb_files)} NWB files")
    print(f"Total size: {total_size:,} bytes")
    
    # Show the 5 largest files
    nwb_files.sort(key=lambda a: a.size, reverse=True)
    print("\nLargest files:")
    for asset in nwb_files[:5]:
        print(f"  {asset.path}: {asset.size:,} bytes")
```

### Example 3: Extract Metadata from Multiple Assets

```python
from dandi.dandiapi import DandiAPIClient
import json

with DandiAPIClient.for_dandi_instance("dandi") as client:
    dandiset = client.get_dandiset("000027")
    
    metadata_list = []
    
    for asset in dandiset.get_assets_by_glob("*.nwb"):
        raw_meta = asset.get_raw_metadata()
        
        # Extract specific fields
        info = {
            "path": raw_meta.get("path"),
            "size": raw_meta.get("contentSize"),
            "modified": raw_meta.get("blobDateModified"),
            "digest": raw_meta.get("digest", {}).get("dandi:dandi-etag")
        }
        metadata_list.append(info)
    
    # Save to JSON
    with open("asset_metadata.json", "w") as f:
        json.dump(metadata_list, f, indent=2)
    
    print(f"Extracted metadata for {len(metadata_list)} assets")
```

### Example 4: Compare Draft vs Published Version

```python
from dandi.dandiapi import DandiAPIClient

with DandiAPIClient.for_dandi_instance("dandi") as client:
    dandiset_id = "000027"
    
    # Get both versions
    draft = client.get_dandiset(dandiset_id, "draft")
    published = client.get_dandiset(dandiset_id)
    
    if published.version_id == "draft":
        print("No published version available")
    else:
        print(f"Comparing draft vs {published.version_id}")
        
        # Build path -> asset mappings
        draft_assets = {a.path: a for a in draft.get_assets()}
        published_assets = {a.path: a for a in published.get_assets()}
        
        # Find differences
        new_paths = set(draft_assets.keys()) - set(published_assets.keys())
        removed_paths = set(published_assets.keys()) - set(draft_assets.keys())
        common_paths = set(draft_assets.keys()) & set(published_assets.keys())
        
        modified = []
        for path in common_paths:
            if draft_assets[path].size != published_assets[path].size:
                modified.append(path)
        
        print(f"\nNew files in draft: {len(new_paths)}")
        print(f"Removed from draft: {len(removed_paths)}")
        print(f"Modified files: {len(modified)}")
        
        if new_paths:
            print("\nNew files:")
            for path in sorted(new_paths)[:10]:
                print(f"  + {path}")
```

### Example 5: Search Across Multiple Dandisets

```python
from dandi.dandiapi import DandiAPIClient

def search_for_file_pattern(pattern: str, max_dandisets: int = 10):
    """Search for files matching a pattern across multiple Dandisets."""
    
    with DandiAPIClient.for_dandi_instance("dandi") as client:
        results = []
        
        for i, dandiset in enumerate(client.get_dandisets()):
            if i >= max_dandisets:
                break
            
            print(f"Searching {dandiset.identifier}...", end=" ")
            
            matches = list(dandiset.get_assets_by_glob(pattern))
            if matches:
                print(f"found {len(matches)} matches")
                for asset in matches:
                    results.append({
                        "dandiset": dandiset.identifier,
                        "version": dandiset.version_id,
                        "path": asset.path,
                        "size": asset.size
                    })
            else:
                print("no matches")
        
        return results

# Find all Zarr files across first 10 Dandisets
zarr_files = search_for_file_pattern("*.zarr", max_dandisets=10)
print(f"\nTotal Zarr files found: {len(zarr_files)}")
```

### Example 6: Generate a Report on Dandiset Contents

```python
from dandi.dandiapi import DandiAPIClient
from collections import Counter
from pathlib import Path

with DandiAPIClient.for_dandi_instance("dandi") as client:
    dandiset = client.get_dandiset("000027")
    
    # Collect statistics
    file_extensions = Counter()
    total_size_by_ext = Counter()
    
    for asset in dandiset.get_assets():
        ext = Path(asset.path).suffix or "(no extension)"
        file_extensions[ext] += 1
        total_size_by_ext[ext] += asset.size
    
    # Generate report
    print(f"Dandiset {dandiset.identifier}: {dandiset.version.name}")
    print(f"Version: {dandiset.version_id}")
    print(f"\nFile Type Distribution:")
    print(f"{'Extension':<20} {'Count':>10} {'Total Size':>15}")
    print("-" * 50)
    
    for ext, count in file_extensions.most_common():
        size = total_size_by_ext[ext]
        print(f"{ext:<20} {count:>10} {size:>15,} bytes")
    
    print(f"\nTotal: {sum(file_extensions.values())} files, "
          f"{sum(total_size_by_ext.values()):,} bytes")
```

---

## Additional Notes

### Authentication

Most read operations don't require authentication, but some features do (embargoed Dandisets, private Dandisets):

```python
# Authenticate using an API key from environment variable
# Set DANDI_API_KEY environment variable before running
client = DandiAPIClient.for_dandi_instance("dandi", authenticate=True)

# Or authenticate after creating the client
client = DandiAPIClient()
client.dandi_authenticate()

# Or provide the token directly
client.authenticate("your-api-token-here")
```

### Performance Considerations

- Use `lazy=True` when creating Dandisets to defer API calls
- Iterators (`get_assets()`, `get_dandisets()`) fetch data in pages; they don't load everything into memory
- When iterating over many assets, consider filtering with path prefixes or glob patterns
- The client automatically retries failed requests and handles pagination

### Instance URLs

Common DANDI instances:
- Production: `https://api.dandiarchive.org/api` (instance name: "dandi")
- Staging: `https://api-staging.dandiarchive.org/api` (instance name: "dandi-staging")

### String Representations

```python
# Dandiset string format: "{instance}:{id}/{version}"
print(str(dandiset))  # "DANDI:000027/draft"

# Asset string format: "{instance}:assets/{asset_id}"
print(str(asset))     # "DANDI:assets/abc123..."

# Version string is just the version identifier
print(str(version))   # "0.210831.2033"
```

---

This document covers the essential read-only operations for browsing and exploring the DANDI Archive using Python. For write operations (uploading, deleting, modifying), refer to the official DANDI documentation.
