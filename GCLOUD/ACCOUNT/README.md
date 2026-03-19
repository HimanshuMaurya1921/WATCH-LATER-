# GCP ACCOUNT

## **Resource**

**Service-level resources**

- Compute Instance VM’s
- Cloud Storage buckets
- Cloud SQL databases

**Account-level resources**

- Organization
- Folders
- Projects

![image.png](GCP%20ACCOUNT/image.png)

# **Billing Account**

- Defines who pays for a given set of Google Cloud resources
- Tracks all costs incurred by Google Cloud usage
- Linked to a Payments profile
- Can be linked to one or more projects
- Billing specific roles and permissions to control access

![image.png](GCP%20ACCOUNT/image%201.png)

- Self-service (online) or Invoiced (offline) payments available
    - NOT EVRYONE WILL BE ELIGEBLE FOR OFFLINE-PAYMENT (INVOICED)
- Sub-accounts can be used for resellers
- SAME Billing account can pay for projects in a different organization
- Projects that are not linked to a Cloud Billing account cannot use paid Google Cloud services

# **Payments Profile**

- Processes payments for all Google services
- Stores all payment methods
- Single pane of glass for viewing invoices and payment history
- Controls who can view and receive invoices
- Individual or Business profile types – cannot be changed
    - You **cannot convert** an individual profile into a business profile.
    - But you can create a **new payments profile**.

# Billing ROLE (SHORT-NOTE)

This is a crucial hierarchy to understand in GCP because separating "who pays" (Billing Account) from "who builds" (Project) is a key DevOps pattern.

Here is the short explanation for each role shown in your dropdown:

### **1. Billing Account Administrator (The "Owner")**

- **Power:** Full control.
- **What they do:** They can update credit cards, set budgets, view all invoices, and assign roles to other people.
- **Use Case:** The Finance Head or CTO.
- NOTE : The **Billing Account Administrator** role (`roles/billing.admin`) gives you full control over an **existing** billing account, but it does **not** grant permission to create a *new* one.

### **2. Billing Account Creator**

- **Power:** Creation only.
- **What they do:** Can create *new* self-serve billing accounts.
- **Use Case:** Developers who need to set up their own personal billing for experiments.

### **3. Billing Account User (The "Spender")**

- **Power:** Link projects.
- **What they do:** They cannot see credit card details or invoices. They can only **link** a Project to the Billing Account to start spending money.
- **Use Case:** A Lead Developer who needs to launch a new project but shouldn't have access to the company bank details.

### **4. Billing Account Viewer (The "Auditor")**

- **Power:** Read-only.
- **What they do:** Can view spending, invoices, and budgets, but cannot change anything.
- **Use Case:** An external auditor or a junior accountant.

### **5. Project Billing Manager (The "Linker")**

- **Power:** Project-level control.
- **What they do:** This role lets a user enable or disable billing **for a specific project**.
- **Key Difference:** Unlike the "Billing Account User" (who works at the bank account level), this role is often used at the *Project* level to let someone attach/detach the billing account without giving them full Project Owner rights.

# NEW TOPIC : COST-MANAGEMENT (BUDGET)

# **Committed Use Discounts (CUD’s) (IMP)**

- Discounted prices when you commit to using a minimum level of resources for a specified term
- 1 or 3 Year Commitment

**Commitment Types**

- Spend-based
- Resource-based

NOTE : The commitment fee is billed monthly

# **Spend-based commitment (CUD’s)**

- Discount for a commitment to spend a minimum amount for a service (hours) in a particular region
- 25% discount for 1 year – 52% discount on a 3 year

**Available for**

- Cloud SQL database instances
- Google Cloud VMWare Engine

NOTE : Applies only to CPU and memory usage

# **Resource-based commitment**

- Discount for a commitment to spend a minimum amount for Compute Engine resources in a particular region
- For use across Projects (CUD Sharing )

**Available for**

- vCPU, Memory, GPU and Local SSD
- 57% discount for most resources
- 70% for memory-optimized machine types

