# API Route Pattern

Next.js API routes handle server-side operations that don't fit server actions — token refresh, webhooks, and proxy endpoints. They live in `src/app/api/`.

---

## File locations

```
src/app/api/
  auth/
    refresh/route.ts        ← token refresh
  webhooks/
    <provider>/route.ts     ← external webhook handlers
  health/route.ts           ← health check endpoint
```

---

## Structure — token refresh

```ts
// src/app/api/auth/refresh/route.ts
import { cookies } from 'next/headers'
import { NextResponse } from 'next/server'

export async function POST() {
  const cookieStore = await cookies()
  const refreshToken = cookieStore.get('refresh_token')?.value

  if (!refreshToken) {
    return NextResponse.json(
      { message: 'No refresh token' },
      { status: 401 },
    )
  }

  const response = await fetch(`${process.env.API_URL}/v1/auth/refresh`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken }),
  })

  if (!response.ok) {
    cookieStore.delete('refresh_token')
    return NextResponse.json(
      { message: 'Token refresh failed' },
      { status: 401 },
    )
  }

  const data = await response.json()

  cookieStore.set('refresh_token', data.refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    path: '/',
    maxAge: 60 * 60 * 24 * 7, // 7 days
  })

  return NextResponse.json({ accessToken: data.accessToken })
}
```

---

## Structure — webhook handler

```ts
// src/app/api/webhooks/<provider>/route.ts
import { headers } from 'next/headers'
import { NextResponse } from 'next/server'

export async function POST(request: Request) {
  const headersList = await headers()
  const signature = headersList.get('x-webhook-signature')

  if (!signature || !verifySignature(signature, await request.text())) {
    return NextResponse.json({ message: 'Invalid signature' }, { status: 401 })
  }

  const body = await request.json()

  // Process webhook event
  switch (body.event) {
    case 'payment.completed':
      // forward to backend API or process directly
      break
    default:
      // unknown event — log and acknowledge
      break
  }

  return NextResponse.json({ received: true })
}

function verifySignature(signature: string, body: string): boolean {
  // verify HMAC signature with provider's secret
  return true // implement per provider
}
```

---

## Structure — health check

```ts
// src/app/api/health/route.ts
import { NextResponse } from 'next/server'

export async function GET() {
  return NextResponse.json({ status: 'ok', timestamp: new Date().toISOString() })
}
```

---

## Rules

- API routes are for infrastructure concerns only — never put business logic here
- Use `route.ts` (not `page.ts`) — exports HTTP method functions (`GET`, `POST`, `PUT`, `DELETE`)
- Always validate incoming data — webhooks must verify signatures
- Cookies: always set `httpOnly`, `secure` (in production), `sameSite`
- Never expose secrets in responses — token refresh returns access token only, refresh token stays in httpOnly cookie
- Health check endpoint should be public and lightweight
- For business operations, prefer server actions over API routes

---

## Anti-patterns

```ts
// ❌ business logic in API route
export async function POST(request: Request) {
  const body = await request.json()
  const order = await prisma.order.create({ data: body })  // use backend API
  return NextResponse.json(order)
}

// ❌ no signature verification on webhook
export async function POST(request: Request) {
  const body = await request.json()
  // processing without verifying signature — security risk
}

// ❌ refresh token in response body
return NextResponse.json({
  accessToken: data.accessToken,
  refreshToken: data.refreshToken,  // keep in httpOnly cookie only
})

// ❌ non-httpOnly cookie for tokens
cookieStore.set('token', value)  // always set httpOnly: true for auth tokens

// ❌ using page.ts instead of route.ts
// src/app/api/auth/page.ts  ← this creates a page, not an API route
```
