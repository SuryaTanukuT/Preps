# Banking Fundamentals (Real Industry View)

## Bank Types (what they do + examples)

| Type            | What they do                         | Examples                                       |
| --------------- | ------------------------------------ | ---------------------------------------------- |
| Retail Bank     | Savings, loans, credit cards         | HDFC Bank, Emirates NBD                        |
| Corporate Bank  | Large company lending, trade finance | HSBC, Qatar National Bank                      |
| Investment Bank | IPO, M&A, trading                    | Goldman Sachs                                  |
| Central Bank    | Monetary policy                      | Reserve Bank of India, Central Bank of the UAE |

---

# Core Banking Operations

## 1) Deposits → CASA Model

* **Savings Account**
* **Current Account**
* **Fixed Deposits**

### How a bank earns (core revenue engine)

**Interest spread = Lending Rate – Deposit Rate**

Example:

* Deposit @ 4%
* Loan @ 9%
* **Spread = 5%**

---

## 2) Loans & Credit Creation

* Personal Loans
* Home Loans
* Business Loans
* Working Capital
* Overdraft
* Credit Cards

**Reality:** Banks don’t lend deposited money directly.
They create credit based on:

* Capital adequacy ratio (Basel norms)
* Risk-weighted assets
  This is where **Basel III** comes in.

---

# Payment Systems (Very Important for Interviews)

## India (domestic rails)

* **UPI** – instant P2P
* **NEFT** – batch settlement
* **RTGS** – high-value real-time
* **IMPS** – 24/7 transfer

## Global rails

* **SWIFT** – cross-border messaging network
* Visa / Mastercard rails
* **SEPA** (Europe)
* **ACH** (US)

---

## Saudi Arabia

* **SARIE** (Saudi Arabian Riyal Interbank Express) → RTGS system for interbank transfers in Riyals
* **Mada** → national debit card network (POS + online)
* **SADAD** → bill payment system (utilities/telecom/govt)
* **Saudi Payments (under SAMA)** → operates instant payment services, supports Vision 2030 digital transformation
* **E-Payments adoption:** by 2023, ~70% retail payments were electronic (as per your note)

## Qatar

* **Qatar Payment System (QPS)** → RTGS for interbank transfers
* **QCB Clearing System** → cheque clearing + retail payments
* **Instant payments:** modernization initiatives for faster retail transfers
* **SWIFT integration:** QPS uses SWIFT messaging standards for reconciliation + settlement

## United Arab Emirates (Dubai angle)

* Central bank oversees payment infrastructure
* **UAESWITCH** → national ATM/POS network connecting banks
* **Instant Payment Platform (IPP)** → real-time transfers (IMPS/UPI-like)
* **Direct Debit System (DDS)** → recurring payments (utilities, loans)
* **Wages Protection System (WPS)** → salary payments to workers via banks/exchanges
* Payment gateways commonly used for e-commerce:

  * Network International
  * PayTabs
  * Telr
  * Checkout.com

---

# Credit Cards (Deep Understanding)

## Lifecycle (end-to-end)

1. Customer applies
2. Underwriting (CIBIL/credit score checks)
3. Credit limit assigned
4. Transaction happens
5. Authorization → Issuer → Network → Merchant
6. Settlement
7. Billing cycle
8. Interest if unpaid

## Revenue streams

* Interest
* Interchange fee
* Annual fee
* Late payment fee

---

# Locker Management (Retail core systems)

Used in: retail banking core systems

**Tech flow**

* Account linked
* Rental charge auto-debited
* Audit logging required
* Dual control access

---

# Risk Management (Very Important)

## Credit risk

Borrower may default
Mitigation:

* Credit scoring
* Income verification
* AI scoring models

## Market risk

* Interest rate fluctuation
* Forex exposure

## Operational risk

* System failures
* Fraud
* Cyberattacks

**Middle East banks strongly focus on**

* Fraud prevention
* AML automation
* Real-time transaction monitoring

---

# Regulatory Frameworks

## Basel III

* Capital adequacy norms
* Liquidity coverage ratio
* Stress testing

## AML / KYC

* Identity verification
* Suspicious transaction monitoring
* PEP checks
* Sanctions screening

## Regulators (by region)

* India: RBI / SEBI / IRDAI
* Middle East: UAE Central Bank / Qatar Central Bank / SAMA (Saudi)

---

# Financial Services (Non-Bank Side)

## Capital markets

* Stocks
* Bonds
* Derivatives
* Clearing & settlement

## Asset management

* Mutual Funds
* ETFs
* PMS

Revenue:

* AUM fees
* Brokerage fees

---

# Insurance Deep Dive

## Life insurance

* Term
* Whole life
* ULIPs

## Non-life

* Motor
* Health
* Travel
* Marine

**Model:** Premium → Risk pool → Claims → Reinsurance

Key concepts:

* Underwriting
* Actuarial modeling
* Claims automation
* Fraud detection

---

# Technology in BFSI (Your strong area)

## Core banking software (examples)

* Finacle
* Oracle FLEXCUBE
* Temenos

These systems handle:

* Accounts
* Ledger
* Transactions
* GL
* Interest calculation

---

# Typical BFSI Architecture (Interview Gold)

## Core Layer

* Core Banking
* Loan Management
* Card Management

## Integration Layer

* API Gateway
* ESB
* Kafka / MQ

