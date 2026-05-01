# Building-a-Secure-Serverless-Order-Processing-System-in-Azure

> **Skills demonstrated:** Azure Functions, API Management, Azure SQL Database, Key Vault, Managed Identity, Virtual Networks, Private Endpoints, Application Insights, Azure Monitor, Azure Communication Services, C# .NET 8, event-driven architecture, cloud security

---

## Overview

This project is a fully serverless, event-driven order processing system built on Microsoft Azure as part of my final semester Cloud Computing Technologies course. The system receives customer orders through a secure API gateway, validates them, queues them for async processing, writes them to a relational database, and sends personalised confirmation emails — all without a single virtual machine, all without a single hardcoded password, and all running at effectively zero cost when idle.

The architecture covers seven Azure services working together, demonstrates production-grade security through Managed Identity and Private Endpoints, and implements real reliability patterns like idempotency, dead-letter queues, and distributed telemetry.

**Tech stack:** Azure Functions (Flex Consumption), API Management, Azure Storage Queue, Azure SQL Database, Azure Key Vault, Azure Communication Services, Application Insights, Log Analytics, Azure Monitor, Virtual Network with Private Endpoint, C# .NET 8 isolated worker model.

---

## Architecture diagram
<img width="2496" height="2393" alt="Azure Serverless Order Processing System Diagram" src="https://github.com/user-attachments/assets/82809306-9350-4444-ad5b-10953d97b64d" />

The system processes an order through 16 numbered steps:

| Step | Action |
|------|--------|
| 1 | Postman sends HTTPS POST to API Management with subscription key |
| 2 | APIM validates key and forwards to ValidateOrder via Function host key |
| 3 | ValidateOrder returns 202 Accepted immediately |
| 4 | ValidateOrder enqueues plain JSON message to orders-queue |
| 5 | Queue trigger fires ProcessOrder — SDK auto-deletes on success |
| 6 | After 3 failed dequeues, SDK moves message to orders-queue-poison |
| 7 | ProcessOrder retrieves secrets from Key Vault via Managed Identity |
| 8 | Key Vault returns SQL and ACS connection strings |
| 9 | ProcessOrder routes SQL traffic through VNet to Private Endpoint |
| 10 | Private Endpoint forwards SQL INSERT to OrdersDB |
| 11 | ProcessOrder sends personalised email via Azure Communication Services |
| 12 | Customer receives HTML confirmation email |
| 13 | ValidateOrder emits invocation telemetry to Application Insights |
| 14 | ProcessOrder emits SQL latency and email status telemetry |
| 15 | App Insights forwards metrics to Azure Monitor |
| 16 | Azure Monitor fires Sev2 alert when poison queue depth exceeds zero |

---

## Why I made each architecture decision

Before getting into the build steps I want to explain the reasoning behind every service choice, because these decisions matter more than the services themselves.

### Azure API Management over a raw Function URL

The first instinct might be to expose the Function App's HTTP trigger URL directly. I chose not to because raw function URLs have no rate limiting, no subscription-based authentication, and change when you redeploy the function. APIM gives a stable gateway URL that never changes, validates the `Ocp-Apim-Subscription-Key` header before the request reaches any code, and lets you add policies like request transformation or IP filtering without touching function code. The Consumption tier costs essentially nothing at student scale.

### Flex Consumption over standard Consumption plan

This was the most research-intensive decision of the project. I initially created the Function App on the standard Consumption plan and then discovered that VNet Integration — which routes outbound traffic through a Virtual Network to reach the SQL Private Endpoint — is not supported on standard Consumption. Only Flex Consumption and Premium plans support it. Flex Consumption scales to zero when idle like standard Consumption, so the cost profile is similar, but it unlocks the network security capability I needed.

### Async queue pattern — 202 Accepted instead of synchronous processing

