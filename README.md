# 🚴 Bicycle Order Management System - n8n Automation

## 📋 Project Overview

## 🔄 Universal Application

**Note:** This system uses bicycle orders as a demonstration, but the logic is **fully adaptable** to any business model:

- 🛍️ **E-commerce:** Any product catalog (clothing, electronics, furniture)
- 📦 **B2B:** Lead qualification and supplier management
- 🏠 **Real Estate:** Property inquiry handling and follow-ups
- 🎓 **Education:** Course enrollment and student onboarding
- 🍕 **Food Delivery:** Order intake and kitchen management
- 💼 **Services:** Appointment booking and client management

Simply replace:

- “Bicycle Models” → Your product/service name
- “Stock” → Your resource availability
- “Warehouse” → Your fulfillment system

The **AI validation**, **loop prevention**, and **delayed follow-up** logic remain universal.

An end-to-end automated order management system for bicycle e-commerce, built entirely with **n8n workflows**. The system handles order intake, AI-powered data validation, inventory management, customer follow-ups, and automated restocking alerts.

**Key Features:**

- 🤖 AI-powered customer data validation (OpenAI GPT-4)
- 📦 Real-time inventory tracking and stock management
- 🔄 Automated customer response processing with loop prevention
- ⏰ Time-based order cleanup (>48h auto-discard)
- 📧 Smart email notifications to customers and warehouse team
- 🔓 Automatic order unlock when stock becomes available

-----

## 🏗️ System Architecture

The system consists of **4 interconnected workflows**:

```
WF1: Order Intake & Validation
↓
WF2: Stale Order Cleanup (runs every 6h)
↓
WF3: Customer Response Handler
↓
WF4: Delayed Orders Follow-up (runs every 6h)
```

### Visual Architecture

```mermaid
graph TD
    A[Webhook POST] --> B[WF1: AI Validation]
    B -->|VALID| C[Stock Check]
    B -->|PENDING| D[Manual Verification DB]
    B -->|INVALID| E[Reject Order]
    C -->|In Stock| F[Confirm Order ✅]
    C -->|Out of Stock| G[Delayed DB]
    H[Schedule 6h] --> I[WF2: Cleanup 48h+]
    I --> J[Archive DB]
    K[Gmail Trigger] --> L[WF3: Response Handler]
    L -->|retryCount < 2| B
    L -->|retryCount >= 2| M[Team Alert 🚨]
    N[Schedule 6h] --> O[WF4: Delayed Follow-up]
    O -->|Stock arrived| F
    O -->|Still no stock| P[Restock Alert 📦]
```

-----

## 🏛️ Architecture Decisions

> This section explains **why** specific choices were made, not just **what** the system does.

### Why 4 separate workflows instead of 1 monolithic flow?

Each workflow has a single responsibility and can fail independently without affecting the others. WF2 and WF4 run on a schedule — merging them with WF1 would create unnecessary coupling, harder debugging, and unpredictable execution timing. Separation also allows activating/deactivating each workflow independently during maintenance.

### Why Google Sheets as the database layer?

Chosen for portfolio visibility and zero-infrastructure setup — anyone can clone, configure credentials, and run the system without provisioning a server. In a production scenario, this layer would be replaced with **Postgres or Supabase** to support atomic transactions and proper concurrency handling.

### Why cap retryCount at 2?

Prevents infinite loops while giving customers two correction attempts. After 2 failed re-validations, human review is statistically more efficient than further automation. The threshold is intentionally low to avoid customer frustration from repeated emails.

### Why use OpenAI for validation instead of pure regex?

Regex handles structured patterns but fails on ambiguous real-world data — e.g., a partial address that looks plausible but is incomplete. GPT-4 can reason about context, classify edge cases into PENDING rather than VALID/INVALID, and extract structured data from unstructured email replies in WF3. The tradeoff is latency and API cost, which is acceptable at this order volume.

-----

## ⚠️ Known Limitations & Design Decisions

### Inventory Race Condition

