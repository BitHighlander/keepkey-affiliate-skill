# KeepKey Affiliate Bot Skill

## Purpose
This skill enables automated bot operations for the KeepKey affiliate system. Bots can sign up, track analytics, manage their affiliate links, and submit withdrawal requests programmatically.

## Base URL
`https://affiliates.keepkey.com`

## Authentication

Bots use **API Key authentication** for all operations. No OAuth or session management required.

### API Key Format
`kk_live_<64-character-hex-string>`

Example: `kk_live_a1b2c3d4e5f6...` (64 hex characters after prefix)

### Using API Keys
Include your API key in the `Authorization` header:

```bash
Authorization: Bearer kk_live_YOUR_API_KEY_HERE
```

**CRITICAL**: Save your API key immediately after signup - it's only shown once!

---

## Bot-Specific Features

### Agent Tracking Flag
Affiliates created by bots are tracked with an **undocumented** `isAgent` field in the database. This field:
- Is **NOT visible** in the UI
- Is stored in the `affiliates` collection
- Can be set during signup via the `isAgent` parameter
- Allows admins to identify and analyze bot-generated affiliates

---

## Core Workflows

### 1. Signup as Affiliate (Bot Registration)

**Endpoint**: `POST /api/affiliates/signup`

**Authentication**: None required (public endpoint)

**Request Body**:
```json
{
  "name": "Crypto Education Bot",
  "email": "crypto-edu-bot@example.com",
  "cryptoAddress": "YOUR_WALLET_ADDRESS_HERE",
  "customCode": "CRYPTOEDU",
  "isAgent": true
}
```

**Parameters**:
- `name` (required): Affiliate display name
- `email` (required): Must match authenticated session email
- `cryptoAddress` (required): Crypto wallet address for payouts
- `customCode` (optional): Custom discount code (if not provided, auto-generated)
- `isAgent` (optional, **undocumented**): Boolean flag to mark as bot-created affiliate

**Response**:
```json
{
  "success": true,
  "affiliate": {
    "id": "507f1f77bcf86cd799439011",
    "name": "Crypto Education Bot",
    "email": "crypto-edu-bot@example.com",
    "discountCode": "CRYPTOEDU",
    "cryptoAddress": "YOUR_WALLET_ADDRESS_HERE",
    "apiKey": "kk_live_a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456"
  },
  "verificationEmailSent": true,
  "message": "Please check your email for a verification code. Save your API key - it will not be shown again!"
}
```

**Notes**:
- **No authentication required** - fully autonomous bot signup
- Creates Shopify discount code automatically (10% discount)
- Fixed commission: $20 per sale
- **Returns API key** - save immediately, shown only once
- Initial status: `isApproved: false` (requires admin approval)
- Sends verification email to provided email address
- Human must verify email via link sent
- If `isAgent: true`, stores this flag for tracking (hidden from UI)

**CRITICAL**: Store the `apiKey` from the response securely. Use it for all subsequent API calls.

---

### 2. Get Affiliate Profile

**Endpoint**: `GET /api/affiliates`

**Authentication**: Required (API key)

**Headers**:
```
Authorization: Bearer kk_live_YOUR_API_KEY_HERE
```

**Response**:
```json
{
  "success": true,
  "data": {
    "_id": "507f1f77bcf86cd799439011",
    "name": "Crypto Education Bot",
    "email": "crypto-edu-bot@example.com",
    "discountCode": "CRYPTOEDU",
    "cryptoAddress": "YOUR_WALLET_ADDRESS_HERE",
    "isApproved": true,
    "isActive": true,
    "joinedAt": "2024-01-15T10:30:00.000Z",
    "totalClicks": 150,
    "totalOrders": 5,
    "totalCommission": 100,
    "totalCommissionFromOrders": 100,
    "totalBonusEarned": 0,
    "availableCommission": 100,
    "pendingCommission": 0,
    "orders": [],
    "bonuses": null
  }
}
```

**Key Fields**:
- `isApproved`: Whether admin approved the affiliate
- `totalCommission`: Total earnings (orders + bonuses)
- `availableCommission`: Amount available for withdrawal
- `pendingCommission`: Amount in pending payout requests
- `orders`: Array of order objects with commission details

---

### 3. Get Analytics

**Endpoint**: `GET /api/affiliates/analytics?discountCode={CODE}&period={PERIOD}`

**Authentication**: Required (API key)

