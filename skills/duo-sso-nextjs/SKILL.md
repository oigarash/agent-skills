---
name: duo-sso-nextjs
description: Implement Duo SSO (OIDC/OAuth2.1) authentication with PKCE flow in Next.js App Router. Use when adding Duo authentication, implementing SSO login, or setting up OIDC authentication without NextAuth.js.
---

# Duo SSO Authentication for Next.js

Duo SSO authentication implementation using OAuth2.1 with PKCE flow for Next.js App Router (v14/15).

## Prerequisites

- Next.js 14+ with App Router
- `jose` library for JWT encryption
- Duo Admin Panel access for OAuth configuration

```bash
npm install jose
```

## Environment Variables

```env
# Duo SSO OAuth
DUO_ISSUER=https://sso-xxx.sso.duosecurity.com/oidc/DIxxx
DUO_CLIENT_ID=your_client_id
DUO_CLIENT_SECRET=your_client_secret
DUO_REDIRECT_URI=http://localhost:3000/api/auth/callback

# Session (generate with: openssl rand -base64 32)
SESSION_SECRET=your_32_byte_random_string_here

# Optional: Base path if app is served under a subpath
NEXT_PUBLIC_BASE_PATH=

# Optional: Application URL (required for Kubernetes/reverse proxy)
# Set this to your external URL for correct redirects after login
APP_URL=https://your-domain.com
```

## File Structure

```
src/
├── lib/auth/
│   ├── duo-oauth.ts    # Duo OAuth helpers (PKCE, token exchange)
│   ├── session.ts      # JWT session management
│   └── dal.ts          # Data Access Layer
├── app/api/auth/
│   ├── login/route.ts  # Initiate OAuth flow
│   ├── callback/route.ts # Handle OAuth callback
│   └── logout/route.ts # Clear session
├── app/login/page.tsx  # Login page UI
└── middleware.ts       # Route protection
```

## Implementation

### 1. Duo OAuth Helper (`src/lib/auth/duo-oauth.ts`)

```typescript
export interface DuoOAuthConfig {
  issuer: string;
  clientId: string;
  clientSecret: string;
  redirectUri: string;
}

export interface DuoUserInfo {
  sub: string;
  email: string;
  // Standard OIDC claims
  name?: string;
  given_name?: string;
  family_name?: string;
  preferred_username?: string;
  picture?: string;
  // Duo custom claims (configured in Duo Admin)
  fullname?: string;
  firstname?: string;
  lastname?: string;
  username?: string;
}

export function getDuoConfig(): DuoOAuthConfig {
  const issuer = process.env.DUO_ISSUER;
  const clientId = process.env.DUO_CLIENT_ID;
  const clientSecret = process.env.DUO_CLIENT_SECRET;
  const redirectUri = process.env.DUO_REDIRECT_URI;

  if (!issuer || !clientId || !clientSecret || !redirectUri) {
    throw new Error('Missing Duo OAuth environment variables');
  }

  return { issuer, clientId, clientSecret, redirectUri };
}

// PKCE: Generate code verifier (32 bytes, base64url encoded)
export function generateCodeVerifier(): string {
  const array = new Uint8Array(32);
  crypto.getRandomValues(array);
  return base64UrlEncode(array);
}

// PKCE: Generate code challenge (SHA-256 hash of verifier)
export async function generateCodeChallenge(verifier: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(verifier);
  const hash = await crypto.subtle.digest('SHA-256', data);
  return base64UrlEncode(new Uint8Array(hash));
}

// Generate random state for CSRF protection
export function generateState(): string {
  const array = new Uint8Array(16);
  crypto.getRandomValues(array);
  return base64UrlEncode(array);
}

// Build Duo authorization URL
export async function buildAuthorizationUrl(
  codeVerifier: string,
  state: string
): Promise<string> {
  const config = getDuoConfig();
  const codeChallenge = await generateCodeChallenge(codeVerifier);

  const params = new URLSearchParams({
    response_type: 'code',
    client_id: config.clientId,
    redirect_uri: config.redirectUri,
    scope: 'openid email profile',
    state: state,
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
  });

  return `${config.issuer}/authorize?${params.toString()}`;
}

// Exchange authorization code for tokens
export async function exchangeCodeForTokens(
  code: string,
  codeVerifier: string
): Promise<{ access_token: string; id_token: string }> {
  const config = getDuoConfig();

  const params = new URLSearchParams({
    grant_type: 'authorization_code',
    code: code,
    redirect_uri: config.redirectUri,
    client_id: config.clientId,
    client_secret: config.clientSecret,
    code_verifier: codeVerifier,
  });

  const response = await fetch(`${config.issuer}/token`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: params.toString(),
  });

  if (!response.ok) {
    throw new Error(`Token exchange failed: ${response.status}`);
  }

  return response.json();
}

// Get user info from userinfo endpoint
export async function getUserInfo(accessToken: string): Promise<DuoUserInfo> {
  const config = getDuoConfig();

  const response = await fetch(`${config.issuer}/userinfo`, {
    headers: { Authorization: `Bearer ${accessToken}` },
  });

  if (!response.ok) {
    throw new Error(`Failed to get user info: ${response.status}`);
  }

  return response.json();
}

// Parse ID token (fallback when userinfo fails)
export function parseIdToken(idToken: string): DuoUserInfo {
  const parts = idToken.split('.');
  if (parts.length !== 3) throw new Error('Invalid ID token');

  const payload = JSON.parse(base64UrlDecode(parts[1]));

  return {
    sub: payload.sub,
    email: payload.email,
    name: payload.name,
    given_name: payload.given_name,
    family_name: payload.family_name,
    preferred_username: payload.preferred_username,
    picture: payload.picture,
    fullname: payload.fullname,
    firstname: payload.firstname,
    lastname: payload.lastname,
    username: payload.username,
  };
}

function base64UrlEncode(buffer: Uint8Array): string {
  let binary = '';
  for (let i = 0; i < buffer.length; i++) {
    binary += String.fromCharCode(buffer[i]);
  }
  return btoa(binary).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

function base64UrlDecode(str: string): string {
  str = str.replace(/-/g, '+').replace(/_/g, '/');
  const pad = str.length % 4;
  if (pad) str += '='.repeat(4 - pad);
  return atob(str);
}
```

