# 2026 Bridges Gala - Payment System Documentation

## Overview

The Gala payment system consists of multiple integrated applications handling different stages of the sponsorship and payment journey.

---

## System Architecture

### Component Overview

| Component | Repository | Purpose | URL |
|-----------|------------|---------|-----|
| Gala Payment Page | `2026GalaFrontEnd` | Immediate credit card payments | https://gala.bridgestowork.org |
| Pledge Form | `2026GalaPledgeForm` | Deferred payment pledges | https://pledge.bridgestowork.org |
| Main Backend | `bridges-donation-backend` | Payment processing & webhooks | Azure App Service |
| Pledge Backend | `gala-pledge-backend` | Pledge form handling & PDF | Azure App Service |

### Backend URLs

| Environment | URL |
|-------------|-----|
| Main Backend (Production) | https://bridges-donation-backend-b2cfejegfjgcc6f6.eastus2-01.azurewebsites.net |
| Pledge Backend (Production) | https://gala-pledge-backend-h0arg5f8g8e2hwf6.eastus2-01.azurewebsites.net |
| Main Backend (Local) | http://localhost:4242 |
| Pledge Backend (Local) | http://localhost:8080 |

---

## Payment Workflows

### Workflow A: Immediate Payment (Gala Payment Page)

```
User visits gala.bridgestowork.org
         ↓
Selects sponsor tier + fills contact info
         ↓
Clicks "Continue to Payment"
         ↓
Frontend calls POST /create-gala-checkout
         ↓
Redirected to Stripe Checkout
         ↓
Completes payment on Stripe
         ↓
Returns to thank-you.html
         ↓
Webhook: checkout.session.completed
         ↓
Confirmation email sent to donor
Internal notification sent to Gala team
```

### Workflow B: Pledge Submission (Pledge Form)

```
User visits pledge.bridgestowork.org
         ↓
Selects tier + fills sponsor info + selects payment method
         ↓
Clicks "Submit Pledge"
         ↓
Frontend calls POST /submit-pledge
         ↓
Backend generates PDF + sends emails
         ↓
Redirects to thank-you.html
         ↓
Sponsor pays later via selected method
```

### Key Difference

| Feature | Gala Payment Page | Pledge Form |
|---------|-------------------|-------------|
| Payment Timing | Immediate | Deferred |
| Payment Method | Credit card only | Check, wire, invoice, or credit card |
| Stripe Checkout | Yes | No (pledge only) |
| PDF Generated | No | Yes |

---

## Stripe Integration

### API Keys

| Environment | Key Type | Location |
|-------------|----------|----------|
| Test | Publishable | `2026GalaFrontEnd/index.html` |
| Test | Secret | Backend `.env` file |
| Live | Publishable | `2026GalaFrontEnd/index.html` |
| Live | Secret | Azure App Service configuration |

### Stripe Configuration

| Setting | Value |
|---------|-------|
| API Version | 2024-06-20 |
| Mode | Payment (one-time) |
| Currency | USD |
| Payment Methods | Card |

### Fee Calculation

Stripe processing fees are calculated as follows:

```
Fee Rate: 2.9% + $0.30 per transaction

Formula (when donor covers fees):
Gross = (Subtotal + $0.30) / (1 - 0.029)
Fees = Gross - Subtotal
```

| Tier | Base Amount | With Fees (if covered) |
|------|-------------|------------------------|
| Champion | $75,000.00 | $77,524.02 |
| Premier | $50,000.00 | $51,776.60 |
| Leadership Circle | $35,000.00 | $36,278.04 |
| Partner | $17,500.00 | $18,280.72 |
| Benefactor | $12,500.00 | $13,031.96 |
| Individual Ticket | $1,500.00 | $1,574.02 |

---

## API Endpoints

### Create Gala Checkout Session

Creates a Stripe checkout session for immediate payment.

