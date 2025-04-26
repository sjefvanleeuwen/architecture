You are an architect. You will use the 4plus1_architecture_template.md to create a complete architectural view  named cdr-aggregation.md



on a system described below:

A software solution running on azure:

## Component 1: Webhook integration

* Gets data from a webhook from an external trhird party service

Introduction
Webhooks allow API users to receive timely updates without constant polling. You can register a URL to receive a POST request whenever there are changes in the data exposed by the API.

Message structure
Each message has a generic envelope that wraps the data of the event. The data uses the same schema as the API's endpoints. Here's an example of a customer update message:

```json
{
  "id": "5615894c-56ba-73ac-8fc8-4120262d7fd9",
  "occurred_at": "2022-11-01T01:02:45.123Z",
  "modification_kind": "update",
  "kind": "customer",
  "data": {
    "id": "ca904ea7-4281-714e-a3cd-72df1d61907b",
    "admin_id": "tdr-france",
    "name": "TandemDrive BV",
    "country": "NL",
    "entity_type": "organisation",
    "company_reg_num": "C123456",
    "external_reference": "ABC-123",
    "created_at": "2022-11-01T01:01:45.123Z",
    "updated_at": "2022-11-01T01:02:45.123Z",
    "billing_email": "info@tandemdrive.com",
    "billing_phone": "+31123456789",
...
  }
}
```

## Component 2: Digest and aggreagte the data through an azure function


### Azure Function Flow Overview: CDR Journal Storage + Aggregation (with Idempotency)

---

### 1. **Trigger Source**
- **Trigger Type**: HTTP Request, Azure Queue, or Event Grid.
- **Purpose**: Receives incoming **Charge Detail Records (CDRs)** from external systems.

---

### 2. **Input Validation**
- Validate incoming **CDR payload**:
  - Required fields:
    - `cdrId`, `timestamp`, `evseId`, `sessionId`
    - `energy` (kWh), `duration` (seconds), `cost` (EUR)
  - Data integrity and business rules (e.g., energy > 0).

---

### 3. **Idempotency Check**
- **Purpose**: Prevent duplicate processing when the same CDR is received again.
- **Action**:
  - Check if **cdrId** already exists in:
    - **Blob Storage** (`/cdr-journal/yyyy/MM/dd/HH/cdrId.json`)
    - Or in a **processed log** (e.g., Table Storage or Cosmos DB with cdrId keys).
  - **If exists**:
    - Log as duplicate.
    - Skip aggregation, return success (idempotent handling).
  - **If not exists**:
    - Continue processing.

---

### 4. **Blob Storage - CDR Journal Entry**
- **Path Structure**: `/cdr-journal/{yyyy}/{MM}/{dd}/{HH}/{cdrId}.json`
- **Action**:
  - Serialize the **CDR** as JSON.
  - Save to **Blob Storage** for replay, auditing, compliance.

---

### 5. **Aggregation Logic**
- **Read Current Aggregate**:
  - Retrieve current **daily/monthly aggregates** from storage.
  - Example: Total energy, total revenue, session counts.

- **Update Aggregate**:
  - `total_energy += energy`
  - `total_cost += cost`
  - `total_sessions += 1`

- **Store Updated Aggregate**:
  - Save updated aggregate to **Table Storage**, **Blob**, or **Cosmos DB**.
  - Optionally, store a **record of processed cdrId** for future idempotency checks.

---

### 6. **Logging & Monitoring**
- Log:
  - CDR receipt.
  - Idempotency status (new or duplicate).
  - Journal storage success.
  - Aggregate update results.
- Use **Application Insights** for telemetry.

---

### 7. **Error Handling & Retry Scenario**
- If an error occurs **after journaling but before aggregation**:
  - The external service may **resend the same CDR**.
  - On retry, the function checks for **cdrId** and **skips** processing if already handled.
  - Ensures **no double counting** in aggregates.

---

### Example Flow:
```plaintext
Incoming CDR → Azure Function →
    1. Idempotency Check: cdrId exists?
        - Yes → Skip processing.
        - No → 
            a. Store CDR to Blob (/cdr-journal/yyyy/MM/dd/HH/cdrId.json)
            b. Fetch Aggregate → Update kWh, Cost, Sessions → Store Updated Aggregate
            c. Mark cdrId as processed.
```

---

### Optional Enhancements:
- Use **ETags** or **Optimistic Concurrency** for aggregate updates.
- Store **processed cdrIds** in a dedicated **Table Storage** or **Cosmos DB container**.
- Implement a **Replay Function** to recompute aggregates from the journal.
- Add **alerts** on repeated failed CDRs for manual intervention.