### 2. Session Management (`src/lib/auth/session.ts`)

```typescript
import { EncryptJWT, jwtDecrypt } from 'jose';
import { cookies } from 'next/headers';

const SESSION_COOKIE_NAME = 'app-session';
const SESSION_DURATION = 7 * 24 * 60 * 60 * 1000; // 7 days

export interface SessionPayload {
  userId: string;
  duoId: string;
  email: string;
  name: string;
  role: 'admin' | 'user';
  expiresAt: Date;
}

async function getSessionKey(): Promise<Uint8Array> {
  const secret = process.env.SESSION_SECRET;
  if (!secret) throw new Error('SESSION_SECRET not set');

  const encoder = new TextEncoder();
  const keyData = encoder.encode(secret);

  // Hash to ensure 32 bytes for A256GCM
  if (keyData.length !== 32) {
    const hash = await crypto.subtle.digest('SHA-256', keyData);
    return new Uint8Array(hash);
  }
  return keyData;
}

export async function createSession(
  payload: Omit<SessionPayload, 'expiresAt'>
): Promise<string> {
  const key = await getSessionKey();
  const expiresAt = new Date(Date.now() + SESSION_DURATION);

  return new EncryptJWT({ ...payload, expiresAt: expiresAt.toISOString() })
    .setProtectedHeader({ alg: 'dir', enc: 'A256GCM' })
    .setIssuedAt()
    .setExpirationTime(expiresAt)
    .encrypt(key);
}

export async function decryptSession(
  token: string
): Promise<SessionPayload | null> {
  try {
    const key = await getSessionKey();
    const { payload } = await jwtDecrypt(token, key);

    const expiresAt = new Date(payload.expiresAt as string);
    if (expiresAt < new Date()) return null;

    return {
      userId: payload.userId as string,
      duoId: payload.duoId as string,
      email: payload.email as string,
      name: payload.name as string,
      role: payload.role as 'admin' | 'user',
      expiresAt,
    };
  } catch {
    return null;
  }
}

export async function getSession(): Promise<SessionPayload | null> {
  const cookieStore = await cookies();
  const cookie = cookieStore.get(SESSION_COOKIE_NAME);
  if (!cookie?.value) return null;
  return decryptSession(cookie.value);
}

export async function setSessionCookie(token: string): Promise<void> {
  const cookieStore = await cookies();
  cookieStore.set(SESSION_COOKIE_NAME, token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    path: '/',
    maxAge: SESSION_DURATION / 1000,
  });
}

export async function clearSessionCookie(): Promise<void> {
  const cookieStore = await cookies();
  cookieStore.delete(SESSION_COOKIE_NAME);
}

// OAuth flow cookies (PKCE verifier, state, return URL)
export const OAUTH_COOKIES = {
  CODE_VERIFIER: 'oauth-code-verifier',
  STATE: 'oauth-state',
  RETURN_TO: 'oauth-return-to',
} as const;

export async function setOAuthCookies(
  codeVerifier: string,
  state: string,
  returnTo?: string
): Promise<void> {
  const cookieStore = await cookies();
  const options = {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax' as const,
    path: '/',
    maxAge: 10 * 60, // 10 minutes
  };

  cookieStore.set(OAUTH_COOKIES.CODE_VERIFIER, codeVerifier, options);
  cookieStore.set(OAUTH_COOKIES.STATE, state, options);
  if (returnTo) cookieStore.set(OAUTH_COOKIES.RETURN_TO, returnTo, options);
}

export async function getAndClearOAuthCookies(): Promise<{
  codeVerifier?: string;
  state?: string;
  returnTo?: string;
}> {
  const cookieStore = await cookies();

  const result = {
    codeVerifier: cookieStore.get(OAUTH_COOKIES.CODE_VERIFIER)?.value,
    state: cookieStore.get(OAUTH_COOKIES.STATE)?.value,
    returnTo: cookieStore.get(OAUTH_COOKIES.RETURN_TO)?.value,
  };

  cookieStore.delete(OAUTH_COOKIES.CODE_VERIFIER);
  cookieStore.delete(OAUTH_COOKIES.STATE);
  cookieStore.delete(OAUTH_COOKIES.RETURN_TO);

  return result;
}
```