```
POST /create-gala-checkout
Host: bridges-donation-backend-*.azurewebsites.net
Content-Type: application/json
```

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amountCents` | number | Yes | Total amount in cents |
| `tierLabel` | string | Yes | Sponsorship tier name |
| `donorEmail` | string | Yes | Donor's email address |
| `donorName` | string | Yes | Donor's full name |
| `donorPhone` | string | Yes | Phone number |
| `donorTitle` | string | No | Job title |
| `donorCompany` | string | No | Company name |
| `successUrl` | string | Yes | Redirect URL on success |
| `cancelUrl` | string | Yes | Redirect URL on cancel |
| `coverFees` | boolean | No | Whether donor covers Stripe fees |

**Response:**

```json
{
  "url": "https://checkout.stripe.com/c/pay/...",
  "sessionId": "cs_live_..."
}
```

### Submit Gala Pledge

Submits a pledge for deferred payment.

```
POST /submit-pledge
Host: gala-pledge-backend-*.azurewebsites.net
Content-Type: application/json
```

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tierLabel` | string | Yes | Sponsorship tier |
| `amount` | number | Yes | Pledge amount in dollars |
| `quantity` | number | No | Ticket quantity (Individual Ticket only) |
| `companyOrg` | string | Yes | Company/organization name |
| `sponsorshipName` | string | Yes | Name for event materials |
| `primaryContact` | string | Yes | Contact person name |
| `title` | string | No | Contact's job title |
| `streetAddress` | string | Yes | Mailing address |
| `city` | string | Yes | City |
| `state` | string | Yes | State abbreviation |
| `zip` | string | Yes | ZIP code |
| `email` | string | Yes | Email address |
| `phone` | string | Yes | Phone number |
| `paymentContactName` | string | No | Alternate payment contact |
| `paymentContactEmail` | string | No | Alternate payment email |
| `paymentOption` | string | Yes | Payment method selection |
| `pledgeDate` | string | Yes | Date of pledge |

**Response:**

```json
{
  "success": true,
  "message": "Pledge submitted successfully"
}
```

### Stripe Webhook

Handles Stripe events for payment processing.

```
POST /webhook
Host: bridges-donation-backend-*.azurewebsites.net
Headers: stripe-signature (required)
```

**Events Handled:**

| Event | Action |
|-------|--------|
| `checkout.session.completed` | Send confirmation emails, log payment |
| `payment_intent.succeeded` | Update payment status |
| `charge.refunded` | Handle refund processing |

---

## Email Notifications

### Immediate Payment Emails

| Email Type | Recipient | Trigger |
|------------|-----------|---------|
| Donor Confirmation | Donor's email | Successful Stripe payment |
| Internal Notification | Gala team | Successful Stripe payment |

**Internal Recipients for Gala Payments:**
- bridgesgala@orrgroup.com
- Addresses in `GALA_INTERNAL_EMAILS` environment variable

### Pledge Form Emails

| Email Type | Recipient | Trigger | Attachment |
|------------|-----------|---------|------------|
| Sponsor Confirmation | Sponsor's email | Pledge submission | PDF |
| Internal Notification | Gala team | Pledge submission | PDF |

**Internal Recipients for Pledges:**
- Ann.Hearn@bridgestowork.org
- Kate.Brown@bridgestowork.org
- Justin.Down@bridgestowork.org
- bridgesgala@orrgroup.com

### Email Content

**Donor Confirmation (Immediate Payment):**
- Thank you message
- Tier and amount
- Payment confirmation
- Event details

**Sponsor Confirmation (Pledge):**
- Thank you message
- Pledge details
- Payment instructions based on selected method
- PDF attachment with pledge summary

**Internal Notification:**
- Sponsor/donor details
- Tier and amount
- Contact information
- Payment method selected

---

## Sponsorship Tiers

### Tier Amounts

| Tier | Amount | Amount (cents) | Quantity Support |
|------|--------|----------------|------------------|
| Champion | $75,000 | 7500000 | No |
| Premier | $50,000 | 5000000 | No |
| Leadership Circle | $35,000 | 3500000 | No |
| Partner | $17,500 | 1750000 | No |
| Benefactor | $12,500 | 1250000 | No |
| Individual Ticket | $1,500 | 150000 | Yes (1-50) |
| Other Amount | Custom | >= 100 | N/A |
| Donation | Custom | >= 100 | N/A |

### Backend Validation

The backend validates that submitted amounts match expected tier values:

```javascript
const GALA_TIER_AMOUNTS = {
  'Champion': 7500000,
  'Premier': 5000000,
  'Leadership Circle': 3500000,
  'Partner': 1750000,
  'Benefactor': 1250000,
  'Individual Ticket': 150000,
  'Other Amount': 0,
};
```

**Validation Rules:**
- Fixed tiers must match exact amount (with or without fees)
- Individual Ticket: amount must be multiple of $1,500 (quantity 1-50)
- Other Amount/Donation: minimum $1.00

---

## Environment Variables

### Main Backend (bridges-donation-backend)

| Variable | Required | Description |
|----------|----------|-------------|
| `STRIPE_SECRET_KEY` | Yes | Stripe secret API key |
| `STRIPE_WEBHOOK_SECRET` | Recommended | Webhook signature verification |
| `SENDGRID_API_KEY` | Yes | SendGrid API key for emails |
| `GALA_INTERNAL_EMAILS` | No | Additional internal notification recipients |
| `DONATION_PRODUCT_ID` | No | For monthly subscription products |
| `MANAGE_TOKEN_SECRET` | No | Subscription management tokens |
| `FRONTEND_BASE_URL` | No | Base URL for email links |

### Pledge Backend (gala-pledge-backend)

