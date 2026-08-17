# Build-Start

# Automated Bank Payment Verification System

An intelligent, low-cost, and robust payment verification and fraud detection prototype built for WhatsApp business agents. The system validates bank transfer receipts against active orders and asynchronous bank SMS notifications without requiring direct bank APIs.

---

## 1. Architecture Overview

The system is built on a high-throughput Java Servlet architecture backed by a MySQL relational database. It operates on a **4-tier verification pipeline** that minimizes computational overhead and API costs while maintaining strict fraud prevention:
---

## 2. Setup Instructions

### Prerequisites
* **Java Development Kit (JDK):** Version 17 or higher
* **Application Server:** Apache Tomcat 9.0+ (or Tomcat 10 with `javax.*` servlet runtime)
* **Database:** MySQL Server 8.0+
* **Build Tool:** Apache Maven 3.8+
* **IDE:** Eclipse / Spring Tool Suite (STS) / IntelliJ IDEA

---

### Step 1: Database Setup
1. Open your MySQL client (Workbench or CLI).
2. Execute the initialization script located at `src/main/resources/schema.sql`:
   ```bash
   mysql -u root -p < src/main/resources/schema.sql

Customer Upload / WhatsApp Bot
              │
              ▼
    [ PaymentSubmissionServlet ]
              │
    ┌─────────┴─────────┐
    ▼                   ▼
1. SHA-256 Hash    2. Text / Pattern Extraction
(Deduplication)       (Amount, Ref, Account, Date)
    │                   │
    └─────────┬─────────┘
              ▼
    3. Deterministic Rule Engine
    (Order Matching, Account Whitelist, Reference Check)
              │
              ▼
    4. Bank SMS Reconciliation
    (Dual-Key Matching on Amount + Reference Number)
              │
              ▼
    5. Discrete Decision Output
   [ APPROVED | REJECTED | NEEDS VERIFICATION ]
              │
    ┌─────────┴─────────┐
    ▼                   ▼
Customer Feedback  Admin Audit Dashboard
(Contextual Alert) (Side-by-Side Review)

Step 2: Configure Database Credentials
Open src/main/java/com/buildstart/util/DBUtil.java and verify your local database credentials:

Java
config.setJdbcUrl("jdbc:mysql://localhost:3306/buildstart_payments?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC");
config.setUsername("root");
config.setPassword("YOUR_MYSQL_PASSWORD");

Step 3: Build and DeployOption 
A: Running Inside Eclipse / Spring Tool Suite (STS)
B: Maven CLI Build

Step 4: Access the Interfaces
WhatsApp Bot Simulator: http://localhost:8080/payment-verification/index.html  
Admin Review Hub: http://localhost:8080/payment-verification/admin.html  
Inbound Bank SMS Simulator: http://localhost:8080/payment-verification/sms-simulator.html  

Major Design Decisions
1.Deterministic Rule Engine over Pure Generative AI: Relying exclusively on LLMs for payment approval introduces non-deterministic hallucinations and high operational costs. A strict, rule-based Java engine ensures reproducible, auditable verification.  
2.Dual-Key Bank SMS Reconciliation: To prevent false approvals when multiple customers pay identical amounts simultaneously, transactions must match on both extracted_amount AND extracted_ref.  
3.Three-State Decision Framework: Every verification concludes as APPROVED, REJECTED, or NEEDS VERIFICATION with distinct internal rationale and customer-friendly feedback.  
4.Customer vs. Internal Privacy Separation: Clear, non-revealing explanations are sent to customers (e.g., requesting a clearer slip) without exposing internal anti-fraud trigger logic.  

Fraud-Handling Approach
Exact Duplicate Uploads: Instant SHA-256 cryptographic image hashing detects identical re-uploads across submissions without calling OCR or AI models.  
Slip Recycling Across Customers: Uniqueness constraints and indexed queries on extracted_ref block approved slips from being re-used for other orders.  
Account Manipulation: Slips showing transfers to personal or unapproved third-party accounts are immediately flagged (WRONG_RECEIVING_ACCOUNT) and rejected.  Tampered Amounts & Fonts: Mismatched amounts trigger immediate rejection (AMOUNT_MISMATCH). Visual inconsistencies and unreadable data escalate to NEEDS VERIFICATION for employee inspection. 

Cost Considerations
The system implements a tiered escalation model to achieve low operating costs:  
Tier 1 ($0.00): Image hashing and database deduplication eliminate re-processing costs on repeated submissions.  
Tier 2 (Micro-cost / Local): Regex-based text parsing and local heuristic OCR (Tess4J) handle standard receipts locally.  
Tier 3 ($0.00): Relational SQL operations handle state validation and SMS reconciliation.
Tier 4 (Human/AI Escalation): Expensive AI vision models or manual employee reviews are reserved exclusively for borderline, unreadable, or conflicted submissions. 

Known Limitations
Receipt Layout Diversity: Non-standard, handwritten, or degraded receipts require fallback to manual review.  
Synchronous Upload Lifecycle: In this prototype, image parsing occurs inside the HTTP request loop; heavy volume requires an asynchronous message broker.  
Delayed SMS Delivery: If a bank's SMS alert is delayed due to telecom latency, the transaction remains in NEEDS VERIFICATION until the SMS arrives or an employee manually approves it.  