### 3. Login Route (`src/app/api/auth/login/route.ts`)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import {
  generateCodeVerifier,
  generateState,
  buildAuthorizationUrl,
} from '@/lib/auth/duo-oauth';
import { setOAuthCookies } from '@/lib/auth/session';

export async function GET(request: NextRequest) {
  const basePath = process.env.NEXT_PUBLIC_BASE_PATH || '';

  try {
    const returnTo = request.nextUrl.searchParams.get('returnTo') || '/dashboard';

    const codeVerifier = generateCodeVerifier();
    const state = generateState();

    await setOAuthCookies(codeVerifier, state, returnTo);

    const authUrl = await buildAuthorizationUrl(codeVerifier, state);
    return NextResponse.redirect(authUrl);
  } catch (error) {
    console.error('Login error:', error);
    return NextResponse.redirect(
      new URL(`${basePath}/login?error=oauth_error`, request.url)
    );
  }
}
```

### 4. Callback Route (`src/app/api/auth/callback/route.ts`)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import {
  exchangeCodeForTokens,
  getUserInfo,
  parseIdToken,
} from '@/lib/auth/duo-oauth';
import {
  createSession,
  setSessionCookie,
  getAndClearOAuthCookies,
} from '@/lib/auth/session';
// Import your DB connection and User model

export async function GET(request: NextRequest) {
  const basePath = process.env.NEXT_PUBLIC_BASE_PATH || '';

  try {
    const searchParams = request.nextUrl.searchParams;
    const code = searchParams.get('code');
    const state = searchParams.get('state');
    const error = searchParams.get('error');

    if (error) {
      return NextResponse.redirect(
        new URL(`${basePath}/login?error=${error}`, request.url)
      );
    }

    const { codeVerifier, state: savedState, returnTo } =
      await getAndClearOAuthCookies();

    if (!code || !codeVerifier) {
      return NextResponse.redirect(
        new URL(`${basePath}/login?error=missing_params`, request.url)
      );
    }

    // CSRF protection
    if (state !== savedState) {
      return NextResponse.redirect(
        new URL(`${basePath}/login?error=invalid_state`, request.url)
      );
    }

    // Exchange code for tokens
    const tokens = await exchangeCodeForTokens(code, codeVerifier);

    // Get user info
    let userInfo;
    try {
      userInfo = await getUserInfo(tokens.access_token);
    } catch {
      userInfo = parseIdToken(tokens.id_token);
    }

    // Construct full name (Duo custom claims > OIDC claims > email)
    const fullName =
      userInfo.fullname ||
      userInfo.name ||
      (userInfo.firstname && userInfo.lastname
        ? `${userInfo.firstname} ${userInfo.lastname}`
        : null) ||
      (userInfo.given_name && userInfo.family_name
        ? `${userInfo.given_name} ${userInfo.family_name}`
        : null) ||
      userInfo.firstname ||
      userInfo.given_name ||
      userInfo.username ||
      userInfo.preferred_username ||
      userInfo.email;

    // Upsert user to database (implement your own)
    // const user = await upsertUser(userInfo.sub, userInfo.email, fullName);

    // Create session
    const sessionToken = await createSession({
      userId: 'user_id_from_db',
      duoId: userInfo.sub,
      email: userInfo.email,
      name: fullName,
      role: 'user',
    });

    await setSessionCookie(sessionToken);

    return NextResponse.redirect(
      new URL(`${basePath}${returnTo || '/dashboard'}`, request.url)
    );
  } catch (error) {
    console.error('Callback error:', error);
    return NextResponse.redirect(
      new URL(`${basePath}/login?error=callback_error`, request.url)
    );
  }
}
```

