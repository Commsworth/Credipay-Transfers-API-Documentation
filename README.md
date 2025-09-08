# Credipay Transfers API (SaaS Integrations)

> **Scope:** This doc focuses on **bank transfer** flows only — using **initiate**, **complete**, and **simulate-fund-transfer**. Card endpoints are intentionally omitted.

---

## 📄 Overview

With Credipay APIs, you can collect payments and run transactions across Nigeria via multiple channels. To get started:

- **Create a free account:** Sign up at your Credipay portal to access a **test environment**.
- **Build & test:** Simulate payments with mock data, perform test transfers to dummy accounts, and customize your integration.
- **KYC → Live:** Complete business verification to activate live processing and receive **live keys**.
- **Two modes:** **Test** mode uses no real funds; **Live** mode moves real money. Test keys typically include a `_test` suffix.

> These docs assume you’re familiar with REST APIs and can make HTTP requests from your backend.

---

## 🔐 Authentication & API Keys

Credipay requires an API key on every server-side request. For **Transfer** endpoints, use the header:

```
x-api-key: YOUR_SECRET_KEY
```

### Where to get your key (Dev Portal → Webhooks)
1. Log in to the Credipay portal.
2. Go to **Webhooks**.
3. View and copy your keys from the page.
4. If a key is exposed/compromised, **Generate new keys** on the same page and rotate them in your app.

> Tip: You will have separate keys for **Test** and **Live** environments. Ensure you’re using the correct one.

---

## 🔁 Transfer Flow (High-Level)

1. **Initiate Transfer** — Generate a **virtual account** (account number, bank, name) for a specific payment reference and amount.
2. **Customer Pays** — Payer transfers the amount to the provided virtual account.
3. **(Optional in Test) Simulate Funding** — In **Test** mode, simulate inbound transfer to the generated account.
4. **Complete Transfer** — Confirm you’re done with this payment reference so it can settle/close.
5. **Receive Webhook** — Your webhook receives the final status and details for reconciliation.

> Always keep a resilient webhook endpoint to receive asynchronous status updates.

---

## 🔧 Base URL & Headers

```
Base URL: {{baseUrl}}
Header:   x-api-key: YOUR_SECRET_KEY
Content:  application/json
```

---

## 1) Initiate Transfer

Generate a unique virtual account for a payer to fund.

**Endpoint**  
```
POST {{baseUrl}}/api/v1/Gateway/initiate-payment
```

**Headers**
```
x-api-key: YOUR_SECRET_KEY
Content-Type: application/json
```

**Request Body (example)**
```json
{
  "amount": 10000,
  "firstName": "John",
  "lastName": "Doe",
  "email": "johndoe@gmail.com",
  "phone": "08133061111",
  "paymentReference": "JOHN-DOE-0002",
  "paymentChannel": "transfer",
  "currencyCode": "NGN"
}
```

**cURL**
```bash
curl -X POST "{{baseUrl}}/api/v1/Gateway/initiate-payment"   -H "x-api-key: YOUR_SECRET_KEY"   -H "Content-Type: application/json"   -d '{
    "amount": 10000,
    "firstName": "John",
    "lastName": "Doe",
    "email": "johndoe@gmail.com",
    "phone": "08133061111",
    "paymentReference": "JOHN-DOE-0002",
    "paymentChannel": "transfer",
    "currencyCode": "NGN"
  }'
```

**Success Response (abridged)**
```json
{
  "isSuccess": true,
  "value": {
    "amount": 10000,
    "transactionReference": "TRX_...",
    "paymentReference": "JOHN-DOE-0002",
    "currencyCode": "NGN",
    "accountInfo": {
      "accountName": "John Doe",
      "accountNumber": "9XXXXXXXXX",
      "bankName": "Credipay Test Bank"
    }
  }
}
```

> Use the returned **accountInfo** to instruct the customer where to send the funds. Keep **paymentReference** unique per attempt/order.

---

## 2) (Test Only) Simulate Funding

In **Test** mode, you can simulate an inbound bank transfer to the generated account. This helps you drive your integration end‑to‑end without real money.

**Endpoint**  
```
POST {{baseUrl}}/api/v1/Gateway/simulate-fund-transfer
```

**Headers**
```
x-api-key: YOUR_SECRET_KEY
Content-Type: application/json
```

**Request Body (example)**
```json
{
  "recipientAccountNumber": "9000060980",
  "amount": 10000
}
```

**cURL**
```bash
curl -X POST "{{baseUrl}}/api/v1/Gateway/simulate-fund-transfer"   -H "x-api-key: YOUR_SECRET_KEY"   -H "Content-Type: application/json"   -d '{
    "recipientAccountNumber": "9000060980",
    "amount": 10000
  }'
```

> After simulating, Credipay will dispatch a **webhook** to your configured URL with the transaction details. Always reconcile against your webhook events.

---

## 3) Complete Transfer

Call this when the payer has funded the generated account (or after you’ve simulated funding in Test).

**Endpoint**  
```
POST {{baseUrl}}/api/v1/Gateway/complete-payment
```

**Headers**
```
x-api-key: YOUR_SECRET_KEY
Content-Type: application/json
```

**Request Body (example)**
```json
{
  "paymentReference": "JOHN-DOE-0002",
  "paymentChannel": "transfer"
}
```

**cURL**
```bash
curl -X POST "{{baseUrl}}/api/v1/Gateway/complete-payment"   -H "x-api-key: YOUR_SECRET_KEY"   -H "Content-Type: application/json"   -d '{
    "paymentReference": "JOHN-DOE-0002",
    "paymentChannel": "transfer"
  }'
```

**Success Response (abridged)**
```json
{
  "isSuccess": true,
  "message": "Completed"
}
```

---

## 🔔 Webhooks (for Transfers)

When a transfer is funded, Credipay **POSTs** a JSON payload to your webhook URL. Make sure your endpoint:
- Responds quickly (e.g., `200 OK`) and is idempotent.
- Validates event signatures/keys as applicable.
- Reconciles transactions by `paymentReference` / `transactionReference` in your database.

**Example Payload (abridged)**
```json
{
  "Amount": 100.00,
  "TransactionType": "Purchase",
  "SourceAccountNumber": "1234567890",
  "SourceAccountName": "John Doe",
  "SourceBankName": "Example Bank",
  "TransactionReference": "TXN123456",
  "TransactionStatus": "Successful",
  "TransactionRecievedDate": "2023-10-16T10:30:00",
  "TransactionDate": "2023-10-15T15:45:00"
}
```

---

## ✅ Recommended Handling

- **Always** rely on **webhooks** for final settlement state.
- Use **`paymentReference`** as your idempotency/reconciliation key.
- In Test:
  - Create (initiate) ➜ Simulate ➜ Complete.
- In Live:
  - Create (initiate) ➜ Wait for real bank funding ➜ Complete.

---

## 🔎 Troubleshooting

- **401 Unauthorized**: Missing/incorrect `x-api-key`, or using a Test key on Live (or vice‑versa).
- **Duplicate reference**: Ensure each `paymentReference` is unique per attempt.
- **No webhook received**: Confirm your Webhook URL, network accessibility, and that your server returns `200 OK` quickly.

---

## Changelog
- v1.0 — Transfer-only quickstart (initiate, complete, simulate-fund-transfer)

