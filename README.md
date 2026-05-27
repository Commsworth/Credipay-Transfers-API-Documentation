# Gateway API - Bank Transfer Collection Flow

## Overview

This document describes the **bank transfer payment collection** flow via the Gateway API, including the complete-payment webhook payload and how to query transaction details.

**Base URL (Dev):** `https://dev-developers-huerd7b8gahphecv.uksouth-01.azurewebsites.net`  
**Auth Header:** `x-api-key: {your_api_key}`

---

## Transfer Flow (Step-by-Step)

### 1. Initiate Payment

```
POST /api/v1/Gateway/initiate-payment
```

**Request Body:**
```json
{
  "amount": 5000,
  "paymentReference": "unique-ref-123",
  "paymentChannel": "transfer",
  "email": "customer@email.com",
  "firstName": "John",
  "lastName": "Doe",
  "phone": "08012345678",
  "currencyCode": "NGN",
  "paymentRoute": "direct"
}
```

**Response (Success):**
```json
{
  "value": {
    "amount": 5000.0,
    "charges": 0.0,
    "paymentReference": "unique-ref-123",
    "transactionReference": "270520263AE8",
    "currencyCode": "NGN",
    "accountInfo": {
      "accountName": "Business Name",
      "accountNumber": "9000027534",
      "bankName": "9japay MFB"
    },
    "cardInfo": {
      "authorizationType": null
    },
    "ussdInfo": null
  },
  "isSuccess": true,
  "error": "",
  "message": "Payment intitated successfully",
  "responseCode": "00"
}
```

The response provides a **transient virtual account** for the customer to transfer into. Display these details to the payer.

---

### 2. Customer Makes Transfer

The customer transfers the exact `amount` to the virtual account returned above. In **test environment**, you can simulate this:

```
POST /api/v1/Gateway/simulate-fund-transfer
```

**Request Body:**
```json
{
  "recipientAccountNumber": "1234567890",
  "amount": 5000
}
```

> **Note:** This endpoint only works in non-production environments.

---

### 3. Webhook Fires Automatically

Once the transfer is received by the system (via 9jaPay notification), a `Collection.CollectionSuccess` webhook is **automatically sent** to your configured webhook URL. You do NOT need to call `complete-payment` to receive the webhook.

### 4. Complete Payment (Confirmation/Re-trigger)

You can call `complete-payment` to confirm the payment was received and optionally re-trigger the webhook. This also serves as a synchronous check.

```
POST /api/v1/Gateway/complete-payment
```

**Request Body (for transfer channel):**
```json
{
  "paymentReference": "unique-ref-123",
  "paymentChannel": "transfer"
}
```

> **Important:** For the `transfer` channel, only `paymentReference` and `paymentChannel` are required. The `cardHolderInfo`, `otp`, and `pin` fields are **not needed** for transfers — they are only used for card payments.

**Response (Success):**
```json
{
  "isSuccess": true,
  "message": "Payment successful"
}
```

**Response (Failure - Payment Not Found):**
```json
{
  "isSuccess": false,
  "message": "Cannot find payment"
}
```

This means the fund transfer has not been received yet. Retry after ensuring the transfer was made to the correct account.

---

## Webhook Payload: Collection Success

The webhook is sent **automatically** when the fund transfer is received by the system. It is also re-sent when `complete-payment` is called successfully.

**Event Type:** `Collection.CollectionSuccess`

**Full Payload Structure (with envelope):**
```json
{
  "id": "6c4b408c-332e-4f7a-a7d4-0fa1976370d1",
  "eventType": "Collection.CollectionSuccess",
  "eventTimestamp": "2026-05-20T14:34:44.0817036Z",
  "version": "1.0",
  "eventMessage": "Collection completed.",
  "data": {
    "amount": 5000.00,
    "charge": 50.00,
    "paymentReference": "unique-ref-123",
    "transactionReference": "unique-ref-123",
    "paymentChannel": 0,
    "currencyCode": "NGN",
    "paymentRoute": 0,
    "linkId": null,
    "transactionType": 1,
    "accountInfo": {
      "accountName": "Business Name",
      "accountNumber": "9000027534",
      "bankName": "9japay MFB",
      "bankCode": "090629"
    },
    "payerInfo": {
      "firstName": "John",
      "lastName": "Doe",
      "email": "customer@email.com",
      "phone": "08012345678",
      "parentId": null,
      "customerGroup": null
    },
    "successPayerInfo": {
      "sourceAccountNumber": "NA",
      "sourceAccountName": "NA",
      "sourceBankName": "NA"
    }
  }
}
```

