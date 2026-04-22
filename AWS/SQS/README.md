# 📄 Amazon SQS Cost & Design Considerations
---

## 1. Overview

Amazon SQS (Simple Queue Service) is a fully managed message queuing service used to decouple distributed systems and ensure reliable message delivery.

It supports:

- **Standard Queues** (high throughput, best-effort ordering)
- **FIFO Queues** (strict ordering, exactly-once processing)

---

## 2. Pricing Model

SQS pricing is primarily based on **API requests**, not just messages.

### 🔹 Key Pricing Points

- First **1 million requests per month are free**
- After that:
    - **Standard Queue** → ~$0.40 per 1 million requests
    - **FIFO Queue** → ~$0.50 per 1 million requests

---

### 💰 FIFO Pricing Example

- Cost = **$0.50 per 1 million requests**
- For **10 million requests**:

```
10 × $0.50 = $5
```

👉 **Total = $5**

---

## 3. What Counts as a “Request”

Each of the following is billed as a request:

- SendMessage
- ReceiveMessage
- DeleteMessage
- ChangeMessageVisibility

⚠️ Important:

> One message typically results in **3 requests**
> 
> 
> (Send + Receive + Delete)
> 

---

## 4. Payload Size Consideration

- SQS pricing is based on **64 KB chunks**

### 📦 Rules:

- Message ≤ 64 KB → **1 request**
- Message > 64 KB → multiple requests

### Example:

- 128 KB message → **2 requests**
- 256 KB message → **4 requests**

👉 Larger payload = higher cost

---

## 5. Retry & Processing Impact

- Each retry increases request count
- Failed processing can significantly increase cost

### Example:

- 1 message with 5 retries:
    - ~15+ requests instead of 3

---

## 6. Polling Strategy (Cost Driver)

### ❌ Inefficient (Bad Polling)

- Constant polling with no delay
- Empty queue polling
- Too many idle consumers

👉 Leads to **high number of empty billed requests**

---

### ✅ Efficient (Recommended)

- Use **Long Polling** (`WaitTimeSeconds = 10–20`)
- Reduce empty responses
- Optimize number of consumers

---

## 7. Standard vs FIFO Cost Consideration

| Feature | Standard Queue | FIFO Queue |
| --- | --- | --- |
| Cost per 1M requests | ~$0.40 | ~$0.50 |
| Ordering | Best-effort | Strict |
| Throughput | Very high | Limited |
| Deduplication | ❌ | ✅ |

---

## 8. Additional Cost Factors

### 🔹 Data Transfer

- Same region → minimal cost
- Cross-region → additional charges

### 🔹 Encryption (KMS)

- Each encrypted request may incur extra cost

### 🔹 Large Payloads (S3 Integration)

- Messages > 256 KB stored in S3
- Adds:
    - S3 storage cost
    - S3 request cost

---

## 9. Cost Estimation Formula

```
Total Cost ≈
(Requests per message × Total messages × Price per request)
+ Additional costs (Data transfer, KMS, S3)
```

---

## 10. Key Recommendations

- Optimize **message size** (keep ≤ 64 KB where possible)
- Use **long polling** to reduce empty requests
- Control **retry behavior**
- Avoid unnecessary consumers
- Choose **FIFO only when strict ordering is required**

---

## 🧠 Clean Recommendation

If reliability matters (and clearly it does):

👉 Never rely on SNS alone for critical delivery

Use:

SNS → fan-out

SQS → durability + retries + ACK control

DLQ → failure handling
