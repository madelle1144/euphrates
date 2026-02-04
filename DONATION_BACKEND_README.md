# Euphrates Community Centre - Donation System Backend Documentation

## Overview

The donation system backend is built with Next.js 16 and provides a complete payment processing solution supporting multiple payment methods:

- **M-Pesa** - Mobile money payments
- **Stripe** - International credit/debit card payments
- **PayBill** - Kenyan bank bill payments
- **Bank Transfer** - Direct bank deposits with manual verification

## Architecture

### Directory Structure

```
src/
├── lib/
│   ├── types/
│   │   └── donation.ts          # TypeScript types and interfaces
│   └── services/
│       └── paymentService.ts    # Payment provider integrations
├── app/
│   ├── api/
│   │   ├── donations/
│   │   │   ├── route.ts         # POST/GET donation endpoints
│   │   │   └── [id]/route.ts    # Get specific donation
│   │   └── webhooks/
│   │       ├── stripe/route.ts  # Stripe webhook handler
│   │       ├── mpesa/route.ts   # M-Pesa webhook handler
│   │       └── paybill/route.ts # PayBill webhook handler
│   ├── donate/page.tsx          # Donation page UI
│   └── donations-dashboard/page.tsx # Admin dashboard
└── components/
    └── DonationForm.tsx         # Reusable donation form
```

## API Endpoints

### 1. Create Donation
**POST** `/api/donations`

Creates a new donation and initiates payment processing.

**Request Body:**
```json
{
  "donorName": "John Doe",
  "donorEmail": "john@example.com",
  "donorPhone": "+254723239761",
  "amount": 1000,
  "paymentMethod": "mpesa",
  "message": "Keep up the good work"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Donation created",
  "donation": {
    "id": "uuid-string",
    "donorName": "John Doe",
    "amount": 1000,
    "paymentMethod": "mpesa",
    "status": "pending",
    "createdAt": "2026-02-01T10:00:00Z"
  },
  "payment": {
    "transactionId": "uuid-string",
    "paymentReference": "reference-code"
  }
}
```

### 2. Get Donation
**GET** `/api/donations/[id]`

Retrieve a specific donation by ID.

**Response:**
```json
{
  "success": true,
  "donation": {
    "id": "uuid-string",
    "donorName": "John Doe",
    "amount": 1000,
    "status": "completed",
    "transactionId": "stripe-payment-id",
    "processedAt": "2026-02-01T10:05:00Z"
  }
}
```

### 3. List Donations
**GET** `/api/donations?status=completed&paymentMethod=mpesa&limit=50`

List donations with optional filtering and statistics.

**Query Parameters:**
- `status` - Filter by status (pending, completed, failed, cancelled)
- `paymentMethod` - Filter by payment method (stripe, mpesa, paybill, bank_transfer)
- `limit` - Number of results (default: 50)

**Response:**
```json
{
  "success": true,
  "donations": [...],
  "stats": {
    "totalDonations": 45,
    "totalAmount": 125000,
    "byPaymentMethod": {
      "mpesa": 25,
      "stripe": 15,
      "paybill": 3,
      "bank_transfer": 2
    },
    "byStatus": {
      "completed": 40,
      "pending": 3,
      "failed": 2,
      "cancelled": 0
    }
  },
  "count": 45
}
```

## Webhook Endpoints

### Stripe Webhook
**POST** `/api/webhooks/stripe`

Handles payment intent events from Stripe.

**Events Processed:**
- `payment_intent.succeeded` - Updates donation status to COMPLETED
- `payment_intent.payment_failed` - Updates donation status to FAILED
- `charge.refunded` - Updates donation status to CANCELLED

### M-Pesa Webhook
**POST** `/api/webhooks/mpesa`

Handles M-Pesa STK push callback.

**Expected Payload:**
```json
{
  "Body": {
    "stkCallback": {
      "MerchantRequestID": "...",
      "CheckoutRequestID": "...",
      "ResultCode": 0,
      "ResultDesc": "The service request has been processed successfully.",
      "CallbackMetadata": {
        "Item": [
          { "Name": "Amount", "Value": 1000 },
          { "Name": "MpesaReceiptNumber", "Value": "LK451A1A912" }
        ]
      }
    }
  }
}
```

### PayBill Webhook
**POST** `/api/webhooks/paybill`

Handles PayBill payment confirmation.

**Expected Payload:**
```json
{
  "reference": "ECC-12345678",
  "status": "success",
  "transactionId": "PB12345678",
  "amount": 1000
}
```

## Environment Variables

Create or update `.env.local`:

```bash
# Site Configuration
NEXT_PUBLIC_SITE_URL=http://localhost:3000

# Stripe
STRIPE_SECRET_KEY=sk_test_your_key
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_your_key
STRIPE_WEBHOOK_SECRET=whsec_your_secret

# M-Pesa
MPESA_CONSUMER_KEY=your_consumer_key
MPESA_CONSUMER_SECRET=your_consumer_secret
MPESA_SHORTCODE=your_shortcode
MPESA_PASSKEY=your_passkey
MPESA_CALLBACK_URL=http://localhost:3000/api/webhooks/mpesa
MPESA_ENVIRONMENT=sandbox

# PayBill
PAYBILL_API_KEY=your_api_key
PAYBILL_BUSINESS_NUMBER=your_business_number
PAYBILL_CALLBACK_URL=http://localhost:3000/api/webhooks/paybill

# Bank Details
BANK_ACCOUNT_NAME=Euphrates Community Centre
BANK_ACCOUNT_NUMBER=01101729876002
BANK_NAME=Cooperative Bank
BANK_BRANCH=Dandora

# Webhook Security
WEBHOOK_SIGNING_SECRET=your_signing_secret
```

