# OpenNext.js SSR Test

This directory contains tests for running actual OpenNext.js Cloudflare bundled output in workerd.
The test verifies that workerd can correctly execute Next.js SSR applications built with the
`@opennextjs/cloudflare` adapter.

## Files

- `opennext-ssr-test.js` - Test cases for API routes, SSR pages, streaming, RSC, etc.
- `opennext-ssr-test.wd-test` - workerd test configuration
- `opennext-ssr-worker.js` - Bundled OpenNext.js worker (generated)

## Current Versions

- Next.js: 16.1.6
- @opennextjs/cloudflare: 1.16.1
- React: 19.x

## Regenerating the Bundled Worker

To regenerate `opennext-ssr-worker.js` with updated versions:

### 1. Create a new Next.js project

```bash
mkdir /tmp/nextjs-opennext-test && cd /tmp/nextjs-opennext-test
npm init -y
npm install next@latest react@latest react-dom@latest @opennextjs/cloudflare@latest
```

### 2. Create the project structure

**package.json** - Add scripts:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "opennext": "opennextjs-cloudflare build"
  }
}
```

**next.config.ts**:

```typescript
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  // Disable image optimization (requires external service)
  images: {
    unoptimized: true,
  },
};

export default nextConfig;
```

**app/layout.tsx**:

```tsx
export const metadata = {
  title: 'SSR Test App',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

**app/page.tsx**:

```tsx
import { Suspense } from 'react';
import { cookies } from 'next/headers';

async function AsyncSection({ id }: { id: string }) {
  return <div id={`async-section-${id}`}>Loaded data for section-{id}</div>;
}

export default async function Home() {
  const cookieStore = await cookies();
  const testCookie = cookieStore.get('test-cookie');

  return (
    <main>
      <h1>SSR Test Page</h1>
      <p id="server-time">Server rendered at: {Date.now()}</p>
      <p id="cookie-value">Cookie value: {testCookie?.value ?? 'not set'}</p>
      <section id="suspense-section">
        <h2>Suspense Boundaries</h2>
        <Suspense fallback={<div>Loading 1...</div>}>
          <AsyncSection id="1" />
        </Suspense>
        <Suspense fallback={<div>Loading 2...</div>}>
          <AsyncSection id="2" />
        </Suspense>
        <Suspense fallback={<div>Loading 3...</div>}>
          <AsyncSection id="3" />
        </Suspense>
      </section>
    </main>
  );
}
```

**app/posts/[id]/page.tsx** (dynamic route):

```tsx
export default async function PostPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;

  return (
    <main>
      <h1>Post {id}</h1>
      <p id="post-id">Post ID: {id}</p>
      <p id="render-time">Rendered at: {Date.now()}</p>
    </main>
  );
}
```

**app/streaming/page.tsx**:

```tsx
export default function StreamingPage() {
  return (
    <main>
      <h1>Streaming Test Page</h1>
      <div id="large-content">
        {Array.from({ length: 100 }, (_, i) => (
          <p key={i}>Content chunk {i + 1}</p>
        ))}
      </div>
    </main>
  );
}
```

**app/redirect-test/page.tsx**:

```tsx
import { redirect } from 'next/navigation';

export default function RedirectTestPage({
  searchParams,
}: {
  searchParams: Promise<{ target?: string }>;
}) {
  return <RedirectHandler searchParams={searchParams} />;
}

async function RedirectHandler({
  searchParams,
}: {
  searchParams: Promise<{ target?: string }>;
}) {
  const params = await searchParams;
  if (params.target) {
    redirect(params.target);
  }
  return (
    <main>
      <h1>Redirect Test</h1>
      <p>Add ?target=/path to redirect</p>
    </main>
  );
}
```

**app/api/data/route.ts**:

```typescript
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const headers: Record<string, string> = {};
  request.headers.forEach((value, key) => {
    headers[key] = value;
  });

  const url = new URL(request.url);
  const searchParams: Record<string, string> = {};
  url.searchParams.forEach((value, key) => {
    searchParams[key] = value;
  });

  return NextResponse.json({
    message: 'API response',
    timestamp: Date.now(),
    method: 'GET',
    headers,
    searchParams,
  });
}

export async function POST(request: NextRequest) {
  const body = await request.json().catch(() => ({}));

  return NextResponse.json({
    message: 'API response',
    timestamp: Date.now(),
    method: 'POST',
    received: body,
  });
}

export async function OPTIONS() {
  return new NextResponse(null, {
    status: 204,
    headers: {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
    },
  });
}
```

**app/api/cookies/route.ts**:

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { cookies } from 'next/headers';

export async function GET(request: NextRequest) {
  const cookieStore = await cookies();
  const allCookies: Record<string, string> = {};

  cookieStore.getAll().forEach((cookie) => {
    allCookies[cookie.name] = cookie.value;
  });

  return NextResponse.json({
    message: 'Cookies API',
    timestamp: Date.now(),
    cookies: allCookies,
  });
}

export async function POST(request: NextRequest) {
  const body = await request.json().catch(() => ({}));
  const { name, value, options } = body;

  const response = NextResponse.json({
    message: 'Cookie set',
    timestamp: Date.now(),
    cookie: { name, value },
  });

  if (name && value) {
    response.cookies.set(name, value, options || {});
  }

  return response;
}

export async function DELETE(request: NextRequest) {
  const url = new URL(request.url);
  const name = url.searchParams.get('name');

  const response = NextResponse.json({
    message: 'Cookie deleted',
    timestamp: Date.now(),
    deleted: name,
  });

  if (name) {
    response.cookies.delete(name);
  }

  return response;
}
```

**wrangler.jsonc**:

```jsonc
{
  "name": "nextjs-opennext-test",
  "main": ".open-next/worker.js",
  "compatibility_date": "2024-09-23",
  "compatibility_flags": ["nodejs_compat"],
  "assets": {
    "directory": ".open-next/assets",
    "binding": "ASSETS",
  },
}
```

### 3. Build with OpenNext

```bash
npm run opennext
```

### 4. Bundle with Wrangler

```bash
npx wrangler deploy --dry-run --outdir=dist
```

This creates `dist/worker.js` - the bundled worker.

### 5. Patch the worker

The bundled worker imports WASM modules for `@vercel/og` (image generation) which we don't need
for SSR tests. Stub these out:

```bash
# Find and replace the WASM imports at the top of the file
sed -i "s|import resvg_wasm from \"./.*-resvg.wasm?module\";|const resvg_wasm = null; // Stubbed - not needed for SSR tests|g" dist/worker.js
sed -i "s|import yoga_wasm from \"./.*-yoga.wasm?module\";|const yoga_wasm = null; // Stubbed - not needed for SSR tests|g" dist/worker.js
```

### 6. Copy to workerd

```bash
cp dist/worker.js /path/to/workerd/src/workerd/api/tests/opennextjs/opennext-ssr-worker.js
```

## Running the Test

```bash
bazel test //src/workerd/api/tests/opennextjs:opennext-ssr-test@
```
