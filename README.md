
# 🚴 Bicycle Order Management System - n8n Automation

## 📋 Project Overview

An end-to-end automated order management system for bicycle e-commerce, built entirely with **n8n workflows**. The system handles order intake, AI-powered data validation, inventory management, customer follow-ups, and automated restocking alerts.

**Key Features:**
- 🤖 AI-powered customer data validation (OpenAI GPT-4)
- 📦 Real-time inventory tracking and stock management
- 🔄 Automated customer response processing with loop prevention
- ⏰ Time-based order cleanup (>48h auto-discard)
- 📧 Smart email notifications to customers and warehouse team
- 🔓 Automatic order unlock when stock becomes available

---

## 🏗️ System Architecture

The system consists of **4 interconnected workflows**:

```

W1: Order Intake \& Validation
↓
W2: Stale Order Cleanup (runs every 6h)
↓
W3: Customer Response Handler
↓
W4: Delayed Orders Follow-up (runs every 6h)

```

---

## 📊 Database Structure

### Google Sheets Databases:

1. **Inventory - Warehouse**
   - Bicycle Models | stock | price | orderStatus | lastUpdated

2. **Orders - Monthly**
   - id | CustomerName | Email | phone | Shipping Address | Bicycle Models | quantity | price | notes | orderStatus | Timestamp

3. **Manual Verification DB**
   - id | CustomerName | email | phone | Shipping Address | Bicycle Models | quantity | price | orderStatus | notes | Timestamp | validatedAt | retryCount

4. **Orders DELAYED**
   - id | CustomerName | Email | phone | Shipping Address | Bicycle Models | quantity | price | notes | orderStatus | Timestamp

5. **Archive / NO_SHOW_UP**
   - id | customerName | Phone | Email | Shipping Address | DISCARDED | Timestamp | discardedAt

---

## 🔄 Workflow Details

### **W1 - ORDER INTAKE & VALIDATION**

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

---

### **W2 - STALE ORDER CLEANUP**

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


---

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
    │   └─ HTTP Request → Loop back to W1 webhook
    └─ ❌ NO (≥2):
        └─ Email: Alert team (manual intervention needed)
```

**Loop Prevention:**

- Each order starts with `retryCount = 0`
- After 2 failed re-validation attempts, stops auto-retry
- Team receives alert for manual review

---

### **W4 - DELAYED ORDERS FOLLOW-UP**

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


---

## 🛠️ Technologies Used

- **n8n** - Workflow automation platform
- **OpenAI GPT-4** - AI-powered data validation and extraction
- **Google Sheets** - Database layer (5 sheets)
- **Gmail API** - Email triggers and notifications
- **JavaScript** - Custom logic in Code nodes
- **Webhooks** - External integration endpoints

---

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

---

## 📈 System Flow Example

**Scenario: Customer orders 2 mountain bikes**

1. **W1 receives order** via webhook
2. **AI validates** customer data → VALID ✅
3. **Check inventory** → 10 bikes available
4. **Update stock** → 8 bikes remaining
5. **Save order** with status CONFIRMED
6. **Send email** → "Order confirmed, delivery in 2-4 days"

**If stock was insufficient:**

1. Order saved to **DELAYED** database
2. Customer receives **delay notification**
3. **W4 monitors** every 6 hours
4. When stock arrives → **auto-unlock**
5. Customer receives **confirmation email**

---

## 🔒 Error Handling

- **Empty database handling:** Workflows return `[]` gracefully
- **Type conversion:** Automatic string-to-number conversion in comparisons
- **Missing fields:** Default values with `|| 0` and `|| ''`
- **AI parsing errors:** JSON cleaning with regex
- **Retry exhaustion:** Manual intervention fallback

---

## 📝 Future Improvements

- [ ] Add payment processing integration
- [ ] Implement multi-warehouse support
- [ ] Create customer dashboard for order tracking
- [ ] Add analytics and reporting
- [ ] Integrate SMS notifications
- [ ] Implement dynamic pricing based on stock levels

---

## 👤 Author

**Fabio Roggero**

- 🌍 Languages: Italian, Spanish, English, German
- 🎯 Focus: Workflow automation, n8n, AI integration
- 📧 Contact: fabio.roggero90@gmail.com
- 🔗 LinkedIn: www.linkedin.com/in/f-roggero



---

## 📄 License

This project is for portfolio demonstration purposes.

---

**Built with ❤️ using n8n automation**

```

