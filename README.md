<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/2f0ea161-6855-42ff-b237-415c71b51725" />

# Cashback Backend - Coupon Redemption API

> Node.js · Express · MongoDB · Cashfree Payouts · Fast2SMS OTP

![Node](https://img.shields.io/badge/Node.js-20_LTS-green)
![Express](https://img.shields.io/badge/Express-4.18-blue)
![MongoDB](https://img.shields.io/badge/MongoDB-Mongoose_8-green)
![License](https://img.shields.io/badge/license-MIT-green)

---

## What This Is

A production backend for a cashback coupon redemption platform. Users authenticate via OTP, submit
a coupon code, and receive an instant UPI payout via Cashfree. Payout lifecycle is tracked through
a state machine (`unused → pending → redeemed / failed`) and finalized by signed webhooks — not polling.

19 REST endpoints, 2 MongoDB collections, 7 middleware layers, role-based access, atomic state
transitions that prevent double-redemption, webhook signature verification, and a multi-layer
rate limiter protecting both the OTP flow and the general API surface.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│               Clients · Web · Mobile                │
└──────────────────────┬──────────────────────────────┘
                       │ HTTPS
                       ▼
┌─────────────────────────────────────────────────────┐
│                  Express API                        │
│                                                     │
│  Middleware chain (per request):                    │
│  requestId → helmet → cors → morgan                 │
│  → rateLimiter → jwtAuth → adminGuard               │
│  → validateRequest → controller                     │
│                                                     │
│  5 route modules · 19 endpoints                     │
└──────────────────┬──────────────────────────────────┘
                   │
     ┌─────────────┴──────────────┐
     ▼                            ▼
┌──────────────┐        ┌──────────────────────────┐
│  MongoDB     │        │  External Services        │
│              │        │                          │
│  2 models    │        │  Cashfree Payouts V2      │
│  User        │        │  Fast2SMS (OTP)           │
│  Coupon      │        │  Razorpay (legacy)        │
└──────────────┘        └──────────────────────────┘
```
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/f6140bee-ec36-4493-a04b-7877005d327d" />

---

## Why These Patterns

| Pattern | Problem It Solves | Where |
|---|---|---|
| **Coupon State Machine** | User double-taps "Redeem". Two requests land simultaneously. Both succeed. Same coupon paid out twice. | `couponService.js` |
| **Atomic Status Transition** | Race condition between concurrent redemption requests. `findOneAndUpdate` with status filter ensures exactly one request wins. | `couponController.js` |
| **Webhook-Driven Finalization** | Polling Cashfree for transfer status burns API quota and adds latency. Webhooks push status on state change. | `webhookController.js` |
| **HMAC Signature Verification** | Any HTTP client can POST to `/webhooks/cashfree` and fake a successful payout. Cashfree signs payloads — backend verifies before acting. | `webhookController.js` |
| **Dual JWT Audiences** | Single token type means a stolen user token works on admin endpoints. `client` and `admin` audiences are verified separately. | `userService.js`, `jwtAuth.js` |
| **OTP Rate Limiting** | Attacker enumerates phone numbers by flooding OTP endpoint. 15 attempts / 10-min window per phone enforced before SMS is sent. | `otpAttemptLimiter.js` |
| **Fallback Payout Strategy** | Cashfree V2 beneficiary creation can fail for new UPI IDs. Service tries V2 with beneficiary, falls back to V2 direct transfer before failing. | `payoutService.js` |
| **Request ID Tracing** | Support ticket arrives: "payout failed." No way to correlate log lines across middleware. UUID injected on every request, forwarded on all downstream calls. | `requestId.js` |

---

## Coupon Lifecycle

```
                      POST /coupons/redeem
                             │
                    ┌────────▼────────┐
                    │   status check  │  findOneAndUpdate({ code, status: "unused" })
                    └────────┬────────┘
                             │ match found
                    ┌────────▼────────┐
                    │    "pending"    │  atomic write — all concurrent requests
                    └────────┬────────┘  see "pending", only one wins
                             │
                    ┌────────▼────────┐
                    │ Cashfree payout │  POST /transfers (V2 API)
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
    POST /webhooks/cashfree
    TRANSFER_SUCCESS  TRANSFER_FAILED  TRANSFER_REVERSED
              │              │              │
         "redeemed"      "failed"       "failed"
                        (reset-able     (logged)
                         to "unused")
```

---

## Module Overview

| Module | Prefix | Notable Design |
|---|---|---|
| Auth | `/api/v1/auth` | OTP via Fast2SMS, JWT with audience separation, admin phone/password login |
| Coupons | `/api/v1/coupons` | Atomic state machine redemption, UPI payout trigger, history |
| Dashboard | `/api/v1/dashboard` | Admin: bulk coupon creation, user listing, JSON export |
| Payouts | `/api/v1/payouts` | Manual admin payout trigger (RazorpayX, legacy path) |
| Webhooks | `/api/v1/webhooks` | Cashfree HMAC-verified transfer callbacks, manual coupon reset |

---

## Endpoints

**Auth** (`/api/v1/auth`)
```
POST  /send-otp        OTP dispatch (rate limited: 15/10 min per phone)
POST  /precheck        Profile completeness check before OTP
POST  /verify-otp      OTP verification → JWT issued
POST  /admin-login     Admin login via phone + password
POST  /admins-create   Bootstrap first admin account
POST  /update-profile  Update name, UPI ID, address
```

**Coupons** (`/api/v1/coupons`)
```
POST  /redeem          Atomic redemption + Cashfree payout trigger
POST  /history         User's redemption history
```

**Dashboard** (`/api/v1/dashboard`) — admin only
```
GET   /coupons         List all coupons (filtered, paginated)
POST  /coupons         Bulk or custom coupon creation
GET   /users           List all users
POST  /sellers         Create seller account
GET   /logged-in-users Users with active OTP window
GET   /coupons/export  JSON export of redeemed coupons
```

**Payouts** (`/api/v1/payouts`) — admin only
```
POST  /razorpayx       Trigger RazorpayX payout
```

**Webhooks** (`/api/v1/webhooks`)
```
POST  /cashfree        Cashfree transfer webhook (HMAC verified)
GET   /cashfree        Webhook connectivity test
GET   /test            Health test
POST  /reset-coupon    Manually reset failed coupon to "unused"
```

---

## Data Models

**User**
```
phone          String   required, unique, indexed
email          String   required
role           Enum     "user" | "admin"
upiId          String   UPI address for payouts
firstName      String
lastName       String
pincode        String   6-digit
state          String
city           String
otp            String   hashed
otpExpires     Date
coupons        [ObjectId → Coupon]
```

**Coupon**
```
code           String   unique, indexed
amount         Number   INR payout amount
status         Enum     "unused" | "pending" | "redeemed" | "failed"
redeemedBy     ObjectId → User
userPhone      String   snapshot at redemption time
upiIdUsed      String   snapshot at redemption time
payoutId       String   Cashfree / RazorpayX transfer ID
payoutMeta     Mixed    full provider response
redeemedAt     Date
lockedAt       Date     when status → pending
failedAt       Date
attempts       Number   retry count
failureReason  String
batchId        String   indexed, for coupon grouping
expiryDate     Date
```

---

## Numbers

| Dimension | Count |
|---|---|
| REST endpoints | 19 |
| MongoDB models | 2 |
| Route modules | 5 |
| Middleware layers per request | 7 |
| Coupon status states | 4 |
| External service integrations | 3 |
| JWT audience types | 2 |
| Rate limiter layers | 2 |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20 LTS |
| HTTP | Express 4.18 |
| Database | MongoDB · Mongoose 8 |
| Auth | JWT (jsonwebtoken 9) · bcryptjs |
| Validation | express-validator 7 · Joi 17 |
| Payments | Cashfree Payouts V2 · Razorpay 2.8 |
| SMS | Fast2SMS |
| Security | Helmet · CORS · express-rate-limit |
| Logging | Winston · Morgan |
| API Docs | Swagger UI (swagger-jsdoc + swagger-ui-express) |
| Utilities | uuid · axios · dotenv · compression |

---

## Quick Start

```bash
git clone https://github.com/bimalray/cashback-backend
cd cashback-backend
npm install

cp .env.example .env
# Fill in MONGO_URI, JWT_SECRET, CASHFREE_*, FAST2SMS_*

npm run dev

# API:    http://localhost:5000
# Health: http://localhost:5000/health
# Docs:   http://localhost:5000/api-docs
```

---

## Environment Variables

| Variable | Required | Notes |
|---|---|---|
| `PORT` | No | Default: 5000 |
| `MONGO_URI` | Yes | MongoDB connection string |
| `JWT_SECRET` | Yes | Signing secret for all tokens |
| `CASHFREE_CLIENT_ID` | Yes | Cashfree API client ID |
| `CASHFREE_CLIENT_SECRET` | Yes | Cashfree API secret + webhook HMAC key |
| `CASHFREE_ENVIRONMENT` | No | `"PROD"` or `"SANDBOX"` (default: PROD) |
| `FAST2SMS_API_KEY` | Yes | Fast2SMS authorization key |
| `FAST2SMS_TEMPLATE_ID` | Yes | DLT-approved OTP template ID |
| `FAST2SMS_SENDER_ID` | Yes | Registered sender ID |
| `OTP_EXP_MINUTES` | No | OTP validity window (default: 5) |
| `OTP_RESEND_COOLDOWN_SEC` | No | Resend cooldown (default: 45) |
| `ALLOWED_ORIGIN` | No | CORS allowed origins |
| `CASHBACK_NARRATION` | No | Payout narration text (default: "Cashback") |
| `RAZORPAY_KEY_ID` | Yes* | Required by schema; legacy path only |
| `RAZORPAY_KEY_SECRET` | Yes* | Required by schema; legacy path only |

---

[LinkedIn](https://www.linkedin.com/in/bimal-ray)
