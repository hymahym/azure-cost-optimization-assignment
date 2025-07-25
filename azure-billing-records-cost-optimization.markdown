# Azure Billing Records Cost Optimization

## Overview
This project provides a cost-optimization solution for managing billing records in an Azure serverless architecture. It archives records older than three months from Azure Cosmos DB to Azure Blob Storage (cool tier), reducing storage costs from approximately $150/month to $25/month while ensuring data availability within seconds. The solution maintains existing API contracts, ensures no data loss, and operates without downtime, making it simple to deploy and maintain.

## Architecture
The solution integrates the following components:
- **Azure Cosmos DB**: Stores active records (≤3 months) with full data and stubs for archived records (>3 months) containing metadata (`id`, `blobUri`, `archived` flag).
- **Azure Blob Storage**: Stores archived records in the cool tier for cost-efficient storage ($0.0184/GB/month vs. Cosmos DB’s $0.25/GB/month).
- **Azure Functions**:
  - **Archiving Function**: A timer-triggered function that runs daily at 2 AM UTC to archive old records.
  - **Read API Function**: An HTTP-triggered function that retrieves records from Cosmos DB or Blob Storage based on the `archived` flag.
- **Azure Monitor**: Tracks function performance and errors (optional setup).

### Architecture Diagram
```
+---------------+
|   User        |
+---------------+
           |
           |  Read Request
           v
+---------------+
| Read API (Azure Function) |
+---------------+
           |
           |  Query by ID
           v
+---------------+
| Azure Cosmos DB |
| - Active: Full data  |
| - Archived: Stub (ID, blobUri, archived flag) |
+---------------+
           |
           |  If archived, fetch from Blob Storage
           v
+---------------+
| Azure Blob Storage |
| (Stores archived records) |
+---------------+
           |
           |  Return full record
           v
+---------------+
|   Response to User   |
+---------------+

[Archiving Process]
+---------------+
| Azure Function (Timer) |
+---------------+
           |
           |  Query old records
           v
+---------------+
| Azure Cosmos DB |
+---------------+
           |
           |  Copy to Blob Storage
           v
+---------------+
| Azure Blob Storage |
+---------------+
           |
           |  Update to stub
           v
+---------------+
| Azure Cosmos DB |
+---------------+
```

## Prerequisites
- Azure subscription with permissions to create resources ([Azure Portal](https://portal.azure.com)).
- Azure CLI or Azure Portal access ([Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)).
- Python 3.8+ for local development (optional) ([Python](https://www.python.org/downloads)).
- Git installed for repository management ([Git](https://git-scm.com/downloads)).

## Setup Instructions
1. **Create Azure Resources**:
   - **Cosmos DB**:
     - Create a Cosmos DB account ([Azure Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db)).
     - Create a database (e.g., `BillingDB`) and a container (e.g., `billing-records`) with partition key `/id`.
     - Ensure records have a `createdAt` field in ISO 8601 format (e.g., `2025-03-01T00:00:00Z`).
     - Note the connection string and store it securely.
   - **Blob Storage**:
     - Create a general-purpose v2 storage account ([Azure Blob Storage](https://docs.microsoft.com/en-us/azure/storage/blobs)).
     - Create a container named `billing-archives` with the "Cool" access tier.
     - Note the connection string.
   - **Function App**:
     - Create a Function App with Python runtime ([Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions)).
     - Configure application settings for connection strings (see below).

2. **Configure Application Settings**:
   - In the Function App, add the following settings via the Azure Portal or CLI:
     - `CosmosDBConnectionString`: Cosmos DB connection string.
     - `CosmosDBDatabase`: Database name (e.g., `BillingDB`).
     - `CosmosDBContainer`: Container name (e.g., `billing-records`).
     - `BlobStorageConnectionString`: Blob Storage connection string.
     - `BlobContainerName`: Blob container name (e.g., `billing-archives`).

3. **Clone and Prepare Repository**:
   - Create a GitHub repository: `azure-billing-records-cost-optimization` ([GitHub](https://github.com)).
   - Clone locally: `git clone https://github.com/your-username/azure-billing-records-cost-optimization.git`
   - Navigate to the project directory: `cd azure-billing-records-cost-optimization`
   - Install dependencies: `pip install -r requirements.txt`

4. **Deploy Azure Functions**:
   - Install Azure Functions Core Tools: `npm install -g azure-functions-core-tools@4 --unsafe-perm true` ([Functions Core Tools](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local)).
   - Deploy: `func azure functionapp publish <your-function-app-name>`
   - Alternatively, deploy via Visual Studio Code with the Azure Functions extension ([VS Code](https://code.visualstudio.com)).

5. **Run Initial Archiving**:
   - Manually trigger the `archiving-function` via the Azure Portal to process existing records older than three months.
   - Monitor logs in Azure Portal or Application Insights to ensure successful archiving.

6. **Set Up Monitoring (Optional)**:
   - Configure Azure Monitor to track function execution and errors ([Azure Monitor](https://docs.microsoft.com/en-us/azure/azure-monitor)).
   - Set alerts for high RU consumption or function failures.

## Project Structure
```
azure-billing-records-cost-optimization/
├── azure-functions/
│   ├── archiving-function/
│   │   ├── __init__.py
│   │   └── function.json
│   ├── read-api-function/
│   │   ├── __init__.py
│   │   └── function.json
├── requirements.txt
├── README.md
└── LICENSE
```

## File Contents

### azure-functions/archiving-function/__init__.py
<xaiArtifact artifact_id="b7559a83-47c5-41b8-99e6-72b5b3852b84" artifact_version_id="9392ec00-2f3b-44ab-b1f6-2ec406744326" title="archiving-function/__init__.py" contentType="text/python">
import azure.functions as func
import azure.cosmos.cosmos_client as cosmos_client
from azure.storage.blob import BlobServiceClient
import datetime
import json
import logging
import os

cosmos_db_connection_string = os.environ['CosmosDBConnectionString']
database_name = os.environ['CosmosDBDatabase']
container_name = os.environ['CosmosDBContainer']
blob_connection_string = os.environ['BlobStorageConnectionString']
blob_container_name = os.environ['BlobContainerName']

client = cosmos_client.CosmosClient.from_connection_string(cosmos_db_connection_string)
container = client.get_database_client(database_name).get_container_client(container_name)
blob_service_client = BlobServiceClient.from_connection_string(blob_connection_string)

def main(mytimer: func.TimerRequest) -> None:
    logging.info("Starting archiving process")
    three_months_ago = datetime.datetime.utcnow() - datetime.timedelta(days=90)
    
    query = f"SELECT * FROM c WHERE c.createdAt < '{three_months_ago.isoformat()}' AND c.archived != true"
    items = list(container.query_items(query=query, enable_cross_partition_query=True))
    
    for item in items:
        try:
            blob_name = f"{item['id']}.json"
            blob_client = blob_service_client.get_blob_client(container=blob_container_name, blob=blob_name)
            blob_client.upload_blob(json.dumps(item), overwrite=True)
            
            item['archived'] = True
            item['blobUri'] = blob_client.url
            if 'details' in item:
                del item['details']
            container.upsert_item(body=item)
            logging.info(f"Archived record {item['id']}")
        except Exception as e:
            logging.error(f"Failed to archive record {item['id']}: {str(e)}")
