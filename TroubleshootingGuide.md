# Troubleshooting Guide

This document covers common issues encountered during the setup and operation of the IRCTC PubSub to BigQuery streaming pipeline.

## 📑 Table of Contents

1. [Authentication Issues](#authentication-issues)
2. [Pub/Sub Issues](#pubsub-issues)
3. [Dataflow Issues](#dataflow-issues)
4. [BigQuery Issues](#bigquery-issues)
5. [Python Environment Issues](#python-environment-issues)
6. [Protocol Buffer Issues](#protocol-buffer-issues)

---

## 🔐 Authentication Issues

### Issue 1: "Project has been deleted" Error

**Error Message:**
```
403 Project #852119498771 has been deleted. [reason: "CONSUMER_INVALID"]
```

**Cause:** Environment variable `GOOGLE_APPLICATION_CREDENTIALS` points to an old/deleted project's service account key.

**Solution:**

```bash
# Check current environment variable
echo $GOOGLE_APPLICATION_CREDENTIALS

# Unset the environment variable
unset GOOGLE_APPLICATION_CREDENTIALS

# Re-authenticate with correct project
gcloud auth application-default login --project=YOUR_PROJECT_ID

# Set default project
gcloud config set project YOUR_PROJECT_ID
```

**Permanent Fix:**

Edit your shell configuration file:

```bash
# For zsh (macOS default)
nano ~/.zshrc

# For bash
nano ~/.bash_profile
```

Remove or comment out the line:
```bash
# export GOOGLE_APPLICATION_CREDENTIALS="/path/to/old/key.json"
```

Save and reload:
```bash
source ~/.zshrc  # or source ~/.bash_profile
```

---

### Issue 2: "ModuleNotFoundError: No module named 'google'"

**Error Message:**
```
ModuleNotFoundError: No module named 'google'
```

**Cause:** Google Cloud libraries not installed or Python/pip mismatch.

**Solution:**

```bash
# Check Python and pip versions
which python
pip --version

# If they point to different locations, use:
python -m pip install google-cloud-pubsub

# Or reinstall namespace packages
pip uninstall -y google-cloud-pubsub google
pip install --upgrade google-cloud-pubsub
```

---

### Issue 3: Python Alias Mismatch

**Symptoms:** Packages install successfully but script can't find them.

**Cause:** `python` command aliased to system Python while packages installed to Anaconda.

**Solution:**

```bash
# Check alias
which python
pip --version

# Remove alias
unalias python

# Or use full path
/opt/anaconda3/bin/python script.py

# Or use python3 explicitly
python3 script.py
```

---

## 📨 Pub/Sub Issues

### Issue 4: "Topic not found"

**Error Message:**
```
google.api_core.exceptions.NotFound: 404 Resource not found
```

**Solution:**

```bash
# List existing topics
gcloud pubsub topics list

# Create topic if missing
gcloud pubsub topics create irctc-data

# Verify creation
gcloud pubsub topics describe irctc-data
```

---

### Issue 5: "Permission denied" when publishing

**Error Message:**
```
403 Permission denied on topic
```

**Solution:**

```bash
# Check your permissions
gcloud projects get-iam-policy YOUR_PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:YOUR_EMAIL"

# Add Pub/Sub Publisher role if needed
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="user:YOUR_EMAIL" \
  --role="roles/pubsub.publisher"
```

---

### Issue 6: Messages not appearing in subscription

**Solution:**

```bash
# Check if subscription exists
gcloud pubsub subscriptions list

# Create subscription if missing
gcloud pubsub subscriptions create irctc-data-sub \
  --topic=irctc-data

# Pull messages to test
gcloud pubsub subscriptions pull irctc-data-sub --limit=5 --auto-ack
```

---

## 🌊 Dataflow Issues

### Issue 7: "Invalid Protobuf Schema File"

**Error Message:**
```
java.lang.IllegalArgumentException: Schema file is not valid.
com.google.protobuf.InvalidProtocolBufferException$InvalidWireTypeException
```

**Cause:** Uploaded `.proto` text file instead of compiled `.pb` binary.

**Solution:**

```bash
# Install protoc compiler
# On macOS:
brew install protobuf

# On Cloud Shell (already installed):
protoc --version

# Compile the schema
protoc --descriptor_set_out=irctc_schema.pb \
  --include_imports \
  irctc_schema.proto

# Upload binary file
gsutil cp irctc_schema.pb gs://YOUR_BUCKET/schemas/

# Use in Dataflow: gs://YOUR_BUCKET/schemas/irctc_schema.pb
```

---

### Issue 8: "Dataset not found in location US"

**Error Message:**
```
Not found: Dataset project-id:dataset was not found in location US
```

**Cause:** Dataset created in different region than specified in query.

**Solution:**

```bash
# Check dataset location
bq show --format=prettyjson YOUR_PROJECT:irctc_dwh | grep location

# Specify location in BigQuery query settings
# Or recreate dataset in correct region
bq mk --location=US irctc_dwh
```

---

### Issue 9: Dataflow job fails immediately

**Troubleshooting Steps:**

1. Check Dataflow logs in Cloud Console
2. Verify service account permissions:

```bash
# Dataflow service account needs:
# - Dataflow Worker
# - Pub/Sub Subscriber
# - BigQuery Data Editor
# - Storage Object Viewer

# Add permissions if needed
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/dataflow.worker"
```

3. Verify network settings (VPC, firewall rules)
4. Check quota limits in GCP Console

---

## 📊 BigQuery Issues

### Issue 10: "Table not found" error

**Solution:**

```bash
# List tables in dataset
bq ls irctc_dwh

# Create table if missing
bq query --use_legacy_sql=false < sql/create_bigquery_table.sql

# Or manually create:
bq mk --table irctc_dwh.irctc_stream_tb schema.json
```

---

### Issue 11: Schema mismatch errors

**Solution:**

```bash
# Check current schema
bq show --schema irctc_dwh.irctc_stream_tb

# Update table schema (add new fields only)
bq update irctc_dwh.irctc_stream_tb schema.json

# Or drop and recreate (CAUTION: loses data)
bq rm -f -t irctc_dwh.irctc_stream_tb
bq mk --table irctc_dwh.irctc_stream_tb schema.json
```

---

### Issue 12: Export to Cloud Storage fails

**Error Message:**
```
Not found: Dataset project-040088dd-8c9a-464e-96f:lyft was not found in location US
```

**Solution:**

Add location parameter to query:

```sql
-- In Query Settings, set Processing Location to match dataset location
-- Or use format:
EXPORT DATA OPTIONS(
  uri='gs://bucket/path/*.json',
  format='JSON',
  overwrite=true
) AS
SELECT * FROM `project.dataset.table`;
```

---

## 🐍 Python Environment Issues

### Issue 13: DateTime deprecation warning

**Warning Message:**
```
DeprecationWarning: datetime.datetime.utcnow() is deprecated
```

**Solution:**

Replace in your Python code:

```python
# Old (deprecated)
from datetime import datetime
inserted_at = datetime.utcnow()

# New (recommended)
from datetime import datetime, timezone
inserted_at = datetime.now(timezone.utc)
```

---

### Issue 14: Anaconda vs System Python conflicts

**Symptoms:** Commands work in one terminal but not another.

**Solution:**

```bash
# Check which Python is active
which python
which pip

# Ensure using same Python
python --version
pip --version

# Use full paths if needed
/opt/anaconda3/bin/python script.py

# Or activate conda environment properly
conda activate base
```

---

## 🔧 Protocol Buffer Issues

### Issue 15: "protoc: command not found"

**Solution:**

```bash
# On macOS
brew install protobuf

# On Linux (Ubuntu/Debian)
sudo apt-get install protobuf-compiler

# On Cloud Shell (already available)
protoc --version

# Verify installation
which protoc
```

---

### Issue 16: Proto compilation errors

**Error Message:**
```
irctc_schema.proto: File not found
```

**Solution:**

```bash
# Ensure you're in correct directory
ls -la irctc_schema.proto

# Use correct syntax
protoc --descriptor_set_out=output.pb \
  --include_imports \
  input.proto

# Check proto file syntax
protoc --decode_raw < irctc_schema.pb
```

---

## 🆘 General Debugging Tips

### Enable Detailed Logging

```python
import logging
logging.basicConfig(level=logging.DEBUG)
```

### Check GCP Quotas

```bash
gcloud compute project-info describe --project=YOUR_PROJECT_ID
```

### Verify Service APIs are Enabled

```bash
# Enable required APIs
gcloud services enable pubsub.googleapis.com
gcloud services enable dataflow.googleapis.com
gcloud services enable bigquery.googleapis.com
gcloud services enable storage-api.googleapis.com
```

### Monitor Resource Usage

```bash
# Check Pub/Sub metrics
gcloud pubsub topics list --format="table(name,labels)"

# Check Dataflow jobs
gcloud dataflow jobs list --status=active

# Check BigQuery jobs
bq ls -j --max_results=10
```

---

## 📞 Getting Help

If issues persist:

1. Check GCP Console logs
2. Review error messages carefully
3. Search GCP documentation
4. Check Stack Overflow
5. Open GitHub issue with:
   - Error message
   - Steps to reproduce
   - Environment details
   - Logs/screenshots

---

## 🔗 Useful Commands Reference

```bash
# Authentication
gcloud auth list
gcloud config list

# Pub/Sub
gcloud pubsub topics list
gcloud pubsub subscriptions list

# BigQuery
bq ls
bq show dataset.table

# Cloud Storage
gsutil ls
gsutil ls -l gs://bucket/path/

# Dataflow
gcloud dataflow jobs list
gcloud dataflow jobs describe JOB_ID

# Service Account
gcloud iam service-accounts list
gcloud projects get-iam-policy PROJECT_ID
```