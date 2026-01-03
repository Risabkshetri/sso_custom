# NCS ↔ Zobique SSO Integration Guide

> **Single Sign-On integration between NCS (Government Portal) and Zobique**

| Application | Description | Team |
|-------------|-------------|------|
| **NCS** | National Career Service - Government Portal (Phone/Aadhaar Auth) | NCS Team |
| **Zobique** | Next.js Career Platform | Zobique Team |

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication Flow](#authentication-flow)
3. [Architecture](#architecture)
4. [NCS API Requirements](#ncs-api-requirements)
5. [Zobique Implementation](#zobique-implementation)
6. [Security Guidelines](#security-guidelines)
7. [Testing & Checklist](#testing--checklist)

---

## Overview

Users authenticated on **NCS** (via Phone OTP or Aadhaar eKYC) can access **Zobique** without re-authentication.

### Key Points

- NCS uses **custom government authentication** (not OAuth)
- Authentication methods: **Phone OTP** / **Aadhaar eKYC**
- NCS issues a signed token after verification
- Zobique validates the token server-side and creates a session

---

## Authentication Flow

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                           NCS → Zobique SSO Flow                                    │
└────────────────────────────────────────────────────────────────────────────────────┘

   ┌──────────┐                    ┌──────────────┐                    ┌──────────────┐
   │   User   │                    │   Zobique    │                    │     NCS      │
   │ (Browser)│                    │  (Next.js)   │                    │ (Govt Portal)│
   └────┬─────┘                    └──────┬───────┘                    └──────┬───────┘
        │                                 │                                   │
        │ 1. Click "Login with NCS"       │                                   │
        │────────────────────────────────>│                                   │
        │                                 │                                   │
        │ 2. Redirect to NCS              │                                   │
        │<────────────────────────────────│                                   │
        │    /auth/login?                 │                                   │
        │      client_id=ZOBIQUE_ID       │                                   │
        │      &redirect_uri=/callback    │                                   │
        │      &state=<random>            │                                   │
        │                                 │                                   │
        │ 3. User enters Phone/Aadhaar    │                                   │
        │─────────────────────────────────────────────────────────────────────>│
        │                                 │                                   │
        │ 4. NCS sends OTP / eKYC         │                                   │
        │<─────────────────────────────────────────────────────────────────────│
        │                                 │                                   │
        │ 5. User verifies OTP            │                                   │
        │─────────────────────────────────────────────────────────────────────>│
        │                                 │                                   │
        │ 6. Redirect with auth_token     │                                   │
        │<─────────────────────────────────────────────────────────────────────│
        │    /callback?token=<TOKEN>&state=<random>                           │
        │                                 │                                   │
        │ 7. Browser follows redirect     │                                   │
        │────────────────────────────────>│                                   │
        │                                 │                                   │
        │                                 │ 8. Validate token                 │
        │                                 │   POST /api/verify-token          │
        │                                 │──────────────────────────────────>│
        │                                 │                                   │
        │                                 │ 9. Return user profile            │
        │                                 │<──────────────────────────────────│
        │                                 │                                   │
        │ 10. Create session & redirect   │                                   │
        │<────────────────────────────────│                                   │
        │                                 │                                   │
   ┌────┴─────┐                    ┌──────┴───────┐                    ┌──────┴───────┐
   │  Logged  │                    │   Zobique    │                    │     NCS      │
   │    In    │                    │              │                    │              │
   └──────────┘                    └──────────────┘                    └──────────────┘
```

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              System Architecture                                    │
└────────────────────────────────────────────────────────────────────────────────────┘

                                 ┌─────────────────┐
                                 │    End User     │
                                 └────────┬────────┘
                                          │
                       ┌──────────────────┴──────────────────┐
                       │                                     │
                       ▼                                     ▼
           ┌─────────────────────┐               ┌─────────────────────┐
           │      Zobique        │               │        NCS          │
           │     (Next.js)       │               │   (Govt Portal)     │
           │                     │    HTTPS      │                     │
           │  • Callback Handler │ ◄───────────► │  • Phone OTP Auth   │
           │  • Token Validation │               │  • Aadhaar eKYC     │
           │  • Session Mgmt     │               │  • Token Issuer     │
           │  • Protected Routes │               │  • User Database    │
           └─────────────────────┘               └─────────────────────┘
```

---

## Team Responsibilities

| Component | NCS | Zobique |
|-----------|:---:|:-------:|
| User Authentication (Phone/Aadhaar) | ✅ | - |
| Token Generation & Signing | ✅ | - |
| Token Verification API | ✅ | - |
| User Profile API | ✅ | - |
| Login Page Redirect | - | ✅ |
| Callback Handler | - | ✅ |
| Session Management | - | ✅ |
| Protected Routes Middleware | - | ✅ |

---

## NCS API Requirements

### 1. Login Redirect Endpoint

```
GET /auth/login
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `client_id` | Yes | Zobique's registered client ID |
| `redirect_uri` | Yes | `https://zobique.com/auth/callback` |
| `state` | Yes | Random string for CSRF protection |

**Response:** Displays NCS login page (Phone/Aadhaar input)

---

### 2. Token Verification Endpoint

```
POST /api/verify-token
Content-Type: application/json
X-API-Key: <ZOBIQUE_API_KEY>
```

**Request:**
```json
{
  "token": "<auth_token_from_callback>"
}
```

**Success Response (200):**
```json
{
  "valid": true,
  "user": {
    "ncs_id": "NCS123456789",
    "name": "Rahul Sharma",
    "phone": "9876543210",
    "email": "rahul@example.com",
    "aadhaar_verified": true,
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

**Error Response (401):**
```json
{
  "valid": false,
  "error": "token_expired"
}
```

---

### 3. User Profile Endpoint (Optional)

```
GET /api/user-profile
Authorization: Bearer <access_token>
```

**Response:**
```json
{
  "ncs_id": "NCS123456789",
  "name": "Rahul Sharma",
  "phone": "9876543210",
  "email": "rahul@example.com",
  "qualifications": [...],
  "skills": [...],
  "work_experience": [...]
}
```

---

## Zobique Implementation

### Environment Variables

```env
# NCS Configuration
NCS_BASE_URL=https://ncs.gov.in
NCS_CLIENT_ID=zobique_prod_client
NCS_API_KEY=sk_live_xxxxx

# App URLs
NEXT_PUBLIC_APP_URL=https://zobique.com
NCS_CALLBACK_URI=https://zobique.com/auth/callback

# Session
SESSION_SECRET=your-32-char-secret
```

---

### 1. Login Initiation

**File:** `app/api/auth/ncs/login/route.ts`

```typescript
import { NextResponse } from 'next/server';
import { cookies } from 'next/headers';
import crypto from 'crypto';

export async function GET() {
  const state = crypto.randomBytes(16).toString('hex');
  
  // Store state for CSRF validation
  cookies().set('ncs_state', state, {
    httpOnly: true,
    secure: true,
    sameSite: 'lax',
    maxAge: 600 // 10 minutes
  });
  
  const loginUrl = new URL(`${process.env.NCS_BASE_URL}/auth/login`);
  loginUrl.searchParams.set('client_id', process.env.NCS_CLIENT_ID!);
  loginUrl.searchParams.set('redirect_uri', process.env.NCS_CALLBACK_URI!);
  loginUrl.searchParams.set('state', state);
  
  return NextResponse.redirect(loginUrl.toString());
}
```

---

### 2. Callback Handler

**File:** `app/api/auth/ncs/callback/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { cookies } from 'next/headers';
import { createSession } from '@/lib/session';

export async function GET(request: NextRequest) {
  const { searchParams } = request.nextUrl;
  const token = searchParams.get('token');
  const state = searchParams.get('state');
  const error = searchParams.get('error');
  
  // Handle errors from NCS
  if (error) {
    return NextResponse.redirect(`${process.env.NEXT_PUBLIC_APP_URL}/auth/error?error=${error}`);
  }
  
  // Validate state (CSRF protection)
  const storedState = cookies().get('ncs_state')?.value;
  if (!state || state !== storedState) {
    return NextResponse.redirect(`${process.env.NEXT_PUBLIC_APP_URL}/auth/error?error=invalid_state`);
  }
  
  if (!token) {
    return NextResponse.redirect(`${process.env.NEXT_PUBLIC_APP_URL}/auth/error?error=missing_token`);
  }
  
  try {
    // Verify token with NCS
    const verifyResponse = await fetch(`${process.env.NCS_BASE_URL}/api/verify-token`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': process.env.NCS_API_KEY!
      },
      body: JSON.stringify({ token })
    });
    
    if (!verifyResponse.ok) {
      return NextResponse.redirect(`${process.env.NEXT_PUBLIC_APP_URL}/auth/error?error=verification_failed`);
    }
    
    const { valid, user } = await verifyResponse.json();
    
    if (!valid) {
      return NextResponse.redirect(`${process.env.NEXT_PUBLIC_APP_URL}/auth/error?error=invalid_token`);
    }
    
    // Create session and redirect to dashboard
    const response = NextResponse.redirect(`${process.env.NEXT_PUBLIC_APP_URL}/dashboard`);
    await createSession(response, { user, token });
    
    // Clear state cookie
    response.cookies.delete('ncs_state');
    
    return response;
    
  } catch (error) {
    console.error('NCS callback error:', error);
    return NextResponse.redirect(`${process.env.NEXT_PUBLIC_APP_URL}/auth/error?error=server_error`);
  }
}
```

---

### 3. Session Management

**File:** `lib/session.ts`

```typescript
import { SignJWT, jwtVerify } from 'jose';
import { NextRequest, NextResponse } from 'next/server';

const SESSION_NAME = 'zobique_session';
const secret = new TextEncoder().encode(process.env.SESSION_SECRET);

interface SessionData {
  user: {
    ncs_id: string;
    name: string;
    phone: string;
    email?: string;
  };
  expiresAt: number;
}

export async function createSession(response: NextResponse, data: { user: any; token: string }) {
  const sessionData: SessionData = {
    user: data.user,
    expiresAt: Date.now() + 24 * 60 * 60 * 1000 // 24 hours
  };
  
  const jwt = await new SignJWT(sessionData as any)
    .setProtectedHeader({ alg: 'HS256' })
    .setExpirationTime('24h')
    .sign(secret);
  
  response.cookies.set(SESSION_NAME, jwt, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 86400 // 24 hours
  });
}

export async function getSession(request: NextRequest): Promise<SessionData | null> {
  const token = request.cookies.get(SESSION_NAME)?.value;
  if (!token) return null;
  
  try {
    const { payload } = await jwtVerify(token, secret);
    return payload as unknown as SessionData;
  } catch {
    return null;
  }
}

export function clearSession(response: NextResponse) {
  response.cookies.delete(SESSION_NAME);
}
```

---

### 4. Protected Routes Middleware

**File:** `middleware.ts`

```typescript
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { getSession } from '@/lib/session';

const protectedRoutes = ['/dashboard', '/profile', '/jobs'];
const authRoutes = ['/auth/login'];

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const session = await getSession(request);
  
  const isProtected = protectedRoutes.some(route => pathname.startsWith(route));
  const isAuthRoute = authRoutes.some(route => pathname.startsWith(route));
  
  if (isProtected && !session) {
    return NextResponse.redirect(new URL('/auth/login', request.url));
  }
  
  if (isAuthRoute && session) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }
  
  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/profile/:path*', '/jobs/:path*', '/auth/:path*']
};
```

---

### 5. Logout

**File:** `app/api/auth/logout/route.ts`

```typescript
import { NextResponse } from 'next/server';
import { clearSession } from '@/lib/session';

export async function POST() {
  const response = NextResponse.json({ success: true });
  clearSession(response);
  return response;
}
```

---

## Security Guidelines

| Area | Requirement |
|------|-------------|
| **HTTPS** | All endpoints must use HTTPS |
| **State Parameter** | Always validate to prevent CSRF |
| **Token Validation** | Always verify tokens server-side with NCS |
| **API Key** | Never expose `NCS_API_KEY` in frontend |
| **Cookies** | Use `httpOnly`, `secure`, `sameSite` flags |
| **Token Expiry** | Tokens should expire within 5-10 minutes |

---

## Data Exchange Contract

### NCS Provides to Zobique

| Item | Example |
|------|---------|
| Client ID | `zobique_prod_client` |
| API Key | `sk_live_xxxxx` |
| Auth Base URL | `https://ncs.gov.in` |
| Login Endpoint | `/auth/login` |
| Verify Token Endpoint | `/api/verify-token` |
| Token Expiry | 5 minutes |

### Zobique Provides to NCS

| Item | Example |
|------|---------|
| Application Name | `Zobique` |
| Callback URL (Prod) | `https://zobique.com/auth/callback` |
| Callback URL (Dev) | `http://localhost:3000/auth/callback` |

---

## Testing & Checklist

### Test Scenarios

| # | Scenario | Expected Result |
|---|----------|-----------------|
| 1 | Click "Login with NCS" | Redirect to NCS login page |
| 2 | Complete Phone OTP verification | Redirect back to Zobique dashboard |
| 3 | Complete Aadhaar eKYC | Redirect back to Zobique dashboard |
| 4 | Cancel on NCS login | Redirect to Zobique with error |
| 5 | Tampered state parameter | Show "invalid state" error |
| 6 | Expired token | Show "token expired" error |
| 7 | Access protected route without session | Redirect to login |
| 8 | Click logout | Clear session, redirect to home |

### Implementation Checklist

**NCS Team:**
- [ ] Register Zobique as client
- [ ] Implement login redirect with callback support
- [ ] Implement token verification API
- [ ] Provide API credentials to Zobique
- [ ] Set token expiry (5-10 min recommended)

**Zobique Team:**
- [ ] Configure environment variables
- [ ] Implement login initiation route
- [ ] Implement callback handler
- [ ] Implement session management
- [ ] Add protected routes middleware
- [ ] Implement logout
- [ ] Add error handling pages
- [ ] Test end-to-end flow

---

## Error Codes

| Code | Description | Action |
|------|-------------|--------|
| `invalid_state` | CSRF validation failed | Retry login |
| `missing_token` | Token not in callback | Retry login |
| `token_expired` | Token expired | Retry login |
| `verification_failed` | NCS rejected token | Contact support |
| `user_cancelled` | User cancelled login | Show login option |

---

**Document Version:** 1.0  
**Last Updated:** January 3, 2026  
**Maintained By:** Zobique Team