**Headers**:
```
Authorization: Bearer kk_live_YOUR_API_KEY_HERE
```

**Query Parameters**:
- `discountCode` (required): Your discount code
- `period` (optional): `1day`, `7days`, or `30days` (default: `30days`)

**Response**:
```json
{
  "success": true,
  "data": {
    "summary": {
      "totalLandings": 150,
      "totalClicks": 45,
      "totalOrders": 5,
      "conversionRate": 3.33,
      "clickThroughRate": 30.0,
      "totalRevenue": 500.0,
      "totalCommission": 100.0,
      "averageOrderValue": 100.0,
      "todayClicks": 12,
      "todayLandings": 35
    },
    "funnel": [
      {
        "step": "landing",
        "name": "Landing Page Views",
        "count": 150,
        "percentage": 100
      },
      {
        "step": "clicks",
        "name": "Buy Button Clicks",
        "count": 45,
        "percentage": 30
      },
      {
        "step": "purchases",
        "name": "Purchases",
        "count": 5,
        "percentage": 3
      }
    ],
    "sources": [
      {
        "source": "twitter",
        "visitors": 100,
        "conversions": 3,
        "conversionRate": 3,
        "percentage": 67
      }
    ],
    "timeline": [
      {
        "date": "2024-02-01",
        "visitors": 50,
        "conversions": 2,
        "revenue": 200,
        "commission": 40
      }
    ],
    "insights": {
      "bestPerformingSource": "twitter",
      "clickInsights": ["Excellent click-through rate! Your audience is highly engaged"]
    }
  }
}
```

**Use Cases**:
- Monitor traffic and conversion performance
- Track daily/weekly/monthly trends
- Analyze traffic sources
- Optimize marketing strategies

---

### 4. Get Affiliate Links and Landing Pages

**Affiliate URLs**:
- Landing page: `https://keepkey.com/?ref={DISCOUNT_CODE}`
- Direct shop link: `https://keepkey.com/shop?discount={DISCOUNT_CODE}`

**Tracking**:
- Landing page visits tracked in Redis: `affiliate:{CODE}:landings`
- Buy button clicks tracked: `affiliate:{CODE}:buy_clicks`
- Orders tracked in MongoDB `orders` collection

---

### 5. Submit Withdrawal Request

**Endpoint**: `POST /api/affiliates/payouts/request`

**Authentication**: Required (API key)

**Headers**:
```
Authorization: Bearer kk_live_YOUR_API_KEY_HERE
```

**Request Body**:
```json
{
  "amount": 100.00,
  "cryptoAddress": "YOUR_WALLET_ADDRESS_HERE"
}
```

**Parameters**:
- `amount` (required): Withdrawal amount (must be ≤ available commission)
- `cryptoAddress` (required): Wallet address for payout

**Response**:
```json
{
  "success": true,
  "message": "Payout request submitted successfully",
  "payoutId": "507f1f77bcf86cd799439011",
  "withdrawalId": "WD-1706956800000-A1B2C3D",
  "amount": 100.00,
  "orderCount": 5
}
```

**Payout Process**:
1. Bot submits withdrawal request with amount and crypto address
2. System validates available balance (orders + bonuses)
3. Creates payout record with unique `withdrawalId`
4. Marks orders as assigned to this payout (prevents double-spending)
5. Admin processes payout manually (status: pending → completed)
6. Transaction hash recorded on completion

**Payout Statuses**:
- `pending`: Awaiting admin processing
- `completed`: Paid and confirmed
- `rejected`: Denied by admin

---

### 6. Check Verification Status

**Endpoint**: `GET /api/affiliates/status`

**Authentication**: Required (API key)

**Headers**:
```
Authorization: Bearer kk_live_YOUR_API_KEY_HERE
```

**Response**:
```json
{
  "success": true,
  "isAffiliate": true,
  "isApproved": true,
  "emailVerified": true,
  "profileCompleted": true,
  "applicationStatus": "approved"
}
```

**Application Statuses**:
- `pending`: Awaiting admin approval
- `approved`: Active affiliate
- `rejected`: Application denied

---

## Bot Operation Examples

### Example 1: Complete Bot Registration Flow

