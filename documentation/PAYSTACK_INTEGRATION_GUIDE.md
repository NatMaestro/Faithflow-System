# Paystack Subscription Payment Integration Guide

## Overview

This document explains the complete Paystack integration for subscription payments in FaithFlows.

## Architecture

### Payment Flow

1. **User selects plan** on landing page → Plan stored in sessionStorage
2. **User completes onboarding** → Church account created
3. **Payment page** → User enters email/name → Backend initializes Paystack payment
4. **Paystack redirect** → User completes payment on Paystack
5. **Callback page** → Backend verifies payment → Subscription activated
6. **Webhook** (optional) → Paystack sends webhook for reliability

## Backend Setup

### 1. Environment Variables

Add to your `.env` file:

```env
PAYSTACK_SECRET_KEY=sk_test_xxxxxxxxxxxxx  # Your Paystack secret key
PAYSTACK_PUBLIC_KEY=pk_test_xxxxxxxxxxxxx  # Your Paystack public key (optional, for frontend)
```

### 2. Database Migration

Run migrations to create the `SubscriptionPayment` model:

```bash
cd faithflow-backend
python manage.py makemigrations churches
python manage.py migrate
```

### 3. Paystack Webhook Configuration

**IMPORTANT: Webhook URL Format**

Webhooks are server-to-server, so Paystack sends them to your **main domain** (not subdomain).

✅ **Use this URL (Production):**

```
https://api.faithflow360.com/api/v1/churches/subscription-payment/webhook/
```

❌ **Don't use:**

- `https://apostolic.api.faithflow360.com/...` (subdomain won't work - Paystack doesn't know which subdomain)
- `http://localhost:8000/...` (Paystack can't reach localhost from their servers)
- Any subdomain URL

**Why no subdomain?**

- Paystack sends webhooks from their servers to your main domain
- They don't know which church's subdomain to use
- The webhook endpoint is in the public schema and works without tenant context

**Steps:**

1. Go to Paystack Dashboard → Settings → Webhooks
2. Add webhook URL: `https://api.faithflow360.com/api/v1/churches/subscription-payment/webhook/`
3. Select events: `charge.success`, `charge.failed`
4. Copy webhook secret (optional, for signature verification)

**Testing Locally:**

- For local testing, use a tool like **ngrok** to expose your localhost:
  ```bash
  ngrok http 8000
  # Use: https://your-ngrok-url.ngrok.io/api/v1/churches/subscription-payment/webhook/
  ```
- Or test webhooks only in production/staging environment

## API Endpoints

### Initialize Payment

```
POST /api/v1/churches/subscription-payment/initialize/
Body: {
    "church_id": 1,
    "plan": "basic|standard|premium|enterprise",
    "duration": "monthly|yearly",
    "email": "user@example.com",
    "name": "User Name"
}
Response: {
    "success": true,
    "authorization_url": "https://paystack.com/...",
    "reference": "SUB_1_ABC123",
    "payment_id": 1
}
```

### Verify Payment

```
GET /api/v1/churches/subscription-payment/verify/?reference=SUB_1_ABC123
Response: {
    "success": true,
    "message": "Payment verified successfully",
    "payment": {
        "id": 1,
        "plan": "basic",
        "amount": "19.00",
        "status": "completed",
        "subscription_activated": true
    }
}
```

### Webhook Endpoint

```
POST /api/v1/churches/subscription-payment/webhook/
Headers: {
    "X-Paystack-Signature": "..." (optional verification)
}
Body: Paystack webhook payload
```

## Frontend Routes

- `/subscription/payment` - Payment page (collects email/name, initializes payment)
- `/subscription/payment/callback` - Payment callback (verifies payment, activates subscription)

## Plan Pricing

| Plan       | Monthly | Yearly (10% discount) |
| ---------- | ------- | --------------------- |
| Basic      | $19     | $205                  |
| Standard   | $29     | $313                  |
| Premium    | $79     | $853                  |
| Enterprise | $199    | $2,149                |

## Security Best Practices

✅ **Implemented:**

- Secret key never exposed to frontend
- Payment verification on backend
- Webhook support for reliability
- Payment records stored in database
- Multi-tenant aware (church-specific payments)

⚠️ **Recommended (not yet implemented):**

- Webhook signature verification
- Rate limiting on payment endpoints
- Payment retry logic
- Refund handling
- Payment receipt emails

## Testing

### Test Cards (Paystack Test Mode)

- **Success**: `4084084084084081`
- **Decline**: `5060666666666666666`
- **Insufficient Funds**: `5060666666666666667`

### Test Flow

1. Select a paid plan on landing page
2. Complete onboarding
3. Enter test card details on Paystack
4. Verify subscription is activated after payment

## Troubleshooting

### Payment not initializing

- Check `PAYSTACK_SECRET_KEY` is set
- Check backend logs for errors
- Verify church_id is valid

### Payment not verifying

- Check Paystack dashboard for transaction status
- Verify webhook is configured
- Check backend logs for verification errors

### Subscription not activating

- Check payment status in database
- Verify `activate_subscription()` is called
- Check church model is updated correctly

## Database Schema

### SubscriptionPayment Model

- `church` - ForeignKey to Church
- `amount` - Decimal (plan price)
- `currency` - CharField (default: GHS)
- `plan` - CharField (basic/standard/premium/enterprise)
- `duration` - CharField (monthly/yearly)
- `reference` - CharField (unique, for tracking)
- `paystack_reference` - CharField (Paystack transaction reference)
- `status` - CharField (pending/processing/completed/failed)
- `subscription_activated` - Boolean
- `subscription_start_date` - DateTime
- `subscription_end_date` - DateTime
- `user_email` - EmailField
- `user_name` - CharField
- `paystack_response` - JSONField (for audit)

## Next Steps

1. **Add webhook signature verification** for security
2. **Implement payment receipts** (PDF/email)
3. **Add refund handling** for cancellations
4. **Create payment history page** for admins
5. **Add recurring payment support** for subscriptions
6. **Implement payment retry logic** for failed payments