**Current behavior:** WF1 reads stock, compares, then updates in three sequential steps. If two orders arrive simultaneously via webhook, both reads could return the same stock value before either write completes, potentially confirming orders that exceed available inventory.

**Why it’s acceptable here:** Google Sheets does not support row-level locking or atomic transactions. This is a known constraint of using Sheets as a database layer.

**Production solution:** Replace Google Sheets with a database that supports atomic operations (Postgres `SELECT FOR UPDATE`, Supabase RPC functions). The workflow logic remains identical — only the database layer changes.

### Schedule-Based Polling (WF2, WF4)

Running every 6 hours means delayed order processing has up to a 6-hour lag. This is a deliberate tradeoff between resource usage and responsiveness. In a higher-volume system, a webhook-based stock update trigger would replace the schedule.

### Gmail Polling Latency (WF3)

Gmail trigger polls every minute. For time-sensitive responses, a push notification approach (Gmail Pub/Sub) would reduce latency to near-real-time.

-----

## ⚙️ Setup & Configuration

### Prerequisites

- n8n instance (self-hosted or n8n Cloud)
- OpenAI API key (GPT-4 access)
- Google account with Google Sheets and Gmail access
- Gmail account configured as both sender and trigger

### Configuration Steps

1. **Import workflows** — import all 4 JSON files into n8n (`WF1`, `WF2`, `WF3`, `WF4`)
1. **Configure credentials** in n8n:
- `OpenAI API` — API key with GPT-4 access
- `Google Sheets OAuth2` — service account or OAuth
- `Gmail OAuth2` — for both trigger and send nodes
1. **Create Google Sheets** with the schema described in the “Database Structure” section below (5 sheets total)
1. **Update Sheet IDs** — paste your Google Sheet IDs into each Google Sheets node
1. **Link WF1 webhook URL** — copy WF1’s webhook URL and paste it into WF3’s HTTP Request node
1. **Activate** WF2 and WF4 schedule triggers
1. **Test** using the sample payload below

### Sample Test Payload (WF1 Webhook)

Send a `POST` request to your WF1 webhook URL with this body:

```json
{
  "customerName": "Mario Rossi",
  "email": "mario.rossi@example.com",
  "phone": "+39 02 1234567",
  "shippingAddress": "Via Roma 12, 20121 Milano, Italy",
  "bicycleModel": "Mountain Pro 29",
  "quantity": 2,
  "notes": "Please deliver in the afternoon"
}
```

**Expected results by scenario:**

|Scenario        |AI Result             |System Action                     |
|----------------|----------------------|----------------------------------|
|All fields valid|`VALID`               |Stock check → CONFIRMED or DELAYED|
|Missing ZIP/city|`PENDING_VERIFICATION`|Saved to Manual DB + email sent   |
|Fake/test data  |`INVALID`             |Order rejected immediately        |

-----

## 📊 Database Structure

### Google Sheets Databases:

1. **Inventory - Warehouse**
- Bicycle Models | stock | price | orderStatus | lastUpdated
1. **Orders - Monthly**
- id | CustomerName | Email | phone | Shipping Address | Bicycle Models | quantity | price | notes | orderStatus | Timestamp
1. **Manual Verification DB**
- id | CustomerName | email | phone | Shipping Address | Bicycle Models | quantity | price | orderStatus | notes | Timestamp | validatedAt | retryCount
1. **Orders DELAYED**
- id | CustomerName | Email | phone | Shipping Address | Bicycle Models | quantity | price | notes | orderStatus | Timestamp
1. **Archive / NO_SHOW_UP**
- id | customerName | Phone | Email | Shipping Address | DISCARDED | Timestamp | discardedAt

-----

## 🔄 Workflow Details

### **WF1 - ORDER INTAKE & VALIDATION**

**Purpose:** Receives new orders, validates customer data via AI, checks inventory, and routes orders accordingly.

**Flow:**

