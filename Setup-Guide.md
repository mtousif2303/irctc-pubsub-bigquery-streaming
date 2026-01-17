# Setup Guide

Complete step-by-step instructions to set up the IRCTC PubSub to BigQuery streaming pipeline.

## 📋 Prerequisites Checklist

- [ ] Google Cloud Platform account
- [ ] Billing enabled on your GCP project
- [ ] `gcloud` CLI installed ([Installation Guide](https://cloud.google.com/sdk/docs/install))
- [ ] Python 3.8+ installed
- [ ] Git installed
- [ ] Basic knowledge of GCP, Python, and SQL

---

## 🚀 Step 1: GCP Project Setup

### 1.1 Create or Select Project

```bash
# Create new project (optional)
gcloud projects create YOUR_PROJECT_ID --name="IRCTC Pipeline"

# Set as default project
gcloud config set project YOUR_PROJECT_ID

# Verify
gcloud config get-value project
```

### 1.2 Enable Required APIs

```bash
# Enable all required APIs
gcloud services enable pubsub.googleapis.com
gcloud services enable dataflow.googleapis.com
gcloud services enable bigquery.googleapis.com
gcloud services enable storage-api.googleapis.com
gcloud services enable compute.googleapis.com
```

Verify enabled services:
```bash
gcloud services list --enabled
```

### 1.3 Set Up Billing

Ensure billing is enabled for your project via [GCP Console](https://console.cloud.google.com/billing).

---

## 🔐 Step 2: Authentication Setup

### 2.1 Login to GCP

```bash
# Login with your Google account
gcloud auth login

# Set application default credentials
gcloud auth application-default login
```

### 2.2 Create Service Account (Optional)

For production, use a dedicated service account:

```bash
# Create service account
gcloud iam service-accounts create irctc-pipeline-sa \
  --display-name="IRCTC Pipeline Service Account"

# Grant necessary roles
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:irctc-pipeline-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"

gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:irctc-pipeline-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/pubsub.subscriber"

gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:irctc-pipeline-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataEditor"

# Create and download key
gcloud iam service-accounts keys create ~/irctc-key.json \
  --iam-account=irctc-pipeline-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com

# Set environment variable (for service account usage)
export GOOGLE_APPLICATION_CREDENTIALS="$HOME/irctc-key.json"
```

---

## 📦 Step 3: Python Environment Setup

### 3.1 Create Virtual Environment

**Option A: Using Anaconda (Recommended)**

```bash
# Create new environment
conda create -n irctc-pipeline python=3.12 -y

# Activate environment
conda activate irctc-pipeline
```

**Option B: Using venv**

```bash
# Create virtual environment
python -m venv irctc-env

# Activate (macOS/Linux)
source irctc-env/bin/activate

# Activate (Windows)
irctc-env\Scripts\activate
```

### 3.2 Install Dependencies

```bash
# Upgrade pip
pip install --upgrade pip

# Install required packages
pip install google-cloud-pubsub
pip install google-cloud-bigquery
pip install google-cloud-storage

# Verify installation
python -c "from google.cloud import pubsub_v1; print('✓ Pub/Sub installed')"
python -c "from google.cloud import bigquery; print('✓ BigQuery installed')"
```

---

## ☁️ Step 4: Cloud Storage Setup

### 4.1 Create Storage Bucket

```bash
# Create bucket
gsutil mb -l US gs://YOUR_BUCKET_NAME

# Verify
gsutil ls

# Create schemas folder
gsutil mkdir gs://YOUR_BUCKET_NAME/schemas/
```

### 4.2 Set Bucket Permissions

```bash
# Make bucket accessible to Dataflow service account
gsutil iam ch serviceAccount:YOUR_PROJECT_NUMBER-compute@developer.gserviceaccount.com:objectViewer \
  gs://YOUR_BUCKET_NAME
```

---

## 📨 Step 5: Pub/Sub Setup

### 5.1 Create Topic

```bash
# Create Pub/Sub topic
gcloud pubsub topics create irctc-data

# Verify
gcloud pubsub topics describe irctc-data
```

### 5.2 Create Subscription

```bash
# Create subscription
gcloud pubsub subscriptions create irctc-data-sub \
  --topic=irctc-data \
  --ack-deadline=60

# Verify
gcloud pubsub subscriptions describe irctc-data-sub
```

### 5.3 Test Pub/Sub

```bash
# Publish test message
gcloud pubsub topics publish irctc-data \
  --message='{"test": "message"}'

# Pull message
gcloud pubsub subscriptions pull irctc-data-sub \
  --limit=1 \
  --auto-ack
```

---

## 📊 Step 6: BigQuery Setup

### 6.1 Create Dataset

```bash
# Create dataset
bq mk --dataset \
  --location=US \
  --description="IRCTC Data Warehouse" \
  YOUR_PROJECT_ID:irctc_dwh

# Verify
bq ls
```

### 6.2 Create Table

Create file `create_table.sql`:

```sql
CREATE TABLE `YOUR_PROJECT_ID.irctc_dwh.irctc_stream_tb` (
  row_key STRING,
  name STRING,
  age INT64,
  email STRING,
  join_date DATE,
  last_login TIMESTAMP,
  loyalty_points INT64,
  account_balance FLOAT64,
  is_active BOOL,
  inserted_at TIMESTAMP,
  updated_at TIMESTAMP,
  loyalty_status STRING,
  account_age_days INT64
);
```

Execute:

```bash
bq query --use_legacy_sql=false < create_table.sql

# Verify
bq show irctc_dwh.irctc_stream_tb
```

---

## 🔧 Step 7: Protocol Buffer Setup

### 7.1 Install Protobuf Compiler

**macOS:**
```bash
brew install protobuf
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt-get update
sudo apt-get install protobuf-compiler
```

**Windows:**
Download from [GitHub Releases](https://github.com/protocolbuffers/protobuf/releases)

**Or use Google Cloud Shell** (protoc pre-installed)

### 7.2 Create Proto Schema

Create file `irctc_schema.proto`:

```protobuf
syntax = "proto3";

package irctc;

message IrctcRecord {
  string row_key = 1;
  string name = 2;
  int64 age = 3;
  string email = 4;
  string join_date = 5;
  int64 last_login = 6;
  int64 loyalty_points = 7;
  double account_balance = 8;
  bool is_active = 9;
  int64 inserted_at = 10;
  int64 updated_at = 11;
  string loyalty_status = 12;
  int64 account_age_days = 13;
}
```

### 7.3 Compile and Upload

```bash
# Compile proto file
protoc --descriptor_set_out=irctc_schema.pb \
  --include_imports \
  irctc_schema.proto

# Verify compilation
ls -lh irctc_schema.pb

# Upload to Cloud Storage
gsutil cp irctc_schema.pb gs://YOUR_BUCKET_NAME/schemas/

# Verify upload
gsutil ls gs://YOUR_BUCKET_NAME/schemas/
```

---

## 🌊 Step 8: Dataflow Pipeline Setup

### 8.1 Using GCP Console (Easiest)

1. Navigate to [Dataflow](https://console.cloud.google.com/dataflow)
2. Click **"Create job from template"**
3. Select **"Pub/Sub Proto to BigQuery"** template
4. Fill in parameters:

   **Required Parameters:**
   - **Job name**: `irctc-pubsub-to-bigquery`
   - **Regional endpoint**: `us-central1`
   
   **Source:**
   - **Pub/Sub input subscription**: `projects/YOUR_PROJECT_ID/subscriptions/irctc-data-sub`
   
   **Target:**
   - **BigQuery output table**: `YOUR_PROJECT_ID:irctc_dwh.irctc_stream_tb`
   - **Output Pub/Sub topic**: `projects/YOUR_PROJECT_ID/topics/irctc-output` (optional)
   
   **Required Parameters:**
   - **Cloud Storage Path to Proto Schema**: `gs://YOUR_BUCKET_NAME/schemas/irctc_schema.pb`
   - **Full Proto Message Name**: `irctc.IrctcRecord`

5. Click **"Run job"**

### 8.2 Using gcloud (Advanced)

```bash
gcloud dataflow flex-template run "irctc-pubsub-to-bq-$(date +%Y%m%d-%H%M%S)" \
  --template-file-gcs-location=gs://dataflow-templates-us-central1/latest/flex/PubSub_Proto_to_BigQuery_Xlang \
  --region=us-central1 \
  --parameters \
inputSubscription=projects/YOUR_PROJECT_ID/subscriptions/irctc-data-sub,\
outputTableSpec=YOUR_PROJECT_ID:irctc_dwh.irctc_stream_tb,\
protoSchemaPath=gs://YOUR_BUCKET_NAME/schemas/irctc_schema.pb,\
fullMessageName=irctc.IrctcRecord
```

---

## 🧪 Step 9: Testing the Pipeline

### 9.1 Clone Repository

```bash
git clone https://github.com/yourusername/irctc-pubsub-bigquery-streaming-pipeline.git
cd irctc-pubsub-bigquery-streaming-pipeline
```

### 9.2 Update Configuration

Edit `scripts/irctc_mock_data_to_pubsub.py`:

```python
# Update these values
project_id = "YOUR_PROJECT_ID"
topic_id = "irctc-data"
```

### 9.3 Run Mock Data Generator

```bash
python scripts/irctc_mock_data_to_pubsub.py
```

Expected output:
```
Data -> {"row_key": "...", "name": "...", ...}
Published message ID: 123456789
Published message ID: 123456790
...
Published 20 messages successfully.
```

### 9.4 Verify Data in BigQuery

```bash
# Check row count
bq query "SELECT COUNT(*) as total FROM \`YOUR_PROJECT_ID.irctc_dwh.irctc_stream_tb\`"

# View sample records
bq query "SELECT * FROM \`YOUR_PROJECT_ID.irctc_dwh.irctc_stream_tb\` LIMIT 10"

# Check loyalty status distribution
bq query "SELECT loyalty_status, COUNT(*) as count FROM \`YOUR_PROJECT_ID.irctc_dwh.irctc_stream_tb\` GROUP BY loyalty_status"
```

---

## 🔍 Step 10: Monitoring

### 10.1 Pub/Sub Monitoring

```bash
# Check topic metrics
gcloud pubsub topics describe irctc-data

# Monitor subscription
gcloud pubsub subscriptions describe irctc-data-sub
```

View in Console: [Pub/Sub Monitoring](https://console.cloud.google.com/cloudpubsub)

### 10.2 Dataflow Monitoring

```bash
# List active jobs
gcloud dataflow jobs list --status=active

# Get job details
gcloud dataflow jobs describe JOB_ID
```

View in Console: [Dataflow Jobs](https://console.cloud.google.com/dataflow/jobs)

### 10.3 BigQuery Monitoring

```bash
# Recent jobs
bq ls -j --max_results=10

# Check streaming inserts
bq show --format=prettyjson irctc_dwh.irctc_stream_tb | grep streamingBuffer
```

View in Console: [BigQuery Console](https://console.cloud.google.com/bigquery)

---

## 🎯 Next Steps

1. **Set up alerts**: Configure Cloud Monitoring alerts for failures
2. **Implement CI/CD**: Automate deployment with Cloud Build
3. **Add data validation**: Implement data quality checks
4. **Create dashboards**: Build Looker Studio reports
5. **Optimize costs**: Review and adjust resource allocation

---

## 📚 Additional Resources

- [GCP Dataflow Documentation](https://cloud.google.com/dataflow/docs)
- [Pub/Sub Best Practices](https://cloud.google.com/pubsub/docs/best-practices)
- [BigQuery Best Practices](https://cloud.google.com/bigquery/docs/best-practices)
- [Protocol Buffers Guide](https://developers.google.com/protocol-buffers)

---

## ⚠️ Important Notes

- Replace `YOUR_PROJECT_ID` with your actual GCP project ID
- Replace `YOUR_BUCKET_NAME` with your Cloud Storage bucket name
- Ensure billing is enabled to avoid service interruptions
- Monitor costs regularly in the [Billing Console](https://console.cloud.google.com/billing)
- Follow security best practices for production deployments

---

## 🆘 Need Help?

If you encounter issues, check:
1. [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)
2. GCP Console logs
3. Open a GitHub issue with error details