ValidateOrder does not write to the database. It validates the payload, enqueues the message, and returns 202 immediately. This is intentional. If I processed the order synchronously — validate, write to SQL, send email, then return — the client would wait 2 to 5 seconds for every request. By returning 202 immediately and processing asynchronously, the API response time is always under 200 milliseconds regardless of how long the database write or email send takes.

### Azure Storage Queue over Azure Event Grid

The project spec listed Event Grid as a sample service. I chose Storage Queue instead because Event Grid is designed for fan-out — one event triggering multiple independent subscribers simultaneously. My system has exactly one consumer of the order message (ProcessOrder), so Event Grid would add an unnecessary layer. Storage Queue is simpler, costs fractions of a cent, and has a built-in mechanism for moving failed messages to a poison queue after a configurable number of retries. No extra code needed.

### Azure SQL Database over Azure Table Storage

The spec also listed Table Storage. I chose SQL because order data is inherently relational. An order has a customer, line items, a status, a timestamp, and amounts — data that benefits from schema enforcement, ACID transactions, and proper querying. Table Storage is a flat key-value NoSQL store. You can store order data there but querying it beyond simple key lookups requires scanning entire partitions. SQL was the architecturally correct choice. I used the Basic 5 DTU tier at approximately $6 CAD per month to stay within student credit limits.

### Azure Communication Services over SendGrid

The spec mentioned sending confirmation emails but did not specify a service. SendGrid is a popular choice but it is a third-party SaaS service that lives outside Azure. Using it means your architecture has an external dependency, an external API key to manage, and a service that does not integrate with Managed Identity. ACS is a first-party Azure service that stays inside the subscription boundary, uses an Azure-managed email domain, and has a free tier covering 100 emails per day — more than sufficient for any demo or low-volume production workload.

### Token-based SQL authentication over connection string authentication

My SQL Server is configured for Entra ID-only authentication — SQL username and password authentication is completely disabled. In code, I use `DefaultAzureCredential` to fetch a bearer token for the `https://database.windows.net/.default` scope and assign it to `SqlConnection.AccessToken`. This means ProcessOrder authenticates to SQL using the Function App's Managed Identity with zero credentials in code or config. This approach requires the Managed Identity to be added as a database user inside SQL, which is a one-time setup step.

---

## Build guide — step by step

Everything below was built manually through the Azure portal. No Bicep, no Terraform, no ARM templates.

---

### Phase 1 — Foundation: Resource Group, VNet, Subnet, NSG

**Why this phase is first:** Everything else lives inside the resource group. The VNet must exist before the Function App because VNet Integration at Function App creation time requires an existing VNet.

**1.1 — Create the Resource Group**

Portal → Resource groups → Create:
- Name: `rg-order-processing-demo`
- Region: `Canada Central`

**1.2 — Create the Virtual Network**

Portal → Virtual networks → Create:
- Name: `vnet-order-processing`
- Address space: `10.0.0.0/16`
- Delete the default subnet

Add subnet:
- Name: `snet-private-endpoints`
- Range: `10.0.1.0/24`

**1.3 — Create the Network Security Group**

Portal → Network security groups → Create:
- Name: `nsg-private-endpoints`
- Region: `Canada Central`

Add inbound rules:

| Priority | Name | Source | Destination | Action |
|----------|------|--------|-------------|--------|
| 100 | Allow-VNet-Inbound | Service Tag: VirtualNetwork | Service Tag: VirtualNetwork | Allow |
| 200 | Deny-Internet-Inbound | Service Tag: Internet | Any | Deny |

Attach NSG to subnet: VNet → Subnets → `snet-private-endpoints` → Network security group → select `nsg-private-endpoints` → Save.

**Key learning:** NSGs attach to subnets or NICs — never to the VNet itself. This is a common misconception.

---

### Phase 2 — Storage Account and Queues

**Why before Function App:** The Function App requires a storage account at creation time for its own host storage. It cannot be added afterward.

**2.1 — Create Storage Account**