```
📥 Webhook (POST)
↓
🤖 AI Validation (OpenAI GPT-4)
├─ VALID → Continue to stock check
├─ PENDING_VERIFICATION → Manual DB + Email customer
└─ INVALID → Reject order
↓
📊 Read Inventory
↓
💻 Code: Compare stock vs quantity
↓
🔀 IF: Stock available?
├─ ✅ YES:
│   ├─ Update Inventory (decrease stock)
│   ├─ Save to Orders DB (CONFIRMED)
│   └─ Email: Order Confirmed
└─ ❌ NO:
    ├─ Save to DELAYED orders
    └─ Email: Delay notification
```

**AI Validation Logic:**

- **VALID:** Email contains @, phone 9-15 digits, complete address
- **PENDING_VERIFICATION:** Data looks real but incomplete (missing city/ZIP)
- **INVALID:** Fake data, test words, phone <9 digits

![WF1 - Order Intake & Validation](W1.png)

-----

### **WF2 - STALE ORDER CLEANUP**

**Purpose:** Automatically discards orders stuck in PENDING_VERIFICATION for >48 hours.

**Flow:**

```
⏰ Schedule Trigger (every 6h)
↓
📊 Read: Orders with status = PENDING_VERIFICATION
↓
💻 Code: Calculate elapsed time
↓
🔀 IF: >48 hours?
└─ ✅ YES:
    ├─ Copy to Archive DB
    └─ Email: Notify team
```

**Logic:**

```javascript
const now = new Date();
const threshold48h = 48 * 60 * 60 * 1000;
const elapsed = now - new Date(order.Timestamp);
const shouldDiscard = elapsed > threshold48h;
```
### **WF2 - STALE ORDER CLEANUP**
![WF2 - Stale Order Cleanup](W2.png)


-----

### **WF3 - CUSTOMER RESPONSE HANDLER**

**Purpose:** Processes customer email replies with corrected data, prevents infinite loops with retry counter.

**Flow:**

```
📧 Gmail Trigger (polls every 1 min)
    ↓
🤖 AI Extraction (OpenAI GPT-4)
    Extract: orderId, corrected email, phone, address
    ↓
💻 Code: Parse JSON response
    ↓
📊 Read: Fetch order from Manual Verification DB
    ↓
💻 Code: Merge data + increment retryCount
    ↓
🔀 IF: retryCount < 2?
    ├─ ✅ YES:
    │   ├─ Update Manual DB
    │   └─ HTTP Request → Loop back to WF1 webhook
    └─ ❌ NO (≥2):
        └─ Email: Alert team (manual intervention needed)
```

**Loop Prevention:**

* Each order starts with `retryCount = 0`
* After 2 failed re-validation attempts, stops auto-retry
* Team receives alert for manual review

### **WF3 - CUSTOMER RESPONSE HANDLER**
![WF3 - Customer Response Handler](W3.png)

----

### **WF4 - DELAYED ORDERS FOLLOW-UP**

**Purpose:** Monitors delayed orders, automatically unlocks them when stock arrives, sends restock alerts to warehouse.

**Flow:**

```
⏰ Schedule Trigger (every 6h)
    ↓
📊 Read: DELAYED orders
    ↓
📊 Read: Current inventory
    ↓
💻 Code: Compare & calculate restock needs
    ↓
🔀 Switch (by type):
    ├─ 📦 RESTOCK:
    │   └─ Email: Warehouse reorder alert
    └─ 🔓 UNLOCK:
        ├─ Update: Inventory (decrease stock)
        ├─ Update: DELAYED order → CONFIRMED
        ├─ Update: Move to main Orders DB
        └─ Email: Confirmation to customer
```

**Restock Logic:**

```javascript
const REORDER_POINT = 5;
const TARGET_STOCK = 15;

// Scenario A: Delayed orders need stock
if (stock < orderedQty) {
  suggestedOrder = orderedQty - stock;
}

// Scenario B: Stock below threshold
if (stock <= REORDER_POINT) {
  suggestedOrder = TARGET_STOCK - stock;
}
```
### **WF4 - DELAYED ORDERS FOLLOW-UP**
![WF4 - Delayed Orders Follow-up](W4.png)