## Payment Methods Implementation Guide

### M-Pesa Integration

1. **Get Access Token:**
   ```typescript
   const auth = Buffer.from(`${CONSUMER_KEY}:${CONSUMER_SECRET}`).toString("base64");
   const token = await fetch("https://sandbox.safaricom.co.ke/oauth/v1/generate", {
     headers: { Authorization: `Basic ${auth}` }
   });
   ```

2. **Initiate STK Push:**
   ```typescript
   const timestamp = getTimestamp();
   const password = Buffer.from(SHORTCODE + PASSKEY + timestamp).toString("base64");
   
   await fetch("https://sandbox.safaricom.co.ke/mpesa/stkpush/v1/processrequest", {
     method: "POST",
     headers: {
       Authorization: `Bearer ${accessToken}`,
       "Content-Type": "application/json"
     },
     body: JSON.stringify({
       BusinessShortCode: SHORTCODE,
       Password: password,
       Timestamp: timestamp,
       TransactionType: "CustomerPayBillOnline",
       Amount: donation.amount,
       PartyA: phoneNumber,
       PartyB: SHORTCODE,
       PhoneNumber: phoneNumber,
       CallBackURL: CALLBACK_URL,
       AccountReference: donation.id,
       TransactionDesc: "Euphrates Donation"
    })
   });
   ```

### Stripe Integration

1. **Create Payment Intent:**
   ```typescript
   const stripe = require("stripe")(STRIPE_SECRET_KEY);
   const paymentIntent = await stripe.paymentIntents.create({
     amount: donation.amount * 100, // in cents
     currency: "kes",
     metadata: { donationId: donation.id }
   });
   ```

2. **Handle Webhook:**
   Stripe automatically sends webhook events to your configured endpoint.

### PayBill Integration

1. **Generate Reference:**
   Generate unique reference combining business number and donation ID

2. **Listen for Confirmation:**
   PayBill sends confirmation to the configured callback URL

### Bank Transfer

1. **Generate Reference:**
   Create unique reference: `ECC-{DONATION_ID_SUBSTRING}`

2. **Manual Verification:**
   Admin reviews bank statements and updates donation status manually

## Frontend Integration

### Using the Donation Form Component

```tsx
import { DonationForm } from "@/components/DonationForm";

export default function Page() {
  return (
    <DonationForm
      onSuccess={(donation) => console.log("Donation created:", donation)}
      onError={(error) => console.log("Error:", error)}
    />
  );
}
```

### Manual API Integration

```typescript
const response = await fetch("/api/donations", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    donorName: "Jane Doe",
    donorEmail: "jane@example.com",
    amount: 5000,
    paymentMethod: "stripe"
  })
});

const data = await response.json();

// Handle payment based on method
if (data.payment.paymentUrl) {
  window.location.href = data.payment.paymentUrl; // Redirect to Stripe
}
```

## Data Types

```typescript
enum DonationStatus {
  PENDING = "pending",      // Payment not yet confirmed
  COMPLETED = "completed",  // Payment received and verified
  FAILED = "failed",        // Payment failed
  CANCELLED = "cancelled"   // Payment refunded or cancelled
}

enum PaymentMethod {
  STRIPE = "stripe",
  MPESA = "mpesa",
  PAYBILL = "paybill",
  BANK_TRANSFER = "bank_transfer"
}

interface Donation {
  id: string;
  donorName: string;
  donorEmail: string;
  donorPhone?: string;
  amount: number;
  currency: string;
  paymentMethod: PaymentMethod;
  status: DonationStatus;
  message?: string;
  transactionId?: string;
  paymentReference?: string;
  createdAt: Date;
  updatedAt: Date;
  processedAt?: Date;
}
```

## Admin Dashboard

Access the donations dashboard at `/donations-dashboard`

**Features:**
- View all donations with filtering by status and payment method
- Real-time statistics (total donations, total amount, breakdown by method)
- Payment method breakdown visualization
- Donation status tracking

## Security Considerations

1. **Webhook Signature Verification:**
   - Verify all webhook signatures from payment providers
   - Only process authenticated webhooks

2. **API Rate Limiting:**
   - Implement rate limiting on donation endpoints
   - Prevent abuse and DDoS attacks

3. **Payment Reference Validation:**
   - Always validate payment references with payment providers
   - Prevent duplicate processing

4. **Data Encryption:**
   - Encrypt sensitive donation data at rest
   - Use HTTPS for all communications

5. **Authentication:**
   - Protect admin dashboard with authentication
   - Use API keys for admin endpoints

## Testing

### Local Testing with Stripe

Use Stripe test cards:
- Success: 4242 4242 4242 4242
- Decline: 4000 0000 0000 0002
- Auth Required: 4000 0027 6000 3184

### Local Testing with M-Pesa

Use sandbox environment credentials configured in `.env.local`

## Deployment

### Environment Setup

1. Update production credentials in environment variables
2. Configure webhook URLs to production domain
3. Enable webhook signature verification
4. Set up database for production data persistence
5. Configure SSL/TLS certificates

### Monitoring

- Log all transactions and webhook events
- Monitor payment provider APIs for status
- Set up alerts for failed transactions
- Track donation conversion rates

## Future Enhancements

- [ ] Database integration (Prisma/PostgreSQL)
- [ ] Email notifications for donors
- [ ] Donation receipt generation
- [ ] Tax documentation
- [ ] Recurring/monthly donations
- [ ] Donation campaign tracking
- [ ] Multi-currency support
- [ ] PCI compliance for direct card processing
- [ ] Admin authentication and role-based access
- [ ] Advanced reporting and analytics

## Support

For issues or questions:
- Email: euphratescommunity@gmail.com
- Phone: +254 723 239 761