## Channel Layer

* Mobile banking
* Internet banking
* ATM
* POS

## Compliance Layer

* AML engine
* Fraud engine
* Audit logging

---

# Cybersecurity in BFSI

* HSM (Hardware Security Module)
* Tokenization
* PCI-DSS compliance
* 2FA / MFA
* OAuth2 / OIDC
* Zero Trust

---

# Middle East Special Angle

Banks there focus on:

* Islamic Banking (Sharia-compliant finance)
* Trade Finance
* Remittances
* Cross-border payments
* Strong AML compliance
* High Net Worth Individual wealth platforms

**You should learn**

* Murabaha financing
* Sukuk bonds
* Sharia audit systems

---

# AI in BFSI

* Fraud detection models
* Behavioral analytics
* Credit risk ML scoring
* Chatbots
* Document OCR
* KYC automation

---

# Global Trends

* ESG investing
* CBDCs
* Open banking APIs
* Embedded finance
* Real-time payments
* Financial inclusion

---

# What You Should Master for Interviews

## Business understanding

* How bank earns money
* What is NPA
* What is CASA ratio
* What is LTV
* What is CAR

## Technical understanding

* Idempotency in payments
* Ledger design
* Event-driven architecture
* Distributed transactions
* Saga pattern
* Audit compliance logging

## Architecture understanding

* Database per service
* Strong consistency for ledger
* Async processing for notifications
* Redis for rate limiting
* Kafka for settlement events

**If you explain BFSI like this in interview**

> “In BFSI, we must ensure ACID compliance for ledger integrity, implement idempotent payment APIs, integrate AML checks asynchronously, maintain audit logs for regulatory reporting, and use distributed tracing for observability.”

---

# How Fintech Fits Into BFSI

* Banking → Digital payments, neobanks
* Financial Services → WealthTech, trading
* Insurance → InsurTech automation
* Regulation → RegTech AI compliance

**Fintech = Technology layer on top of traditional BFSI.**

---

# 1) Complete BFSI Payment System Deep Design

## A) What a “payment system” must guarantee (banking reality)

**Non-negotiables**

* Ledger correctness (double-entry): money never “disappears”
* Idempotency: retries must not double-debit
* Exactly-once posting (or effectively-once via idempotency keys + atomic ledger TX)
* Reconciliation: internal ledger ↔ bank statements / network settlement files
* Auditability: immutable trail + who/when/why
* Operational resilience: timeouts, reversals, chargebacks, disputes

## B) Payment types you’ll be asked about

* Account-to-Account (A2A): UPI/IMPS/NEFT/RTGS (India), local rails in GCC
* Card payments: Visa/Mastercard authorization + clearing + settlement
* Cross-border: SWIFT messages; migration to ISO 20022 for richer payment data
* Wallet / stored value: internal wallet ledger + external top-ups/payouts

---

# 5) 30-day Structured BFSI Mastery Roadmap (Daily)

## Week 1 — Banking + payments fundamentals

* D1 Bank revenue model (CASA, NIM), core ops
* D2 Deposits/loans lifecycle + underwriting basics
* D3 Payment rails: UPI/IMPS/NEFT/RTGS vs card vs SWIFT
* D4 Card flow deep: auth/clearing/settlement/chargeback
* D5 Risk types: credit/market/ops + examples
* D6 Regulatory basics: AML/KYC, audit, data retention
* D7 Summarize as a 2-minute “banking domain pitch”

## Week 2 — Ledger & core system design

* D8 Double-entry ledger concepts + invariants
* D9 Ledger tables & posting engine design
* D10 Reconciliation pipelines + breaks management
* D11 Idempotency, retries, outbox, saga in payments
* D12 Settlement models (T+0/T+1/T+2), net vs gross
* D13 Multi-currency + FX posting rules
* D14 Mini-project: build a tiny ledger + payment order state machine

## Week 3 — Security, compliance, AML/fraud

* D15 OAuth/OIDC + mTLS + secrets/HSM concepts
* D16 PCI mindset + tokenization (card data handling)
* D17 KYC workflow + sanctions/PEP screening
* D18 Transaction monitoring rules (velocity/geo/merchant)
* D19 Fraud scoring: device fingerprinting, behavioral patterns
* D20 Case management + audit trails + reporting
* D21 Mini-project: “risk gate” service that approves/holds payments

## Week 4 — Middle East specialization + interview readiness

* D22 Islamic banking principles + contract type
* D23 Murabaha & Ijara flows as product/ledger postings
* D24 Sukuk + wealth products overview
* D25 Cross-border payments + ISO 20022 data richness
* D26 Observability: traces per payment + recon SLAs
* D27 Ops: incident playbooks, reversals, customer disputes
* D28 System design mock: “wallet + payouts + reconciliation”
* D29 System design mock: “card issuing + chargebacks”
* D30 Compile: diagrams + 30 interview Q&A bullets

---

# Tech Perspective (Your Lens as a Lead Engineer)

**Banking system = Consistency Engine + Audit Trail + Security Layer**

Every transaction must guarantee:

* **Debit - Credit = 0 ALWAYS**
* No data loss
* No duplication
* No silent failure
* No race conditions
* Full audit history

That’s why banks care about:

* ACID
* Locking
* Isolation
* Idempotency
* Encryption
* Dual control