Portal → Storage accounts → Create:
- Name: `storderprocessing189` (globally unique, lowercase)
- Region: `Canada Central`
- Performance: Standard
- Redundancy: **Geo-redundant storage (GRS)**

Advanced tab:
- Require secure transfer: Enabled
- Minimum TLS version: 1.2
- Allow Blob anonymous access: Disabled

**2.2 — Create queues**

Storage account → Queues → Add queue:
- `orders-queue`
- `orders-deadletter`

**Key learning:** The Azure Storage Queue SDK automatically creates a poison queue named `{queuename}-poison` (so `orders-queue-poison`) when messages fail. The manually created `orders-deadletter` queue was never used by the SDK. Always check the actual SDK behaviour rather than assuming.

---

### Phase 3 — SQL Server and Database

**3.1 — Create SQL Server**

Portal → SQL servers → Create:
- Name: `sql-order-processing-189`
- Region: `Canada Central`
- Authentication: **Use Microsoft Entra-only authentication**
- Entra admin: your student account

Networking tab:
- Connectivity: No access (public endpoint disabled from the start)

**3.2 — Create Database**

SQL Server → Databases → Create:
- Name: `OrdersDB`
- Compute + storage: **Basic, 5 DTUs**
- Backup redundancy: Locally-redundant

**3.3 — Create Orders table**

Temporarily allow your client IP in the SQL Server firewall → Query editor → run:

```sql
CREATE TABLE Orders (
    OrderId         NVARCHAR(50)    NOT NULL PRIMARY KEY,
    CustomerId      NVARCHAR(50)    NOT NULL,
    CustomerName    NVARCHAR(100)   NULL,
    CustomerEmail   NVARCHAR(255)   NULL,
    CustomerAddress NVARCHAR(500)   NULL,
    Items           NVARCHAR(MAX)   NOT NULL,
    TotalAmount     DECIMAL(10,2)   NOT NULL,
    Status          NVARCHAR(20)    NOT NULL DEFAULT 'Received',
    CreatedAt       DATETIME2       NOT NULL DEFAULT GETUTCDATE(),
    ProcessedAt     DATETIME2       NULL
);
```

Disable public access again immediately after.

**Key learning:** Basic tier does not support Active Geo-Replication. Geo-replication requires Standard S3 or higher. Designing for the right tier matters — check capability limitations before selecting.

---

### Phase 4 — Key Vault

**4.1 — Create Key Vault**

Portal → Key vaults → Create:
- Name: `kv-order-processing-189`
- Region: `Canada Central`
- Pricing tier: Standard
- Soft-delete: Enabled (default)
- Purge protection: **Enabled**

Access configuration:
- Permission model: **Azure role-based access control**

**4.2 — Assign yourself Key Vault Administrator**

Key Vault → Access control (IAM) → Add role assignment:
- Role: Key Vault Administrator
- Assign to: your student account

Wait 2 to 3 minutes for role propagation before creating secrets.

**4.3 — Add secrets**

Key Vault → Secrets → Generate/Import:
- `sql-connection-string`: `Server=tcp:sql-order-processing-189.database.windows.net,1433;Initial Catalog=OrdersDB;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;`
- `acs-connection-string`: placeholder (updated in Phase 7)

**Key learning:** Under the RBAC permission model, the vault creator gets no automatic access. You must explicitly assign yourself a role. This is intentional — it enforces least privilege even for the resource owner.

---

### Phase 5 — Function App

This is the most complex phase. Do not skip sub-steps.

**5.1 — Create Function App**

Portal → Function App → Create → **Flex Consumption**:
- Name: `func-order-processing-189`
- Region: `Canada Central`
- Runtime: .NET 8 (LTS), isolated worker model
- OS: Linux
- Instance size: 2048 MB
- Storage account: `storderprocessing189`

Networking tab:
- Enable public access: On (APIM needs to reach it)
- Enable virtual network integration: On
- Virtual network: `vnet-order-processing`
- Outbound subnet: Create new → `snet-functions-outbound` → `10.0.2.0/24`