| Variable | Required | Description |
|----------|----------|-------------|
| `SENDGRID_API_KEY` | Yes | SendGrid API key for emails |
| `PORT` | No | Server port (default: 8080) |

---

## CORS Configuration

### Main Backend Allowed Origins

| Origin | Purpose |
|--------|---------|
| https://donate.bridgestowork.org | Main donation page |
| https://gala.bridgestowork.org | Gala payment page |
| https://galapledge.bridgestowork.org | Pledge form (alternate) |
| https://www.bridgestowork.org | Main website |
| https://bridgestowork.org | Main website (no www) |
| http://localhost:4242 | Local development |
| http://localhost:5500 | Local development |

### Pledge Backend Allowed Origins

| Origin | Purpose |
|--------|---------|
| https://pledge.bridgestowork.org | Pledge form |
| http://localhost:5500 | Local development |
| http://localhost:3000 | Local development |
| http://127.0.0.1:5500 | Local development |

---

## Security

### Stripe Security

| Feature | Implementation |
|---------|----------------|
| Webhook Signature | Verified using `stripe.webhooks.constructEvent()` |
| Publishable Keys | Frontend only, safe to expose |
| Secret Keys | Backend only, stored in environment variables |
| HTTPS | Required for all production endpoints |

### Data Security

| Measure | Description |
|---------|-------------|
| HTML Escaping | All user input escaped in emails |
| CORS | Whitelisted domains only |
| No PCI Data | Card details handled by Stripe only |
| Token Verification | HMAC-SHA256 for subscription tokens |

---

## Troubleshooting

### Common Issues

| Issue | Possible Cause | Solution |
|-------|----------------|----------|
| Checkout fails | Invalid amount | Verify amount matches tier |
| Webhook not firing | Missing webhook secret | Configure `STRIPE_WEBHOOK_SECRET` |
| CORS error | Origin not whitelisted | Add origin to CORS configuration |
| Emails not sending | Invalid SendGrid key | Verify `SENDGRID_API_KEY` |
| Fee calculation wrong | Frontend/backend mismatch | Ensure both use same formula |

### Testing Payments

**Test Card Numbers (Stripe Test Mode):**

| Card Number | Result |
|-------------|--------|
| 4242 4242 4242 4242 | Successful payment |
| 4000 0000 0000 0002 | Card declined |
| 4000 0000 0000 9995 | Insufficient funds |

**Test Webhook Locally:**
```bash
stripe listen --forward-to localhost:4242/webhook
```

### Checking Logs

**Stripe Dashboard:**
1. Go to https://dashboard.stripe.com
2. Navigate to Developers > Logs
3. Filter by endpoint or event type

**Azure Logs:**
1. Azure Portal > App Services > Select backend
2. Monitoring > Log stream

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        GALA PAYMENT SYSTEM                              │
└─────────────────────────────────────────────────────────────────────────┘

IMMEDIATE PAYMENT:                      PLEDGE SUBMISSION:

┌──────────────────┐                    ┌──────────────────┐
│ gala.bridges...  │                    │ pledge.bridges...│
│ Payment Page     │                    │ Pledge Form      │
└────────┬─────────┘                    └────────┬─────────┘
         │                                       │
         │ POST /create-gala-checkout            │ POST /submit-pledge
         ↓                                       ↓
┌──────────────────────────┐            ┌──────────────────────────┐
│ bridges-donation-backend │            │ gala-pledge-backend      │
│ - Validate amount        │            │ - Generate PDF           │
│ - Create Stripe session  │            │ - Send confirmation      │
└────────┬─────────────────┘            │ - Send internal email    │
         │                              └──────────────────────────┘
         │ Redirect to Stripe                    │
         ↓                                       │
┌──────────────────┐                             │
│ Stripe Checkout  │                             │
│ Secure Payment   │                             │
└────────┬─────────┘                             │
         │                                       │
         │ Payment Complete                      │
         ↓                                       │
┌──────────────────────────┐                     │
│ Stripe Webhook           │                     │
│ checkout.session.completed                     │
└────────┬─────────────────┘                     │
         │                                       │
         │ POST /webhook                         │
         ↓                                       │
┌──────────────────────────┐                     │
│ bridges-donation-backend │                     │
│ - Send donor email       │                     │
│ - Send internal email    │                     │
└──────────────────────────┘                     │
         │                                       │
         ↓                                       ↓
┌──────────────────┐                    ┌──────────────────┐
│ thank-you.html   │                    │ thank-you.html   │
│ Confirmation     │                    │ Confirmation     │
└──────────────────┘                    └──────────────────┘
```

---

## Contact

| Role | Contact |
|------|---------|
| Gala Team | BridgesGala@orrgroup.com |
| Phone | (202) 719-8053 |
| Technical | Justin.Down@bridgestowork.org |