> **Note on enum values:** `paymentChannel`, `paymentRoute`, and `transactionType` are serialized as **integers** in the webhook (not strings). See mapping table below.

### Enum Mappings

| Field | Value | Meaning |
|-------|-------|---------|
| `paymentChannel` | `0` | transfer |
| `paymentChannel` | `1` | card |
| `paymentChannel` | `2` | ussd |
| `paymentRoute` | `0` | direct |
| `paymentRoute` | `1` | pop_up |
| `paymentRoute` | `2` | link |
| `transactionType` | `0` | Debit |
| `transactionType` | `1` | Credit |

### Webhook Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| `id` | guid | Unique webhook event ID (use for idempotency) |
| `eventType` | string | Event type identifier |
| `eventTimestamp` | datetime | UTC timestamp of the event |
| `version` | string | Webhook payload version |
| `eventMessage` | string | Human-readable event description |
| `data.amount` | decimal | The transaction amount |
| `data.charge` | decimal | Fee charged on the transaction |
| `data.paymentReference` | string/null | Your unique reference (same as sent in initiate-payment) |
| `data.transactionReference` | string | System transaction reference (equals `paymentReference` for Gateway transfers) |
| `data.paymentChannel` | int | `0`=transfer, `1`=card, `2`=ussd |
| `data.currencyCode` | string | Currency code (e.g., `NGN`) |
| `data.paymentRoute` | int | `0`=direct, `1`=pop_up, `2`=link |
| `data.linkId` | guid/null | Payment link ID (if initiated via payment link) |
| `data.transactionType` | int | `1`=Credit (always Credit for collections) |
| `data.accountInfo` | object | The virtual account that received the transfer |
| `data.accountInfo.bankCode` | string | Bank code (e.g., `090629`) |
| `data.payerInfo` | object | Payer details provided during initiation |
| `data.payerInfo.parentId` | guid/null | Parent customer ID (ERP integrations) |
| `data.payerInfo.customerGroup` | int/null | Customer group enum (ERP integrations) |
| `data.successPayerInfo` | object | Source bank details of the actual sender (may be "NA" in test) |

**Webhook Security:** The webhook includes an `X-Hash` header containing an HMAC-SHA256 signature of the payload body, signed with your secret key. Use this to verify webhook authenticity.

---

## Querying Transaction Details

### Get Collection by Transaction Reference

Use the `get-collection-transactions` endpoint with the `Search` query parameter to find a collection by its transaction reference (which equals your `paymentReference`):

```
GET /api/v1/Gateway/get-collection-transactions?Search={paymentReference}
```

**Response:**
```json
{
  "isSuccess": true,
  "message": "Transaction fetched succesfully.",
  "data": {
    "walletInfo": { ... },
    "collections": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "businessName": "Business Name",
        "sourceAccountNumber": "NA",
        "sourceAccountName": "NA",
        "sourceBankName": "NA",
        "amount": 5000.00,
        "charge": 50.00,
        "channel": "transfer",
        "transactionReference": "unique-ref-123",
        "paymentLinkId": "00000000-0000-0000-0000-000000000000",
        "transactionDate": "2024-01-01T00:00:00Z",
        "status": "Successful",
        "currencyCode": "NGN"
      }
    ],
    "pageInfo": {
      "curentPage": 1,
      "lastPage": 1,
      "dataCount": 1
    }
  }
}
```

> **Key:** The `id` field in the collection object is the **Collection GUID** — use it with `get-collection-id` for detailed info.

### Get Collection Detail by ID

```
GET /api/v1/Gateway/get-collection-id?Id={collectionGuid}
```