---


## Component 3: Replay Storage

 Store that data in blob storage for Yournaling / replay storage. Put this in the architectural choice. Replay Storage: Best Option & Cost Comparison

### 🎯 **Best Option: Azure Blob Storage**

| Storage Type     | Strengths for Replay Storage                                    | Weaknesses                                       |
|------------------|----------------------------------------------------------------|--------------------------------------------------|
| **Blob Storage** | - Ideal for append-only logs / journals                        | - No querying inside blobs                       |
|                  | - Cheap for large or growing data sets                         | - Requires app logic for structured access       |
|                  | - Supports folders, versioning, lifecycle policies             |                                                  |
|                  | - Easy to batch read/write for replay                          |                                                  |

---

### 💾 **Cost Comparison for 7.2 Million Writes/year (10 KB per write)**

| Storage Type     | Annual Storage (GB)* | Annual Storage Cost | Annual Write Cost | Total Annual Cost |
|------------------|----------------------|---------------------|-------------------|-------------------|
| **Blob Storage** | ~69 GB               | $1.27               | $3.60             | **~$4.87**        |
| Table Storage    | ~69 GB               | $3.45               | $0.26             | ~$3.71            |
| Cosmos DB        | ~69 GB               | $17.25              | ~$104**           | ~$121             |

\* Based on 10 KB per write.  
\** Cosmos DB: Requires ~150 RU/s, costing ~$104/year.

---

### 🔍 **Why Blob Storage is Selected**:

1. **Append-Only Journal Friendly**:
   - Replay journals typically grow over time.
   - Blob Storage handles large sequential data efficiently.

2. **Cheap for Bulk Storage**:
   - Very low storage cost per GB.
   - Write cost is predictable and low at scale.

3. **Organized Replay**:
   - Use folder structures for time-based access:
     - Example: `/journal/2025/04/26/entry001.json`

4. **Scalable**:
   - Grows with your data without complexity.
   - Integrates with **Azure Functions** for automated replay triggers.

5. **Simple Replay Logic**:
   - Just read blobs/files in sequence or based on metadata in filenames.

---

### ⚡ **When to Consider Table Storage Instead**:
- If you need **quick lookup** of individual journal entries by ID.
- If entries are **small (< 64 KB)** and structured.

---

### ❌ **Why Not Cosmos DB**:
- Too expensive for simple journaling.
- Better suited for **real-time querying** or **multi-region replication**, which journals usually don’t need.


### Front Door vs APIM Cost Comparison (7.2 Million 10KB Ingestions per Year)

| Service                   | Estimated Annual Cost         | Key Features                            | Suitable For                    |
|---------------------------|-------------------------------|-----------------------------------------|---------------------------------|
| **Azure Front Door**      | ~$35 base/month + ~$0.0075/GB data + $0.25/million requests | Global load balancing, SSL, basic WAF | Cost-effective external access  |
| **Azure API Management (Developer)** | ~$150/month                   | Basic API gateway, limited scale         | Development & Testing only      |
| **Azure API Management (Standard)**  | ~$700+/month                  | API gateway, policies, advanced security | Enterprise-grade API exposure   |
| **Azure API Management (Premium)**   | ~$2750+/month                 | Multi-region, VNET, advanced capabilities | High-scale, mission-critical APIs |

---

### Cost Calculation (Approximate Annual):
- **Front Door**:
  - 7.2M requests/year = $1.80/year
  - 72 GB data/year (7.2M x 10KB) = ~$0.54/year
  - Base cost: ~$420/year
  - Total: ~$422.34/year

- **APIM (Standard)**:
  - Fixed cost: ~$8,400/year minimum

- **APIM (Premium)**:
  - Fixed cost: ~$33,000/year minimum

---

### Recommendation:
- Use **Azure Front Door + Azure Function** if:
  - You need **secure**, **scalable**, **low-cost** webhook ingestion.
  - You can manage without **advanced API policies**.

- Use **APIM Standard or Premium** if:
  - You need **complex API management**, **throttling**, **transformations**, or **multi-region deployments**.
  - Your project requires **enterprise-grade governance** or **VNET integration**.

For this use case (CDR ingestion), **Front Door** is **strongly recommended** to minimize costs while ensuring global access and reliability.



## Create the PUML's for C4 diagrams

c4_component.puml
c4_container.puml
c4_context.puml
c4_deployment.puml

in path ./diagrams/cdr-aggregation/
