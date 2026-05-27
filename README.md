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
  "phoneNumber": "08012345678",
  "currencyType": "NGN"
}
```

**Response (Success):**
```json
{
  "isSuccess": true,
  "message": "Transaction initiated",
  "data": {
    "accountNumber": "1234567890",
    "accountName": "Business Name",
    "bankName": "9jaPay MFB",
    "bankCode": "090629",
    "paymentReference": "unique-ref-123",
    "expiresAt": "2024-01-01T00:30:00Z"
  }
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

### 3. Complete Payment (Trigger Webhook)

After the fund transfer is received (simulated or real), call `complete-payment` to finalize and trigger the collection success webhook to your webhook URL.

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

When `complete-payment` succeeds, a webhook is sent to your configured webhook URL with the following payload:

**Event Type:** `Collection.CollectionSuccess`

**Payload Structure:**
```json
{
  "event": "Collection.CollectionSuccess",
  "data": {
    "amount": 5000.00,
    "charge": 50.00,
    "paymentReference": "unique-ref-123",
    "transactionReference": "unique-ref-123",
    "paymentChannel": "transfer",
    "currencyCode": "NGN",
    "paymentRoute": "direct",
    "linkId": null,
    "transactionType": "Credit",
    "accountInfo": {
      "accountName": "Business Name",
      "accountNumber": "1234567890",
      "bankName": "9jaPay MFB"
    },
    "payerInfo": {
      "firstName": "John",
      "lastName": "Doe",
      "email": "customer@email.com",
      "phone": "08012345678"
    },
    "successPayerInfo": {
      "sourceAccountNumber": "NA",
      "sourceAccountName": "NA",
      "sourceBankName": "NA"
    }
  }
}
```

### Webhook Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| `amount` | decimal | The transaction amount |
| `charge` | decimal | Fee charged on the transaction |
| `paymentReference` | string | Your unique reference (same as sent in initiate-payment) |
| `transactionReference` | string | System transaction reference (equals `paymentReference` for transfers) |
| `paymentChannel` | string | `transfer`, `card`, `ussd`, `qrcode`, `account` |
| `currencyCode` | string | Currency code (e.g., `NGN`) |
| `paymentRoute` | string | `direct`, `pop_up`, `link` |
| `linkId` | guid/null | Payment link ID (if initiated via payment link) |
| `transactionType` | string | Always `Credit` for collections |
| `accountInfo` | object | The virtual account that received the transfer |
| `payerInfo` | object | Payer details provided during initiation |
| `successPayerInfo` | object | Source bank details of the actual sender (may be "NA" in test) |

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

## Known Limitations

| Issue | Status | Notes |
|-------|--------|-------|
| `verify-payment-status` not implemented for transfers | Known | Only works for card payments currently. Use `get-collection-transactions?Search=` instead. |
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
    "phoneNumber": "08012345678",
    "currencyType": "NGN"
  }'

# 2. Simulate fund transfer (test only)
curl -X POST "{baseUrl}/api/v1/Gateway/simulate-fund-transfer" \
  -H "x-api-key: {your_api_key}" \
  -H "Content-Type: application/json" \
  -d '{
    "recipientAccountNumber": "{accountNumber_from_step_1}",
    "amount": 5000
  }'

# 3. Complete payment (triggers webhook)
curl -X POST "{baseUrl}/api/v1/Gateway/complete-payment" \
  -H "x-api-key: {your_api_key}" \
  -H "Content-Type: application/json" \
  -d '{
    "paymentReference": "test-ref-001",
    "paymentChannel": "transfer"
  }'

# 4. Query collection by reference
curl -X GET "{baseUrl}/api/v1/Gateway/get-collection-transactions?Search=test-ref-001" \
  -H "x-api-key: {your_api_key}"
```

---

## Bug Fix Applied

**Issue:** `complete-payment` was returning 400 (Bad Request) when called with only `paymentReference` and `paymentChannel` for the transfer channel — it was requiring `cardHolderInfo` even though transfer payments don't use it.

**Root Cause:** The `[Required]` validation attribute on `CardHolderInfo` in `CompletePaymentRequest` caused ASP.NET model validation to reject the request before it reached the controller.

**Fix:** Removed `[Required]` from `CardHolderInfo` — it is now optional. Card channel consumers should still provide it; transfer channel consumers can omit it entirely.