**Response:**
```json
{
  "isSuccess": true,
  "data": {
    "transaction": {
      "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "sourceAccountNumber": "NA",
      "sourceAccountName": "NA",
      "sourceBankName": "NA",
      "channel": "transfer",
      "amount": 5000.00,
      "paymentRoute": "direct",
      "payableAmount": 4950.00,
      "charge": 50.00,
      "currencyCode": "NGN",
      "trasactionReference": "unique-ref-123",
      "transactionDate": "2024-01-01T00:00:00Z",
      "paymentLinkId": null,
      "linkName": null
    },
    "collectionSplits": []
  }
}
```

---

## How to Get the Transaction/Collection ID from the Webhook

The webhook payload does **not** include the collection database `Id` (GUID) directly. To get it:

1. Receive the webhook → extract `paymentReference` (or `transactionReference` — they are the same for transfers)
2. Call `GET /api/v1/Gateway/get-collection-transactions?Search={paymentReference}`
3. From the response, get the `id` field of the matching collection object
4. Use that `id` with `GET /api/v1/Gateway/get-collection-id?Id={id}` for full details including split info

---

## Verify Payment Status (Transfer)

You can poll payment status instead of (or in addition to) waiting for the webhook:

```
POST /api/v1/Gateway/verify-payment-status
```

**Request Body:**
```json
{
  "reference": "unique-ref-123",
  "paymentChannel": "transfer"
}
```

**Response:**
```json
{
  "value": {
    "reference": "unique-ref-123",
    "status": "Successful"
  },
  "isSuccess": true,
  "error": "",
  "message": "Payment completed successfully.",
  "responseCode": "00"
}
```

**Possible status values:** `InProgress`, `Successful`, `Unknown`

---

## Known Limitations

| Issue | Status | Notes |
|-------|--------|-------|
| `successPayerInfo` shows "NA" in test | By design | Real sender bank details only available in live with supported bank providers |

---

## Complete Test Flow Example

```bash
# 1. Initiate payment
curl -X POST "{baseUrl}/api/v1/Gateway/initiate-payment" \
  -H "x-api-key: {your_api_key}" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 5000,
    "paymentReference": "test-ref-001",
    "paymentChannel": "transfer",
    "email": "test@email.com",
    "firstName": "Test",
    "lastName": "User",
    "phone": "08012345678",
    "currencyCode": "NGN",
    "paymentRoute": "direct"
  }'

# 2. Simulate fund transfer (test only)
# Use the accountNumber from step 1 response: value.accountInfo.accountNumber
curl -X POST "{baseUrl}/api/v1/Gateway/simulate-fund-transfer" \
  -H "x-api-key: {your_api_key}" \
  -H "Content-Type: application/json" \
  -d '{
    "recipientAccountNumber": "{accountNumber_from_step_1}",
    "amount": 5000
  }'

# 3. Verify payment status (poll until Successful)
curl -X POST "{baseUrl}/api/v1/Gateway/verify-payment-status" \
  -H "x-api-key: {your_api_key}" \
  -H "Content-Type: application/json" \
  -d '{
    "reference": "test-ref-001",
    "paymentChannel": "transfer"
  }'

# 4. Complete payment (confirmation/re-trigger webhook)
curl -X POST "{baseUrl}/api/v1/Gateway/complete-payment" \
  -H "x-api-key: {your_api_key}" \
  -H "Content-Type: application/json" \
  -d '{
    "paymentReference": "test-ref-001",
    "paymentChannel": "transfer"
  }'

# 5. Query collection by reference
curl -X GET "{baseUrl}/api/v1/Gateway/get-collection-transactions?Search=test-ref-001" \
  -H "x-api-key: {your_api_key}"
```

---

## Bug Fix Applied

**Issue:** `complete-payment` was returning 400 (Bad Request) when called with only `paymentReference` and `paymentChannel` for the transfer channel — it was requiring `cardHolderInfo` even though transfer payments don't use it.

**Root Cause:** The `[Required]` validation attribute on `CardHolderInfo` in `CompletePaymentRequest` caused ASP.NET model validation to reject the request before it reached the controller.

**Fix:** Removed `[Required]` from `CardHolderInfo` — it is now optional. Card channel consumers should still provide it; transfer channel consumers can omit it entirely.