### 5. Middleware (`src/middleware.ts`)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { decryptSession } from '@/lib/auth/session';

const publicRoutes = ['/login', '/api/auth'];

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const basePath = process.env.NEXT_PUBLIC_BASE_PATH || '';

  // Skip public routes
  if (publicRoutes.some((route) => pathname.startsWith(route))) {
    return NextResponse.next();
  }

  // Check session
  const sessionCookie = request.cookies.get('app-session');

  if (!sessionCookie?.value) {
    return redirectToLogin(request, basePath);
  }

  const session = await decryptSession(sessionCookie.value);

  if (!session) {
    return redirectToLogin(request, basePath);
  }

  // Add user info to headers
  const requestHeaders = new Headers(request.headers);
  requestHeaders.set('x-user-id', session.userId);
  requestHeaders.set('x-user-email', session.email);
  requestHeaders.set('x-user-role', session.role);

  return NextResponse.next({ request: { headers: requestHeaders } });
}

function redirectToLogin(request: NextRequest, basePath: string) {
  const loginUrl = new URL(`${basePath}/login`, request.url);
  const returnTo = request.nextUrl.pathname + request.nextUrl.search;
  if (returnTo && returnTo !== '/') {
    loginUrl.searchParams.set('returnTo', returnTo);
  }
  return NextResponse.redirect(loginUrl);
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

## Duo Admin Configuration

1. **Create OIDC Application** in Duo Admin Panel
2. **Redirect URI**: Set to `{YOUR_APP_URL}/api/auth/callback`
3. **Grant Types**: Authorization Code
4. **PKCE**: Enable S256 method
5. **Scopes**: `openid`, `email`, `profile`
6. **Profile Attributes**: Configure custom claims (fullname, firstname, lastname, username) if needed

## Important: Redirect URL in Kubernetes/Reverse Proxy

When deploying behind a reverse proxy (Kubernetes Ingress, nginx, etc.), `request.url` returns the internal URL (e.g., `http://localhost:3000`), causing incorrect redirects after login.

**Solution**: Use `APP_URL` environment variable or `X-Forwarded-*` headers.

Add this helper function to routes that perform redirects:

```typescript
// Get the correct base URL for redirects
const getBaseUrl = (request: NextRequest) => {
  // 1. Use APP_URL if set (recommended for production)
  if (process.env.APP_URL) {
    return process.env.APP_URL;
  }
  // 2. Use X-Forwarded headers from reverse proxy
  const forwardedProto = request.headers.get('x-forwarded-proto');
  const forwardedHost = request.headers.get('x-forwarded-host');
  if (forwardedProto && forwardedHost) {
    return `${forwardedProto}://${forwardedHost}`;
  }
  // 3. Fallback to request.url (local development)
  return request.url;
};

// Use in redirects
const baseUrl = getBaseUrl(request);
return NextResponse.redirect(new URL(`${basePath}/dashboard`, baseUrl));
```

**Environment variable**:
```env
# Application URL (required for Kubernetes/reverse proxy)
APP_URL=https://your-domain.com
```

**Apply to these files**:
- `src/app/api/auth/callback/route.ts` - All redirects
- `src/app/api/auth/login/route.ts` - Error redirect
- `src/middleware.ts` - Login redirect

## OAuth Flow Summary

```
1. User clicks "Login with Duo"
      ↓
2. /api/auth/login generates PKCE verifier + state, stores in cookies
      ↓
3. Redirect to Duo authorization endpoint
      ↓
4. User authenticates with Duo
      ↓
5. Duo redirects to /api/auth/callback with code + state
      ↓
6. Validate state, exchange code for tokens using PKCE verifier
      ↓
7. Get user info from userinfo endpoint or ID token
      ↓
8. Create encrypted session JWT, set as httpOnly cookie
      ↓
9. Redirect to original destination
```