-----

## 🛠️ Technologies Used

- **n8n** - Workflow automation platform
- **OpenAI GPT-4** - AI-powered data validation and extraction
- **Google Sheets** - Database layer (5 sheets)
- **Gmail API** - Email triggers and notifications
- **JavaScript** - Custom logic in Code nodes
- **Webhooks** - External integration endpoints

-----

## 🚀 Key Features Implemented

### 1. **AI-Powered Validation**

- Real-time customer data quality check
- Smart routing based on data completeness
- Natural language email parsing

### 2. **Loop Prevention Mechanism**

- Retry counter prevents infinite webhook loops
- Automatic escalation to human intervention
- Graceful failure handling

### 3. **Inventory Synchronization**

- Real-time stock updates across workflows
- Accurate stock tracking for multiple concurrent orders
- Automatic delayed order unlocking

### 4. **Time-Based Automation**

- 6-hour schedule for cleanup and follow-ups
- 48-hour threshold for order abandonment
- Timestamp tracking for all operations

### 5. **Smart Notifications**

- Customer order confirmations
- Delay notifications with transparency
- Warehouse restock alerts with quantities
- Team escalation emails

-----

## 📈 System Flow Example

**Scenario: Customer orders 2 mountain bikes**

1. **WF1 receives order** via webhook
1. **AI validates** customer data → VALID ✅
1. **Check inventory** → 10 bikes available
1. **Update stock** → 8 bikes remaining
1. **Save order** with status CONFIRMED
1. **Send email** → “Order confirmed, delivery in 2-4 days”

**If stock was insufficient:**

1. Order saved to **DELAYED** database
1. Customer receives **delay notification**
1. **WF4 monitors** every 6 hours
1. When stock arrives → **auto-unlock**
1. Customer receives **confirmation email**

**If customer data was incomplete:**

1. Order saved to **Manual Verification DB**
1. Customer receives **email requesting correction**
1. **WF3 monitors** Gmail for reply
1. AI extracts corrected data → re-routes to WF1
1. After 2 failed retries → **team alert for manual review**

-----

## 🔒 Error Handling

- **Empty database handling:** Workflows return `[]` gracefully
- **Type conversion:** Automatic string-to-number conversion in comparisons
- **Missing fields:** Default values with `|| 0` and `|| ''`
- **AI parsing errors:** JSON cleaning with regex
- **Retry exhaustion:** Manual intervention fallback

-----

## 📝 Future Improvements

### WF5 - Centralized Error Handler *(planned)*

Currently each workflow fails silently or relies on individual error branches. The next architectural step is a dedicated WF5 configured as the **“Error Workflow”** in n8n’s native settings for all 4 workflows. This would:

- Receive failure events from any workflow automatically
- Log errors consistently to a dedicated Error Log sheet (`timestamp | workflowName | nodeName | errorMessage | inputData`)
- Notify the team with full context in a single place

This avoids duplicating error logic across 4 workflows — a change to the notification format would require updating only WF5 instead of all workflows. Standard pattern in production n8n deployments.

### Additional planned improvements

- Add payment processing integration
- Implement multi-warehouse support
- Create customer dashboard for order tracking
- Add analytics and reporting
- Integrate SMS notifications
- Implement dynamic pricing based on stock levels
- Replace Google Sheets with Postgres/Supabase for atomic inventory transactions
- Add Pub/Sub Gmail trigger to reduce WF3 polling latency

-----

## 👤 Author

**Fabio Roggero**

- 🌍 Languages: Italian, Spanish, English, German
- 🎯 Focus: Workflow automation, n8n, AI integration
- 📧 Contact: [fabio.roggero90@gmail.com](mailto:fabio.roggero90@gmail.com)
- 🔗 LinkedIn: [www.linkedin.com/in/f-roggero](http://www.linkedin.com/in/f-roggero)

-----

## 📄 License

This project is for portfolio demonstration purposes.

-----

**Built with ❤️ using n8n automation**