```javascript
// 1. Sign up as affiliate (no auth required)
const signupResponse = await fetch('https://affiliates.keepkey.com/api/affiliates/signup', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name: 'Crypto Education Bot',
    email: 'your-bot-email@example.com', // Human email for verification
    cryptoAddress: 'YOUR_WALLET_ADDRESS_HERE',
    customCode: 'CRYPTOEDU',
    isAgent: true // Mark as bot
  })
});

const { affiliate } = await signupResponse.json();
const API_KEY = affiliate.apiKey; // Save this securely!

// 2. Human verifies email via link sent to inbox

// 3. Poll for admin approval
const checkStatus = async () => {
  const response = await fetch('https://affiliates.keepkey.com/api/affiliates/status', {
    headers: {
      'Authorization': `Bearer ${API_KEY}`
    }
  });
  const { isApproved } = await response.json();
  return isApproved;
};

// 4. Once approved, start sharing affiliate links
```

### Example 2: Monitor Performance

```javascript
// Get analytics every hour
const API_KEY = 'kk_live_YOUR_API_KEY_HERE'; // From signup response

const analytics = await fetch(
  'https://affiliates.keepkey.com/api/affiliates/analytics?discountCode=CRYPTOEDU&period=7days',
  {
    headers: {
      'Authorization': `Bearer ${API_KEY}`
    }
  }
);

const data = await analytics.json();
console.log('Conversion Rate:', data.data.summary.conversionRate);
console.log('Total Commission:', data.data.summary.totalCommission);
```

### Example 3: Automated Withdrawal

```javascript
const API_KEY = 'kk_live_YOUR_API_KEY_HERE'; // From signup response

// Check available balance
const profile = await fetch('https://affiliates.keepkey.com/api/affiliates', {
  headers: {
    'Authorization': `Bearer ${API_KEY}`
  }
});
const { data } = await profile.json();
const availableCommission = data.availableCommission;

// Submit withdrawal if balance > threshold
if (availableCommission >= 100) {
  await fetch('https://affiliates.keepkey.com/api/affiliates/payouts/request', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${API_KEY}`
    },
    body: JSON.stringify({
      amount: availableCommission,
      cryptoAddress: 'YOUR_WALLET_ADDRESS_HERE'
    })
  });
}
```

---

## Database Schema Reference

### Affiliates Collection
```typescript
{
  _id: ObjectId,
  name: string,
  email: string,
  emailVerified: boolean,
  cryptoAddress: string,
  discountCode: string,
  customCode?: string,
  priceRuleId?: number,
  isApproved: boolean,
  isAdmin: boolean,
  isAgent?: boolean,              // Bot-created flag (undocumented)
  joinedAt: Date,
  totalClicks: number,
  totalOrders: number,
  totalCommission: number,
  availableCommission: number,
  pendingCommission: number,
  orders: Array<{
    orderId: string,
    amount: number,
    commission: number,
    date: Date,
    isPending: boolean,
    pendingUntil: Date
  }>,
  bonuses?: {
    signupBonus?: {...},
    salesBonus?: {...},
    totalBonusEarned: number
  }
}
```

### Orders Collection
```typescript
{
  _id: ObjectId,
  orderId: string,
  shopifyOrderId: string,
  affiliateId: ObjectId,
  orderAmount: number,
  commission: number,
  shopifyCreatedAt: Date,
  payoutId?: ObjectId,
  payoutStatus?: 'pending' | 'completed',
  status: 'active' | 'disputed'
}
```

### Payouts Collection
```typescript
{
  _id: ObjectId,
  withdrawalId: string,
  affiliateId: ObjectId,
  affiliateEmail: string,
  amount: number,
  cryptoAddress: string,
  orderIds: string[],
  orderCount: number,
  orderCommission: number,
  bonusAmount: number,
  status: 'pending' | 'completed' | 'rejected',
  requestedAt: Date,
  processedAt?: Date,
  transactionHash?: string,
  notes?: string
}
```

---

## Error Handling

### Common Error Responses

**401 Unauthorized**:
```json
{
  "success": false,
  "error": "Unauthorized"
}
```

**400 Bad Request**:
```json
{
  "success": false,
  "error": "Affiliate with this email already exists"
}
```

**404 Not Found**:
```json
{
  "success": false,
  "error": "Affiliate not found"
}
```

**500 Internal Server Error**:
```json
{
  "success": false,
  "error": "An unexpected error occurred"
}
```

---

## Rate Limiting & Best Practices

### Recommendations
1. **API Key Storage**: Store API key securely (environment variables, secrets manager)
2. **Polling Intervals**: Check analytics max once per hour
3. **Withdrawal Frequency**: Don't submit multiple pending withdrawals
4. **Error Handling**: Implement exponential backoff on failures
5. **Logging**: Log all API interactions for debugging (never log API keys)

### Bot Identification
- Use `isAgent: true` during signup to be tracked as a bot
- This flag is stored but not exposed in UI
- Admins can query `affiliates` collection: `{ isAgent: true }`
- Helps with analytics and bot performance tracking

---

## Testing Endpoints

### Signup (no auth required)
```bash
curl -X POST https://affiliates.keepkey.com/api/affiliates/signup \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Crypto Education Bot",
    "email": "your-email@example.com",
    "cryptoAddress": "YOUR_WALLET_ADDRESS_HERE",
    "customCode": "CRYPTOEDU",
    "isAgent": true
  }'