Monitoring tab: Disabled (we create App Insights manually in Phase 7 for workspace-based mode)

**5.2 — Enable Managed Identity**

Function App → Settings → Identity → System assigned → Status: **On** → Save.
Copy the Object ID.

**5.3 — Grant MI access to Key Vault**

Key Vault → Access control (IAM) → Add role assignment:
- Role: **Key Vault Secrets User**
- Assign to: Managed identity → `func-order-processing-189`

**5.4 — Grant MI access to Storage Account**

Storage Account → Access control (IAM) → Add role assignment:
- Role: **Storage Queue Data Contributor**
- Assign to: `func-order-processing-189`

**5.5 — Add environment variables**

Function App → Settings → Environment variables → App settings → Add:

| Name | Value |
|------|-------|
| `KEY_VAULT_URI` | `https://kv-order-processing-189.vault.azure.net/` |
| `SQL_SECRET_NAME` | `sql-connection-string` |
| `ACS_SECRET_NAME` | `acs-connection-string` |
| `ORDERS_QUEUE_NAME` | `orders-queue` |
| `STORAGE_ACCOUNT_NAME` | `storderprocessing189` |
| `ACS_SENDER_DOMAIN` | your ACS domain (added after Phase 7) |
| `WEBSITE_DNS_SERVER` | `168.63.129.16` |
| `WEBSITE_VNET_ROUTE_ALL` | `1` |

**5.6 — Create Private Endpoint for SQL**

OrdersDB → Networking → Private access → Create private endpoint:
- Name: `pe-sql-orderdb`
- Region: `Canada Central`
- Target sub-resource: sqlServer
- Virtual network: `vnet-order-processing`
- Subnet: `snet-private-endpoints`
- Integrate with private DNS zone: **Yes** → creates `privatelink.database.windows.net`

**5.7 — Grant Managed Identity SQL database access**

SQL Server → OrdersDB → Query editor → run:

```sql
CREATE USER [func-order-processing-189] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [func-order-processing-189];
ALTER ROLE db_datawriter ADD MEMBER [func-order-processing-189];
```

**5.8 — Deploy function code**

Create the project locally in VS Code using Azure Functions extension:
- Language: C#
- Runtime: .NET 8 Isolated
- First function: HTTP trigger → ValidateOrder
- Second function: Queue trigger → ProcessOrder

Update `host.json`:

```json
{
  "version": "2.0",
  "extensions": {
    "queues": {
      "maxPollingInterval": "00:00:02",
      "visibilityTimeout": "00:00:30",
      "batchSize": 16,
      "maxDequeueCount": 3,
      "newBatchThreshold": 8,
      "messageEncoding": "none"
    }
  },
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      },
      "enableLiveMetricsFilters": true
    }
  },
  "concurrency": {
    "dynamicConcurrencyEnabled": true,
    "snapshotPersistenceEnabled": true
  }
}
```

Install NuGet packages:

```bash
dotnet add package Azure.Identity
dotnet add package Azure.Security.KeyVault.Secrets
dotnet add package Azure.Storage.Queues
dotnet add package Microsoft.Data.SqlClient
dotnet add package Azure.Communication.Email
dotnet add package Microsoft.Azure.Functions.Worker.Extensions.Storage.Queues
```

`ValidateOrder.cs`:

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;
using Azure.Storage.Queues;
using Azure.Identity;
using System.Text.Json;

public class ValidateOrder
{
    private readonly ILogger<ValidateOrder> _logger;

    public ValidateOrder(ILogger<ValidateOrder> logger)
    {
        _logger = logger;
    }

