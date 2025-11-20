# Yelp Review Similarity Search using Shingling, Winnowing, and Hashing

This project implements a document similarity search system using the Winnowing algorithm to generate hashed shingles from Yelp reviews. It demonstrates how to process large-scale text data, build an inverted index in MongoDB Atlas, and measure search performance across different cluster configurations. A paid tier of the MongoDB Atlas was used in this project for sharding experiments.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [MongoDB Atlas Setup](#mongodb-atlas-setup)
  - [No Sharding (M0/M10)](#no-sharding-m0m10)
  - [3-Shard Configuration (M30+)](#3-shard-configuration-m30)
  - [6-Shard Configuration (M30+)](#6-shard-configuration-m30)
- [Installation](#installation)
- [Usage](#usage)
- [Project Structure](#project-structure)
- [Algorithm Details](#algorithm-details)
- [Performance Benchmarking](#performance-benchmarking)
- [Results](#results)
- [Troubleshooting](#troubleshooting)

## Overview

This system processes Yelp reviews to:
1. Clean and normalize text
2. Generate 5-word shingles (k=5)
3. Hash shingles using SHA-1
4. Apply the Winnowing algorithm (window size=4) to reduce fingerprints
5. Build an inverted index in MongoDB Atlas
6. Perform similarity searches across different cluster configurations

## Features

- **Text Processing**: Automated cleaning, normalization, and shingling
- **Winnowing Algorithm**: Efficient fingerprint selection to reduce storage
- **Inverted Index**: Shingle-centric and review-centric data structures
- **Deduplication**: Automatic removal of duplicate shingles
- **Performance Indexing**: Optimized MongoDB indexes for fast queries
- **Multi-Configuration Testing**: Compare performance across 0, 3, and 6 shards
- **Visualization**: Comprehensive charts and statistics
- **Scalability**: Processes 1M+ reviews efficiently

## Prerequisites

### Software Requirements
- Python 3.8+
- Google Colab (or local Jupyter environment)
- MongoDB Atlas account (free tier available)

### Python Packages
```python
dnspython
gdown
pymongo
tqdm
pandas
matplotlib
numpy
```

## MongoDB Atlas Setup

### No Sharding (M0/M10)

**Best for**: Development, testing, small datasets (<512MB)

#### Steps:

1. **Create a Cluster**
   - Go to [MongoDB Atlas](https://cloud.mongodb.com)
   - Click "Build a Database"
   - Choose **M0 (Free)** or **M10 (Shared)**
   - Select your preferred cloud provider and region
   - Name your cluster (e.g., `Cluster0`)

2. **Configure Network Access**
   - Navigate to **Network Access** → **Add IP Address**
   - Click **Allow Access from Anywhere** (0.0.0.0/0)
   - Or add your specific IP for better security

3. **Create Database User**
   - Go to **Database Access** → **Add New Database User**
   - Username: `christinalitsiba_db_user`
   - Password: `[your-secure-password]`
   - Database User Privileges: **Atlas admin**

4. **Get Connection String**
   - Click **Connect** on your cluster
   - Choose **Connect your application**
   - Copy the connection string:
     ```
     mongodb+srv://<username>:<password>@cluster0.xxxxx.mongodb.net/?retryWrites=true&w=majority
     ```

5. **Update Notebook**
   ```python
   ATLAS_URI = "mongodb+srv://christinalitsiba_db_user:YOUR_PASSWORD@cluster0.xxxxx.mongodb.net/?retryWrites=true&w=majority"
   ```

6. **Run Data Load**
   - Execute all cells through **Step 6** (upload data)
   - Monitor progress in the output

---

### 3-Shard Configuration (M30+)

**Best for**: Production workloads, horizontal scalability, 100K-1M+ documents

#### Steps:

1. **Upgrade to M30 Cluster**
   - Go to your cluster → **⋮** → **Edit Configuration**
   - Under **Cluster Tier**, select **M30** (or higher)
   - Click **Review Changes** → **Apply Changes**
   - Wait 10-15 minutes for upgrade to complete

2. **Enable Sharding**
   - After upgrade, go to **Clusters** → **⋮** → **Edit Configuration**
   - Scroll to **Additional Settings**
   - Enable **Sharding**
   - Set **Number of Shards: 3**
   - Click **Review Changes** → **Apply Changes**

3. **Wait for Provisioning**
   - Atlas will provision 3 shards (15-20 minutes)
   - Check **Metrics** tab to confirm all shards are active

4. **Create Shard Key Index** (Run in notebook)
   ```python
   from pymongo import MongoClient, ASCENDING
   
   client = MongoClient(ATLAS_URI)
   db = client["YELP"]
   collection = db["REVIEWS"]
   
   # Create index on shard key
   collection.create_index([("shingle_hash", ASCENDING)], name="shingle_shard_key")
   print("Shard key index created")
   ```

5. **Shard the Collection** (Run in notebook)
   ```python
   admin_db = client.admin
   
   # Enable sharding on database
   admin_db.command("enableSharding", "YELP")
   
   # Shard the collection
   admin_db.command({
       "shardCollection": "YELP.REVIEWS",
       "key": {"shingle_hash": "hashed"}  # Hashed sharding for even distribution
   })
   print("Collection sharded")
   ```

6. **Verify Distribution** (Wait 5-10 minutes)
   ```python
   import time
   
   time.sleep(300)  # Wait 5 minutes
   
   stats = db.command("collStats", "REVIEWS")
   if 'shards' in stats:
       for shard, shard_stats in stats['shards'].items():
           count = shard_stats.get('count', 0)
           print(f"{shard}: {count:,} documents")
   ```

7. **Expected Distribution**
   ```
   atlas-xxxxx-shard-0: ~4,118,000 docs (33.3%)
   atlas-xxxxx-shard-1: ~4,118,000 docs (33.3%)
   atlas-xxxxx-shard-2: ~4,119,000 docs (33.4%)
   ```

8. **Run Performance Tests**
   - Execute **Step 7** (Search Experiment)
   - Execute **Step 8** (Index Analysis)
   - Save results:
     ```python
     save_configuration_results("3 Shards")
     ```

---

### 6-Shard Configuration (M30+)

**Best for**: Maximum parallelism, 1M+ documents, lowest latency requirements

#### Steps:

1. **Scale to 6 Shards**
   - Go to **Clusters** → **⋮** → **Edit Configuration**
   - Under **Additional Settings** → **Number of Shards: 6**
   - Click **Review Changes** → **Apply Changes**

2. **Wait for Provisioning**
   - Atlas adds 3 more shards (10-15 minutes)
   - Check **Metrics** tab for 6 active shards

3. **Trigger Data Redistribution** (Automatic)
   - Atlas automatically rebalances data across new shards
   - Monitor progress: **Metrics** → **Opcounters** (look for "chunk migrations")

4. **Verify Even Distribution** (Wait 10-15 minutes)
   ```python
   import time
   
   time.sleep(600)  # Wait 10 minutes
   
   stats = db.command("collStats", "REVIEWS")
   if 'shards' in stats:
       total = 0
       for shard, shard_stats in stats['shards'].items():
           count = shard_stats.get('count', 0)
           total += count
           pct = (count / 12355500) * 100
           print(f"{shard}: {count:,} docs ({pct:.1f}%)")
       print(f"TOTAL: {total:,}")
   ```

5. **Expected Distribution**
   ```
   atlas-xxxxx-shard-0: ~2,059,000 docs (16.7%)
   atlas-xxxxx-shard-1: ~2,059,000 docs (16.7%)
   atlas-xxxxx-shard-2: ~2,059,000 docs (16.7%)
   atlas-xxxxx-shard-3: ~2,059,000 docs (16.7%)
   atlas-xxxxx-shard-4: ~2,059,000 docs (16.7%)
   atlas-xxxxx-shard-5: ~2,060,500 docs (16.7%)
   ```

6. **Run Performance Tests**
   - Execute **Step 7** (Search Experiment)
   - Execute **Step 8** (Index Analysis)
   - Save results:
     ```python
     save_configuration_results("6 Shards")
     ```

7. **Generate Comparison**
   - Execute **Step 10** to visualize results across all configurations

---

## Installation

### 1. Clone or Download the Notebook
```bash
# If using Git
git clone <repository-url>
cd yelp-similarity-search

# Or download the .ipynb file directly
```

### 2. Open in Google Colab
- Go to [Google Colab](https://colab.research.google.com)
- **File** → **Upload notebook**
- Select `yelp_shingling_export_fixed_dedup_repaired_bootstrapped (2).ipynb`

### 3. Run Bootstrap Cell (Step 1)
```python
# Cell automatically installs required packages
# Run this first, then restart the runtime if prompted
```

### 4. Configure MongoDB Connection
- Replace the `ATLAS_URI` in the notebook with your connection string
- Update the password in the connection string

---

## Usage

### Step-by-Step Execution

#### **Step 1**: Bootstrap Dependencies
```python
# Installs: gdown, pymongo, tqdm, dnspython
# Safe to re-run
```

#### **Step 2**: Download Yelp Dataset
```python
# Downloads from Google Drive (if not already present)
# File: yelp_academic_dataset_review.json (~5GB)
```

#### **Step 3**: Data Processing
```python
# Processes 1,000,000 reviews
# Generates:
#   - review_centric.json: Review → Shingles mapping
#   - shingle_centric.json: Shingle → Reviews mapping
```

#### **Step 4-5**: Upload to MongoDB
```python
# Uploads shingle_centric.json to Atlas
# Creates indexes automatically
# Skips duplicates on reruns
```

#### **Step 5.5**: Remove Duplicates
```python
# Ensures unique shingles (run once)
```

#### **Step 5.6**: Create Indexes
```python
# Creates optimized indexes:
#   - shingle_hash_idx
#   - shingle_docs_idx (compound)
```

#### **Step 6**: Verify Setup
```python
# Confirms:
#   - Database connection
#   - Document count
#   - Field structure
```

#### **Step 7**: Search Performance Test
```python
# Runs 1,000 similarity searches
# Measures latency (Avg, P50, P95, P99)
```

#### **Step 8**: Index Analysis
```python
# Analyzes:
#   - Shingles per review
#   - Storage efficiency
#   - Index size estimation
```

#### **Step 9**: Visualizations
```python
# Generates:
#   - Search time histogram
#   - Boxplot
#   - Cumulative distribution
#   - Percentile bars
```

#### **Step 10**: Multi-Configuration Testing
```python
# Saves results for each configuration:
save_configuration_results("No Shards")    # After M0/M10 tests
save_configuration_results("3 Shards")     # After M30 3-shard tests
save_configuration_results("6 Shards")     # After M30 6-shard tests
```

#### **Steps 11-11.6**: Sharding Configuration (M30+ Only)
```python
# Automated sharding setup and verification
```

---

## Project Structure

```
.
├── yelp_shingling_export_fixed_dedup_repaired_bootstrapped (2).ipynb
├── README.md
├── data/
│   ├── yelp_academic_dataset_review.json (downloaded automatically)
│   ├── review_centric.json (generated)
│   └── shingle_centric.json (generated)
└── results/
    └── [generated visualizations and statistics]
```

---

## Algorithm Details

### Shingling Process (k=5)
```python
def generate_shingles(text, n=5):
    words = text.split()
    return [' '.join(words[i:i+n]) for i in range(len(words) - n + 1)]
```

**Example**:
- Input: `"the quick brown fox jumps"`
- Shingles:
  - `"the quick brown fox jumps"`

### Hashing (SHA-1)
```python
def hash_shingles(shingles):
    return [sha1(shingle.encode('utf-8')).hexdigest() for shingle in shingles]
```

**Output**: 40-character hexadecimal hash per shingle

### Winnowing Algorithm (w=4)
```python
def winnowing(hashes, window_size=4):
    fingerprints = set()
    for i in range(len(hashes) - window_size + 1):
        window = hashes[i:i+window_size]
        min_hash = min(window)
        fingerprints.add(min_hash)
    return list(fingerprints)
```

**Purpose**: To reduce storage by selecting representative fingerprints

---

## Performance Benchmarking

### Metrics Collected

1. **Search Time**:
   - Average, Median (P50), P95, P99
   - Measured over 1,000 queries

2. **Storage**:
   - Collection data size (MB)
   - Index size (MB)
   - Documents per shingle

3. **Distribution**:
   - Documents per shard
   - Balance percentage

### Sample Query
```python
# Search for reviews with 15 matching shingles
query_shingles = [random.choice(all_shingles) for _ in range(15)]

pipeline = [
    {"$match": {"shingle_hash": {"$in": query_shingles}}},
    {"$unwind": "$documents"},
    {"$group": {
        "_id": "$documents",
        "shared_shingles": {"$sum": 1}
    }},
    {"$sort": {"shared_shingles": -1}},
    {"$limit": 5}
]
```

---

## Results

### Example Performance Comparison

| Configuration | Avg Search Time (ms) | P95 Time (ms) | Collection Size (MB) |
|--------------|---------------------|--------------|---------------------|
| No Shards (M10) | 350.25 | 420.15 | 1,639.36 |
| 3 Shards (M30) | 180.42 | 210.88 | 1,639.36 |
| 6 Shards (M30) | 268.05 | 260.57 | 1,639.36 |

### Key Observations

- **3 Shards**: ~48% latency reduction vs no sharding
- **6 Shards**: Slight increase due to coordination overhead for this dataset size
- **Optimal**: 3 shards for 12M documents; 6+ shards beneficial at 50M+ scale

---

## Troubleshooting

### Common Issues

#### **1. Connection Timeout**
```
Error: ServerSelectionTimeoutError
```
**Solution**:
- Check Network Access in Atlas (allow 0.0.0.0/0)
- Verify internet connection
- Test connection string in a new cell

#### **2. Authentication Failed**
```
Error: Authentication failed
```
**Solution**:
- Verify the username/password in the connection string
- Check that the user has "Atlas admin" privileges
- Ensure that special characters in the password are URL-encoded

#### **3. Sharding Not Enabled**
```
Error: Please create an index that starts with the proposed shard key
```
**Solution**:
- Run the step to create `shingle_shard_key` index
- Verify cluster is M30+ (sharding requires dedicated clusters)

#### **4. Slow Data Distribution**
```
Only 1 shard has data after sharding
```
**Solution**:
- Wait 10-15 minutes for initial balancing
- Check **Metrics** → **Opcounters** for migration activity
- Run the step to verify the distribution

#### **5. Out of Memory**
```
Error: Memory allocation error
```
**Solution**:
- Reduce the batch size: `BATCH_SIZE = 100` (instead of 500)
- Process fewer reviews initially (test with 100K)
- Upgrade Colab to Pro for more RAM

#### **6. Index Creation Failed**
```
Error: Index already exists
```
**Solution**:
- This is informational, not an error - it shows us that the indexes already exist
- Indexes are idempotent (safe to recreate)

---

## Advanced Configuration

### Adjusting Parameters

```python
# Shingling
K = 5  # Shingle size (3-7 recommended)
W = 4  # Winnowing window (3-5 optimal)

# Database
BATCH_SIZE = 500  # Upload batch size
DB_NAME = "YELP"
COLLECTION_NAME = "REVIEWS"

# Testing
NUM_QUERIES = 1000  # Number of search tests
SHINGLES_PER_QUERY = 15  # Query complexity
```

### Custom Index Strategies

```python
# Text search optimization
collection.create_index([("documents", 1)])

# Range queries
collection.create_index([("shingle_hash", 1), ("documents", 1)])
```

---

## Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Test with all 3 configurations
4. Submit a pull request

---

## License

This project uses the [Yelp Dataset](https://www.yelp.com/dataset), which is subject to Yelp's Terms of Service.

---

## Acknowledgments

- **Yelp Dataset**: For providing real-world review data
- **MongoDB Atlas**: For scalable document storage
- **Winnowing Algorithm**: Schleimer, Wilkerson, and Aiken (2003)

---

## Contact

For questions or issues:
- Open an issue on GitHub
- Contact: Data Mining Group 3 2025

---

**Note from Group 3**: Always secure your MongoDB credentials. Never commit your connection strings with passwords visible to public repositories. Use environment variables or secret management tools in production.