# **Sustained-use discounts**

- NOTHING MUCH TO DO ON OUR SIDE
- Automatic discounts for running Compute Engine resources a significant portion of the billing month
- Applies to vCPUs and memory for most Compute Engine instance types
- Includes VM’s created by GKE
- UP TO 30% DISCOUNT
- Does not apply to App Engine flexible, Dataflow and E2 machine types

# **Google Cloud's pricing calculator**

[Google Cloud Pricing Calculator](https://cloud.google.com/products/calculator)

# **Cloud Billing Budgets**

![image.png](GCP%20ACCOUNT/image%202.png)

- Enables you to track your actual Google Cloud spend against your planned spend
- Budget alert threshold rules that are used to trigger email notifications to help you stay informed about your spend

## Define the scope of the budget

- Spend of billing account or more granular
- Budget amount can be set to a specified total, or based on previous month's spend
- Alert emails are sent to billing account admins and specific users when costs exceed a percentage of the budget

![image.png](GCP%20ACCOUNT/image%203.png)

Email recipients can be customized by using Cloud Monitoring to specify other people to receive budget alert emails

NOTE  : Use Pub/Sub for programmatic notifications or to automate cost management tasks

# **Billing Export**

- Billing export enables granular billing data (such as usage, cost details, and pricing data) to be exported automatically to BigQuery for detailed analysis
- NOTE:  Not retroactive (**taking effect from a date YOU TURN IT ON , NOT FROM PREVIOUS DATE**)
- TYPE OF CLOUD BILLING DATA THAT WE CAN EXPORT
    - **Standard usage cost**
    - **Detailed usage cost**
    - **Pricing**
    - **Committed Use Discounts Export**
        - updated each day with a consolidated view of your spend-based CUD commitments.

# SETTING UP AN ADMIN USER

![image.png](GCP%20ACCOUNT/image%204.png)

# CLI : Google-cloud sdk

[Gcloud-cmd.pdf](GCP%20ACCOUNT/Gcloud-cmd.pdf)

# gcloud command format

![image.png](GCP%20ACCOUNT/image%205.png)

# GCP — Limits & Quotas (Complete Notes)

---

## What are Quotas?

A **hard limit** on how much of a particular GCP resource your project can consume. Google enforces these so one project doesn't eat the entire cloud buffet.

---

## Types of Quotas

| Type | Description |
| --- | --- |
| **Rate Quota** | Limits API call frequency — **resets after a specified time** (e.g. 1000 requests/min) |
| **Allocation Quota** | Limits resource count (e.g. VMs, IPs) — **must be explicitly released**, doesn't auto-reset |

---

## Why Quotas are Enforced

- **Protection** — prevents accidental/malicious overuse
- **Resource Management** — fair distribution across all GCP users
- **Countable** — every resource usage is tracked and measurable

---

## Viewing Quotas

**Option 1 — Quick view:**

> IAM & Admin → Quotas
> 

**Option 2 — Detailed per-API view:**

> APIs & Services → Select Enabled API → **"Quotas & System Limits"** tab
> 
- Shows current usage, 7-day peak, and all 3,769 system limits (yes, really)
- Can set up **quota alerts** before hitting the limit *(pro tip for interviews!)*

---

## Requesting a Quota Increase

- Select quota → **Edit Quotas** → Enter new limit + justification
- Mention: intent of usage, future growth plans, region spread
- Approval time: **up to 2 business days**

---

## Monitoring & Alerting

- When quota is **nearly exhausted** → GCP triggers an alert via **Cloud Monitoring**
- When quota is **fully exhausted** → API returns **HTTP 429** with error `ResourceExhausted`
- Set up alert policies via: APIs & Services → Quotas & System Limits → **Manage Alert Policies**

---

## Quick Interview Cheatsheet

- Rate Quota → **time-based reset** | Allocation Quota → **manual release**
- Quota exceeded → **HTTP 429**
- Increase request → **up to 2 business days**
- System Limits = **hard caps**, cannot be changed even by Google support