    [Function("ValidateOrder")]
    public async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = "orders")] HttpRequest req)
    {
        _logger.LogInformation("ValidateOrder triggered");

        string body = await new StreamReader(req.Body).ReadToEndAsync();

        if (string.IsNullOrWhiteSpace(body))
            return new BadRequestObjectResult(new { error = "Request body is empty" });

        JsonDocument doc;
        try { doc = JsonDocument.Parse(body); }
        catch { return new BadRequestObjectResult(new { error = "Invalid JSON" }); }

        var root = doc.RootElement;

        string[] requiredFields = {
            "orderId", "customerId", "customerName",
            "customerEmail", "customerAddress", "items", "totalAmount"
        };

        foreach (var field in requiredFields)
        {
            if (!root.TryGetProperty(field, out _))
                return new BadRequestObjectResult(new { error = $"Missing required field: {field}" });
        }

        string customerEmail = root.GetProperty("customerEmail").GetString()!;
        if (!customerEmail.Contains("@") || !customerEmail.Contains("."))
            return new BadRequestObjectResult(new { error = "Invalid customerEmail format" });

        if (root.GetProperty("totalAmount").GetDecimal() <= 0)
            return new BadRequestObjectResult(new { error = "totalAmount must be greater than zero" });

        string orderId = root.GetProperty("orderId").GetString()!;

        string storageAccountName = Environment.GetEnvironmentVariable("STORAGE_ACCOUNT_NAME")!;
        string queueName = Environment.GetEnvironmentVariable("ORDERS_QUEUE_NAME")!;
        var queueUri = new Uri($"https://{storageAccountName}.queue.core.windows.net/{queueName}");
        var queueClient = new QueueClient(queueUri, new DefaultAzureCredential());

        await queueClient.SendMessageAsync(body);

        _logger.LogInformation("Order {OrderId} enqueued", orderId);

        return new AcceptedResult(string.Empty, new
        {
            orderId,
            message = "Order received and queued for processing",
            customerEmail
        });
    }
}
```

`ProcessOrder.cs`:

```csharp
using Azure.Core;
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Data.SqlClient;
using Microsoft.Extensions.Logging;
using System.Text.Json;

public class ProcessOrder
{
    private readonly ILogger<ProcessOrder> _logger;

    public ProcessOrder(ILogger<ProcessOrder> logger)
    {
        _logger = logger;
    }

