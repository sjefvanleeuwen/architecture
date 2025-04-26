# Overview
NOTE: This process supersedes the ([Greenflux CDRs Import Process](https://dev.azure.com/enecomanagedcloud/eMobility/_wiki/wikis/eMobility.wiki/49305/Greenflux-CDRs-Import-process)).

This system is responsible for importing priced charge sessions (CDRs) from TandemDrive external system. The priced charge sessions support `Dynamic electricity tariffs` - instead of paying a fixed price throughout the annual contract, a customer pays a different rate, which typically varies by the hour. This pricing model helps customers reduce energy costs by charging during cheaper periods. 

This import is essential for quick insights and usage analysis (reporting in the Eneco eMobility Portal). It also supports exporting customer-specific data for invoice validation and transparency.

## Data capacity
We expect a similar volume of data and measurements. While the data model differs, the core information, such as energy consumption and charging metrics, remains the same. The goal is to maintain continuity in reporting and analytics - the output should be functionally equivalent.

## Architecture

This system is designed as a `Real-time processing of big data in motion` architecture. While the data volume does not strictly qualify as Big Data, it is still big enough to require a scalable approach. 

### Preferred Approach - Azure Storage
This architecture needs to structure blob containers in partitioned way and transform data to Parquet format and it does require CosmosDB for the read part.

![TandemDrive-CDRs-Import-Simplified.drawio.png](/.attachments/TandemDrive-CDRs-Import-Simplified.drawio-2f97d6a1-43f6-499f-83bb-7027d98c6e71.png)

- TandemDrive sends CDRs to Azure API Management via webhook.
- APIM pushes the data directly into Azure Blob Storage without a backend.
- Azure Storage holds the raw data, organized for easy access by date or ID.
- Azure Synapse runs SQL-like queries on the blob data using serverless pools.
- Backend APIs invoke queries using Synapse, and sends results to the Web Portal.

Blob container should be organized as shown below:
```bash
/cdrs/year=2025/month=04/day=22/locationId=LOC123/evseId=EVSE456/file.parquet
```

### Alternate Approach - Event Hubs
The overall design resembles a `Kappa architecture`, focusing on stream processing without the need for separate batch and real-time layers.

![HLA-TandemDrive-CDRs-Import-Process.drawio.png](/.attachments/HLA-TandemDrive-CDRs-Import-Process.drawio-379e5d9d-9f7b-4519-b1d6-2044a18e3844.png)

- The system ingests CDRs via Azure API Management (APIM), which exposes a webhook. APIM sends data directly to Event Hubs, with no backend in between.
- Event Hubs acts as the streaming buffer, supporting replay/reprocessing (allows to re-compute events when fixing bugs or recomputing aggregations)
- Event Hubs Capture stores all incoming data in Azure Storage automatically, so no custom logic is needed. By default, it stores data in Avro format, optimized for cost efficient long-term storage.
- Consumers hosted in the App Service consume the stream events in parallel: one saves transformed lightweight CDRs, another one - computes daily summaries (aggregations).

### How we handle versioning

TandemDrive CDRs support versioning, where each update to a CDR includes a `version` field. When a new version arrives, the pipeline:
- Overwrites the previous version in the CosmosDb by using cdr id and version.
- Re-computes the daily summaries that include the updated CDR.

This design ensures correct results even if sessions are updated retroactively.

### How we handle failures
Failures in ingestion and processing are mitigated through the following strategy:
- Event replay. Event Hubs retains events for 1-7 days (Standard tier, up to 3 months Premium tier). If consumers fail or a bug is discovered, we can reprocess past data without re-ingestion.
- Retry mechanism. All consumers implement exponential backoff retries to handle transient errors.
- Consumers only commit their offset (checkpoint) after the event has been processed successfully. This guarantees that events are retried on restart or crash, and ensures consistency in case of transient failures.
- Poison messages are not retried indefinitely. After the retry limit we forward the message to a custom Dead Letter Queue on Azure Service Bus.
- Metrics and alerting. Application Insights tracks all failures, retries, and latency metrics. Alerts are configured to notify engineers in case of consistent processing failures or ingestion gaps.

