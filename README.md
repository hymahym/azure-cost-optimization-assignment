# Azure Billing Records Cost Optimization

## Overview
This project provides a cost-optimization solution for managing billing records in an Azure serverless architecture. It archives records older than three months from Azure Cosmos DB to Azure Blob Storage (cool tier), reducing storage costs while ensuring data availability. The system maintains existing API contracts, ensures no data loss, and operates without downtime.

## Architecture
- **Azure Cosmos DB**: Stores active records (≤3 months) with full data and stubs for archived records (>3 months) containing metadata (`id`, `blobUri`, `archived` flag).
- **Azure Blob Storage**: Stores archived records in the cool tier for cost-efficient storage.
- **Azure Functions**:
  - **Archiving Function**: A timer-triggered function that runs daily to archive old records.
  - **Read API Function**: An HTTP-triggered function that retrieves records from Cosmos DB or Blob Storage.
- **Azure Monitor**: Tracks function performance and errors (optional setup).

## Prerequisites
- Azure subscription with permissions to create resources.
- Azure CLI or Azure Portal access.
- Python 3.8+ for local development (optional).
- Git installed for repository management.

## Setup Instructions
1. **Create Azure Resources**:
   - **Cosmos DB**:
     - Create a Cosmos DB account.
     - Create a database (e.g., `BillingDB`) and a container (e.g., `billing-records`) with partition key `/id`.
     - Note the connection string and store it securely.
   - **Blob Storage**:
     - Create a general-purpose v2 storage account.
     - Create a container named `billing-archives` with the "Cool" access tier.
     - Note the connection string.
   - **Function App**:
     - Create a Function App with Python runtime.
     - Configure application settings for connection strings (see below).

2. **Configure Application Settings**:
   - In the Function App, add the following settings:
     - `CosmosDBConnectionString`: Cosmos DB connection string.
     - `CosmosDBDatabase`: Database name (e.g., `BillingDB`).
     - `CosmosDBContainer`: Container name (e.g., `billing-records`).
     - `BlobStorageConnectionString`: Blob Storage connection string.
     - `BlobContainerName`: Blob container name (e.g., `billing-archives`).

3. **Clone and Prepare Repository**:
   - Clone this repository: `git clone https://github.com/your-username/azure-billing-records-cost-optimization.git`
   - Navigate to the project directory: `cd azure-billing-records-cost-optimization`
   - Install dependencies: `pip install -r requirements.txt`

4. **Deploy Azure Functions**:
   - Use Azure CLI: `func azure functionapp publish <your-function-app-name>`
   - Alternatively, deploy via Visual Studio Code with the Azure Functions extension.

5. **Run Initial Archiving**:
   - Manually trigger the `archiving-function` to process existing records older than three months.
   - Monitor logs to ensure successful archiving.

6. **Set Up Monitoring (Optional)**:
   - Configure Azure Monitor to track function execution and errors.
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

## Usage
- **Write API**: No changes required; new records are written to Cosmos DB as usual.
- **Read API**: Access via HTTP GET request: `https://<your-function-app-name>.azurewebsites.net/api/read-api-function?id=<record-id>`
- **Archiving**: Runs daily at 2 AM UTC, archiving records older than three months.

## Cost Analysis
| Component | Before (600 GB in Cosmos DB) | After (60 GB in Cosmos DB, 540 GB in Blob Storage) |
|-----------|-----------------------------|--------------------------------------------------|
| **Cosmos DB Storage** | 600 GB * $0.25/GB = $150/month | 60 GB * $0.25/GB = $15/month |
| **Blob Storage (Cool)** | $0 | 540 GB * $0.0184/GB = $9.94/month |
| **Total** | $150/month | ~$25/month |

Savings: ~$125/month, assuming 90% of data is archived.

## License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
