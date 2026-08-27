# x402 Working Group: Tax & Audit Transparency Strawman Specification

> **Document Status**: Working Group Strawman Proposal (PR #1 Ready)  
> **Target Working Group**: x402 Tax & Accounting Working Group (`x402-foundation/wg-tax`)  
> **Author**: Walter Hawkins (`@whawk46`)  
> **Companion References**: IETF SCITT Working Group (`scitt@ietf.org`) · Issue #3226 · RFC 8785 (JCS) · Section 1202 QSBS / Delaware C-Corp

---

## Abstract

This specification defines a lightweight, non-bloating tax jurisdiction discovery and immutable audit transparency extension for the x402 payment protocol.

Addressing the sub-cent micro-tax problem, this standard:
1. **Enforces Strict Settle-Only Accounting**: Mandates that tax, ledger, and volume counters trigger strictly upon completed `/settle` execution (`result.success`), entirely decoupling pre-flight `/verify` checks to eliminate phantom catalog volume.
2. **Standardizes Upfront Tax Discovery**: Advertises tax jurisdiction and Merchant-of-Record (MoR) status upfront via DNS (`_x402`) and `/.well-known/x402`.
3. **Generates Cryptographic Audit Receipts**: Registers deterministic JSON Canonicalization Scheme (RFC 8785 JCS) accounting receipts onto SCITT Transparency Services for Section 1202 QSBS audit readiness.

---

## 1. Problem Statement & Multi-Chain Consensus

### 1.1 The `/verify` vs `/settle` Conflation (Issue #3226 Data)
Production telemetry across independent implementations (EVM/Base, Stellar/Soroban commit `fae6daa9`, and multi-enclave facilitators) revealed a shared architectural defect:
* Facilitators previously incremented volume and tax counters on `/verify` (`isValid: true`).
* `/verify` proves only that a signature *could* settle, not that capital transferred.
* **Resolution**: Invariant rule that tax and volume ledgers MUST trigger exclusively on final `/settle` execution.

### 1.2 EIP-3009 Event-Driven Ledger Disambiguation
Single-boolean status checks (`authorizationState() == true`) conflate execution with cancellation. In conformance with working group telemetry (zero `AuthorizationCanceled` events across ~108k Base blocks), accounting ledgers MUST derive execution strictly from `AuthorizationUsed` and `AuthorizationCanceled` event logs.

---

## 2. Protocol Extension: `tax-transparency`

### 2.1 Upfront Discovery Manifest
Resource servers advertise jurisdiction, VAT/EIN registration, and Merchant-of-Record (MoR) routing in their `_x402` DNS TXT record or origin manifest:

```dns
_x402.merchant.example. IN TXT "v=x402-1; wk=https://merchant.example/.well-known/x402; k=resource-server; jur=US-DE; mor=facilitator.example"
```

### 2.2 Tax Metadata Schema
```json
{
  "version": "x402-1",
  "extensions": {
    "tax-transparency": {
      "jurisdiction": "US-DE",
      "merchantOfRecord": "Facilitator Treasury, Inc.",
      "vatOrTaxId": "US-EIN-XX-XXXXXXX",
      "taxIncluded": false,
      "rateBasisPoints": 0,
      "protocolAccrualBps": 20,
      "auditReceiptService": "https://scitt.transparency.org"
    }
  }
}
```

---

## 3. Cryptographic Audit Receipts (SCITT & JCS)

### 3.1 Deterministic JCS Canonicalization (RFC 8785)
All accounting entries, settlement summaries, and tax invoices MUST be canonicalized using RFC 8785 (JCS) before signing, eliminating hash malleability across heterogeneous accounting databases.

### 3.2 SCITT Signed Accounting Statements
Upon settlement, the facilitator emits a signed statement registered to a SCITT Transparency Service containing:
* **Settlement Hash & Block Height**
* **Gross Transaction Value & Currency (USDC, RLUSD, FLR)**
* **Facilitator Protocol Accrual (0.20% / 20 bps)**
* **Tax Line Allocation & Jurisdiction Code**
* **Cryptographic Enclave Attestation Hash**

```json
{
  "statementType": "https://x402.org/schemas/v1/tax-receipt",
  "settlementHash": "0x4b78a912e6...",
  "grossAmount": "100.000000",
  "currency": "USDC",
  "netPayee": "99.800000",
  "protocolTreasury": "0.200000",
  "taxAmount": "0.000000",
  "jurisdiction": "US-DE",
  "timestamp": 1787806800,
  "enclaveMeasurement": "0x7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069"
}
```

---

## 4. Multi-Chain Settlement & Corporate Hygiene
* **Delaware C-Corp & QSBS Alignment**: Provides immutable cryptographic paper trails required for Section 1202 Qualified Small Business Stock compliance.
* **Non-Custodial Fee Accrual**: Directs 20 bps protocol fees into cold multi-sig governance contracts (`Safe.global` / XRPL Multisig) with zero intermediate custody risk.