    [Function("ProcessOrder")]
    public async Task Run(
        [QueueTrigger("orders-queue", Connection = "AzureWebJobsStorage")]
        string message,
        FunctionContext context)
    {
        _logger.LogInformation("ProcessOrder triggered");

        string json;
        try
        {
            string decoded = System.Text.Encoding.UTF8.GetString(Convert.FromBase64String(message));
            JsonDocument.Parse(decoded);
            json = decoded;
        }
        catch { json = message; }

        var root        = JsonDocument.Parse(json).RootElement;
        string orderId  = root.GetProperty("orderId").GetString()!;
        string custId   = root.GetProperty("customerId").GetString()!;
        string custName = root.GetProperty("customerName").GetString()!;
        string custEmail = root.GetProperty("customerEmail").GetString()!;
        string custAddr = root.GetProperty("customerAddress").GetString()!;
        string items    = root.GetProperty("items").ToString();
        decimal amount  = root.GetProperty("totalAmount").GetDecimal();

        _logger.LogInformation("Processing order {OrderId} for {Name}", orderId, custName);

        var credential  = new DefaultAzureCredential();
        string kvUri    = Environment.GetEnvironmentVariable("KEY_VAULT_URI")!;
        string sqlName  = Environment.GetEnvironmentVariable("SQL_SECRET_NAME")!;
        var kvClient    = new SecretClient(new Uri(kvUri), credential);
        var secret      = await kvClient.GetSecretAsync(sqlName);
        string connStr  = secret.Value.Value;

        var tokenContext = new TokenRequestContext(
            new[] { "https://database.windows.net/.default" });
        AccessToken token = await credential.GetTokenAsync(tokenContext);

        using var conn = new SqlConnection(connStr);
        conn.AccessToken = token.Token;
        await conn.OpenAsync();

        using var checkCmd = new SqlCommand(
            "SELECT COUNT(1) FROM Orders WHERE OrderId = @Id", conn);
        checkCmd.Parameters.AddWithValue("@Id", orderId);
        int count = (int)await checkCmd.ExecuteScalarAsync();

        if (count > 0)
        {
            _logger.LogWarning("Order {OrderId} already exists — skipping (idempotency)", orderId);
            return;
        }

        using var insertCmd = new SqlCommand(
            @"INSERT INTO Orders 
              (OrderId, CustomerId, CustomerName, CustomerEmail, CustomerAddress, Items, TotalAmount, Status, ProcessedAt)
              VALUES (@OrderId, @CustomerId, @CustomerName, @CustomerEmail, @CustomerAddress, @Items, @Amount, 'Received', @ProcessedAt)",
            conn);
        insertCmd.Parameters.AddWithValue("@OrderId",         orderId);
        insertCmd.Parameters.AddWithValue("@CustomerId",      custId);
        insertCmd.Parameters.AddWithValue("@CustomerName",    custName);
        insertCmd.Parameters.AddWithValue("@CustomerEmail",   custEmail);
        insertCmd.Parameters.AddWithValue("@CustomerAddress", custAddr);
        insertCmd.Parameters.AddWithValue("@Items",           items);
        insertCmd.Parameters.AddWithValue("@Amount",          amount);
        insertCmd.Parameters.AddWithValue("@ProcessedAt",     DateTime.UtcNow);
        await insertCmd.ExecuteNonQueryAsync();

        _logger.LogInformation("Order {OrderId} inserted into OrdersDB", orderId);

        try
        {
            string acsName      = Environment.GetEnvironmentVariable("ACS_SECRET_NAME")!;
            string acsDomain    = Environment.GetEnvironmentVariable("ACS_SENDER_DOMAIN")!;
            var acsSecret       = await kvClient.GetSecretAsync(acsName);
            var emailClient     = new Azure.Communication.Email.EmailClient(acsSecret.Value.Value);

            var content = new Azure.Communication.Email.EmailContent($"Order Confirmation - {orderId}")
            {
                PlainText = $"Dear {custName}, your order {orderId} for ${amount:F2} has been received.",
                Html = $@"<html><body>
                    <h2>Order Confirmation</h2>
                    <p>Dear <strong>{custName}</strong>,</p>
                    <table>
                        <tr><td>Order ID</td><td>{orderId}</td></tr>
                        <tr><td>Total</td><td>${amount:F2}</td></tr>
                        <tr><td>Address</td><td>{custAddr}</td></tr>
                        <tr><td>Status</td><td>Received</td></tr>
                        <tr><td>Processed</td><td>{DateTime.UtcNow:yyyy-MM-dd HH:mm} UTC</td></tr>
                    </table></body></html>"
            };

            var msg = new Azure.Communication.Email.EmailMessage(
                $"DoNotReply@{acsDomain}", custEmail, content);

            await emailClient.SendAsync(Azure.WaitUntil.Completed, msg);
            _logger.LogInformation("Email sent to {Email} for order {OrderId}", custEmail, orderId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Email failed for {OrderId} — order saved successfully", orderId);
        }
    }
}
```

Deploy:

```bash
func azure functionapp publish func-order-processing-189 --dotnet-isolated
```

---

### Phase 6 — API Management

**6.1 — Create APIM**

Portal → API Management → Create:
- Name: `apim-order-processing-189`
- Region: `Canada Central`
- Pricing tier: **Consumption**

Creation takes 15 to 20 minutes.

**6.2 — Store Function host key as Named Value**

Function App → Functions → ValidateOrder → Function Keys → copy default key.

APIM → Named values → Add:
- Name: `func-validate-order-key`
- Type: Secret
- Value: paste function key

**6.3 — Create API and operation**

APIM → APIs → Add API → HTTP:
- Display name: `Order Processing API`
- Web service URL: `https://func-order-processing-189-XXXX.canadacentral-01.azurewebsites.net/api/orders`
- API URL suffix: leave blank

Add operation:
- Method: POST
- URL template: `/`

**6.4 — Add inbound policy**

```xml
<policies>
  <inbound>
    <base />
    <set-header name="x-functions-key" exists-action="override">
      <value>{{func-validate-order-key}}</value>
    </set-header>
  </inbound>
  <backend><base /></backend>
  <outbound><base /></outbound>
  <on-error><base /></on-error>
</policies>
```

**6.5 — Create subscription key for Postman**

APIM → Subscriptions → Add:
- Name: `postman-test-subscription`
- Scope: API → Order Processing API

Copy the primary key — this is your `Ocp-Apim-Subscription-Key`.

---

### Phase 7 — Monitoring and ACS

**7.1 — Create Log Analytics Workspace**

Portal → Log Analytics workspaces → Create:
- Name: `law-order-processing`
- Region: `Canada Central`
- Retention: 30 days

**7.2 — Create Application Insights (workspace-based)**

Portal → Application Insights → Create:
- Name: `appi-order-processing`
- Region: `Canada Central`
- Resource mode: **Workspace-based**
- Log Analytics workspace: `law-order-processing`

Link to Function App: Function App → Application Insights → select `appi-order-processing` → Apply.

**7.3 — Create Azure Communication Services**

Portal → Communication Services → Create:
- Name: `acs-order-processing-189`
- Data location: Canada

ACS → Email → Domains → Add domain → **Azure managed domain**.
Copy the sender domain and update the `ACS_SENDER_DOMAIN` environment variable on the Function App.

Update the `acs-connection-string` secret in Key Vault with the real ACS connection string from ACS → Keys.

**7.4 — Create Azure Monitor Alert**

Monitor → Alerts → Create alert rule:
- Scope: Storage Account → Queue service (sub-resource)
- Signal: Queue Message Count
- Dimension filter: Queue name = `orders-queue-poison`
- Aggregation: Maximum
- Threshold: 0 (fire on any poison message)
- Evaluation frequency: 1 minute
- Lookback: 5 minutes

Action Group:
- Name: `ag-order-alerts`
- Region: Global
- Notification: Email → your address

---

### Phase 8 — End-to-end test

Postman request:

```
POST https://apim-order-processing-189.azure-api.net/
Headers:
  Ocp-Apim-Subscription-Key: your-key
  Content-Type: application/json

Body:
{
  "orderId": "ORD-001",
  "customerId": "CUST-100",
  "customerName": "Jane Smith",
  "customerEmail": "your@email.com",
  "customerAddress": "123 Main St, Toronto, ON M5V 1A1",
  "items": [{"name": "Azure Textbook", "qty": 1, "price": 49.99}],
  "totalAmount": 49.99
}
```

Verify:
1. Response: `202 Accepted`
2. SQL: `SELECT * FROM Orders` shows new row with `ProcessedAt` populated
3. Email: personalised HTML confirmation in inbox
4. App Insights → Transaction search: full end-to-end trace visible
5. Log Analytics → Logs: `traces | where timestamp > ago(1h)` returns function logs

---

## Challenges and what I learned

**Challenge 1 — Flex Consumption VNet Integration DNS resolution**

The private endpoint was created and linked to the Private DNS Zone but ProcessOrder kept failing to connect to SQL. The root cause was that the Function App was not using the Azure internal DNS resolver and was resolving the SQL hostname to its public IP rather than the private endpoint IP. The fix required two environment variables: `WEBSITE_DNS_SERVER=168.63.129.16` to point the Function App at Azure's internal DNS server (which reads Private DNS Zones linked to the VNet), and `WEBSITE_VNET_ROUTE_ALL=1` to force all outbound traffic through the VNet rather than just RFC 1918 addresses. After adding these, the SQL Server's public access could be completely disabled.

**Challenge 2 — SQL authentication with Managed Identity**

My first attempt used `Authentication=Active Directory Managed Identity` in the connection string, which threw a `System.ArgumentException` about a missing authentication provider package. Rather than adding the extension package, I switched to the more reliable approach of fetching an access token manually using `DefaultAzureCredential` and assigning it to `conn.AccessToken`. This approach works with any version of `Microsoft.Data.SqlClient` without additional dependencies.

**Challenge 3 — Queue trigger not firing on Flex Consumption**

After deploying the functions, `ValidateOrder` worked correctly but `ProcessOrder` never triggered even though messages appeared in `orders-queue`. The issue was a missing `messageEncoding: none` setting in `host.json`. Without this, the isolated worker model queue trigger expects Base64-encoded messages and fails silently when it receives plain JSON. Adding the `extensions.queues` block to `host.json` with `messageEncoding: none` and a `maxPollingInterval` of 2 seconds resolved it immediately.

**Challenge 4 — APIM routing 404**

The backend URL in APIM was set to the Function App hostname without the function route, causing APIM to forward to the root of the app rather than the `/api/orders` endpoint. Flex Consumption apps use a unique hostname format (`func-name-XXXX.canadacentral-01.azurewebsites.net`) rather than the standard `func-name.azurewebsites.net` format. Updating the backend URL to include the correct unique hostname and the full path resolved the 404.

---

## Skills this project demonstrates for a recruiter

**Cloud architecture design** — designed a multi-layer serverless architecture from scratch, justified every service selection against alternatives, and iterated based on real deployment constraints rather than theoretical knowledge.

**Azure security** — implemented zero-trust security through Managed Identity, Key Vault, Private Endpoints, NSGs, and token-based SQL authentication. Not a single password exists anywhere in the system.

**Networking** — configured VNet Integration, Private DNS Zones, Private Endpoints, and NSG rules. Debugged DNS resolution issues in a cloud network environment.

**Event-driven patterns** — implemented the async offload pattern, at-least-once delivery with idempotency, dead-letter queue handling, and fan-out prevention through appropriate service selection.

**C# .NET 8** — wrote two production-quality Azure Functions using the isolated worker model, including async/await, JSON parsing, SQL with token auth, and ACS email sending.

**Observability** — configured workspace-based Application Insights, Log Analytics, Azure Monitor alert rules, and Action Groups for complete production-grade telemetry.

**Debugging** — diagnosed and resolved DNS resolution failures, queue trigger binding issues, SQL authentication errors, and APIM routing problems through systematic log analysis using Application Insights KQL queries.

---

## Repository structure

```
azure-serverless-order-processing/
├── README.md
├── diagram.png
├── architecture-description.md
├── src/
│   └── order-processing-functions/
│       ├── ValidateOrder.cs
│       ├── ProcessOrder.cs
│       ├── Program.cs
│       ├── host.json
│       ├── local.settings.json.example
│       └── order-processing-functions.csproj
└── docs/
    ├── build-guide.md
    ├── environment-variables.md
    └── troubleshooting.md
```

---

## Cost breakdown

| Service | Tier | Estimated monthly cost |
|---------|------|----------------------|
| Function App | Flex Consumption — scales to zero | ~$0 at demo scale |
| API Management | Consumption — per call billing | ~$0 at demo scale |
| Storage Account | Standard GRS | ~$0.05 |
| Azure SQL Database | Basic 5 DTU | ~$6 CAD |
| Key Vault | Standard | ~$0.03 |
| Azure Communication Services | Free tier | $0 |
| Application Insights | 5GB free tier | $0 |
| Log Analytics | Pay-as-you-go | ~$0 at demo scale |
| Virtual Network | Free | $0 |
| **Total** | | **~$6-10 CAD/month** |

The only meaningful cost is the SQL Database. Everything else is effectively free at student or low-volume scale because Azure's serverless services bill per execution with generous free tiers.

---