```

### Get Profile (with API key)
```bash
curl https://affiliates.keepkey.com/api/affiliates \
  -H "Authorization: Bearer kk_live_YOUR_API_KEY_HERE"
```

### Get Analytics (with API key)
```bash
curl 'https://affiliates.keepkey.com/api/affiliates/analytics?discountCode=CRYPTOEDU&period=7days' \
  -H "Authorization: Bearer kk_live_YOUR_API_KEY_HERE"
```

---

## Support & Troubleshooting

### Common Issues

**Email Verification Required**:
- Verification emails sent automatically on signup
- Check spam folder
- Resend via `/api/affiliates/send-verification`

**Not Approved**:
- New affiliates require admin approval
- Poll `/api/affiliates/status` for approval status
- Typical approval time: 24-48 hours

**Insufficient Balance**:
- Check `availableCommission` in profile
- Orders have 2-week pending period before payout
- Bonuses credited immediately

**Payout Pending**:
- Only one pending payout allowed at a time
- Wait for admin processing
- Check status via profile endpoint

---

## Admin Endpoints (Reference)

Bots cannot access these, but useful to understand the system:

- `POST /api/affiliates/approve` - Approve affiliate
- `POST /api/affiliates/reject` - Reject affiliate
- `POST /api/admin/payouts` - Process payouts
- `POST /api/admin/credit-order` - Manually credit order
- `POST /api/admin/sync-orders` - Sync Shopify orders

---

## Webhook Events (Future)

Currently not implemented, but planned:
- `affiliate.approved` - Affiliate approved by admin
- `order.created` - New order attributed to affiliate
- `payout.completed` - Withdrawal processed
- `bonus.earned` - Bonus criteria met

---

## Version History

- **v5.1**: API key authentication, autonomous bot signup
- **v5.0**: Bonus system and Redis analytics
- **v4.x**: Legacy LeadDyno import support
- **v3.x**: Initial Shopify integration

---

## Security Considerations

1. **Protect API keys** - Store securely, never commit to version control, never log
2. **API key = full account access** - Treat like a password, rotate if compromised
3. **Validate all inputs** - API validates but client should pre-validate
4. **Use HTTPS only** - No plain HTTP in production
5. **Secure crypto addresses** - **CRITICAL**: Replace `YOUR_WALLET_ADDRESS_HERE` with your actual wallet address. Never use example addresses from documentation. Validate wallet addresses before submission to avoid loss of funds.
6. **Rate limit requests** - Respect API rate limits to avoid blocks
7. **API key shown only once** - Save immediately after signup, cannot be retrieved later

---

## Conclusion

This skill enables **fully autonomous** bot operation for the KeepKey affiliate system. No OAuth, no sessions, no human interaction required.

### Key Features:
- ✅ **Autonomous signup** - No pre-approval needed, fully automated
- ✅ **API key authentication** - Simple Bearer token, no session management
- ✅ **Email verification** - Human validates via email link
- ✅ **Real-time analytics** - Traffic, conversions, revenue tracking
- ✅ **Automated withdrawals** - Programmatic payout requests
- ✅ **Agent tracking** - Bot affiliates tracked via `isAgent` flag (undocumented)

### Bot Workflow Summary:
1. Bot calls `/signup` endpoint → receives API key
2. Human verifies email via inbox link
3. Admin approves affiliate
4. Bot uses API key for all operations
5. Bot shares affiliate links and earns commissions
6. Bot requests withdrawals when threshold met

All operations mirror human affiliate capabilities with zero human intervention except email verification.
