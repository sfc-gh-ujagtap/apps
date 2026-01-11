---
name: build-react-app
description: "Build React/Next.js apps with Snowflake data. Use when: building dashboards, creating data apps, making analytics tools."
---

# Build React Apps with Snowflake Data

## Overview
Build data-intensive Next.js applications that connect to Snowflake. Create dashboards, analytics tools, admin panels, and customer-facing data products.

**For deployment to SPCS, use the `deploy-to-spcs` skill after completing this workflow.**

## Tools Used
- `snowflake_object_search` - Find tables/views for the app
- `ai_browser` - Test local app
- `bash` - Run npm commands

## Stopping Points
- ⚠️ Step 1: Confirm requirements before coding
- ⚠️ Step 4: Confirm app works locally

---

## Workflow

### Step 1: Understand Requirements

Before writing any code, clarify with the user:
- What data should the app use? (Search for tables/views using `snowflake_object_search`)
- What type of application? (Dashboard, admin panel, customer-facing tool, data explorer)
- Any specific aesthetic direction? (Minimal, bold, playful, enterprise, etc.)

**CRITICAL: NEVER use mock/hardcoded data. Always connect to real Snowflake tables.**

**⚠️ STOP:** Confirm requirements with user before proceeding.

---

### Step 2: Tech Stack

#### Prerequisites
```bash
node --version  # Must be v20.x.x or higher
docker --version
```

#### Create Next.js Project
```bash
npx create-next-app@latest <app-name> --typescript --tailwind --eslint --app --src-dir=false --import-alias="@/*"
cd <app-name>
```

#### Initialize shadcn/ui (REQUIRED)
```bash
npx shadcn@latest init -d
npx shadcn@latest add card chart button table select input tabs badge skeleton dialog dropdown-menu separator tooltip sidebar
```

#### Install Dependencies
```bash
npm install recharts@2.15.4 lucide-react snowflake-sdk
```

#### Configure next.config.ts
```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
  serverExternalPackages: ["snowflake-sdk"],
};

export default nextConfig;
```

---

### Step 3: Build the Application

#### Project Structure
```
<app-name>/
├── app/
│   ├── page.tsx
│   ├── layout.tsx
│   ├── globals.css
│   └── api/
├── components/
├── lib/
│   ├── snowflake.ts
│   └── utils.ts
├── Dockerfile
└── next.config.ts
```

#### Snowflake Connection (lib/snowflake.ts)

| Environment | Auth Method | How It Works |
|-------------|-------------|--------------|
| **Local Development** | External Browser (SSO) | Opens browser for login on first API request |
| **SPCS Production** | OAuth Token | Auto-injected at `/snowflake/session/token` |

```typescript
import snowflake from "snowflake-sdk";
import fs from "fs";

snowflake.configure({ logLevel: "ERROR" });

let connection: snowflake.Connection | null = null;
let cachedToken: string | null = null;

function getOAuthToken(): string | null {
  const tokenPath = "/snowflake/session/token";
  try {
    if (fs.existsSync(tokenPath)) {
      return fs.readFileSync(tokenPath, "utf8");
    }
  } catch {
    // Not in SPCS environment
  }
  return null;
}

function getConfig(): snowflake.ConnectionOptions {
  const base = {
    account: process.env.SNOWFLAKE_ACCOUNT || "<account>",
    warehouse: process.env.SNOWFLAKE_WAREHOUSE || "<warehouse>",
    database: process.env.SNOWFLAKE_DATABASE || "<database>",
    schema: process.env.SNOWFLAKE_SCHEMA || "<schema>",
  };

  const token = getOAuthToken();
  if (token) {
    return {
      ...base,
      host: process.env.SNOWFLAKE_HOST,
      token,
      authenticator: "oauth",
    };
  }

  return {
    ...base,
    username: process.env.SNOWFLAKE_USER || "<username>",
    authenticator: "EXTERNALBROWSER",
  };
}

async function getConnection(): Promise<snowflake.Connection> {
  const token = getOAuthToken();

  if (connection && (!token || token === cachedToken)) {
    return connection;
  }

  if (connection) {
    console.log("OAuth token changed, reconnecting");
    connection.destroy(() => {});
  }

  console.log(token ? "Connecting with OAuth token" : "Connecting with external browser");
  const conn = snowflake.createConnection(getConfig());
  await conn.connectAsync(() => {});
  connection = conn;
  cachedToken = token;
  return connection;
}

function isRetryableError(err: unknown): boolean {
  const error = err as { message?: string; code?: number };
  return !!(
    error.message?.includes("OAuth access token expired") ||
    error.message?.includes("terminated connection") ||
    error.code === 407002
  );
}

export async function query<T>(sql: string, retries = 1): Promise<T[]> {
  try {
    const conn = await getConnection();
    return await new Promise<T[]>((resolve, reject) => {
      conn.execute({
        sqlText: sql,
        complete: (err, stmt, rows) => {
          if (err) {
            reject(err);
          } else {
            resolve((rows || []) as T[]);
          }
        },
      });
    });
  } catch (err) {
    console.error("Query error:", (err as Error).message);
    if (retries > 0 && isRetryableError(err)) {
      connection = null;
      return query(sql, retries - 1);
    }
    throw err;
  }
}
```

#### API Routes

All API routes MUST query real Snowflake data:

```typescript
// app/api/data/route.ts
import { NextResponse } from "next/server";
import { query } from "@/lib/snowflake";

export async function GET() {
  try {
    const results = await query<{ COL1: string; COL2: number }>(`
      SELECT COL1, COL2 
      FROM <DATABASE>.<SCHEMA>.<TABLE>
      LIMIT 100
    `);
    return NextResponse.json(results);
  } catch (error) {
    console.error("Error:", error);
    return NextResponse.json({ error: "Failed to fetch data" }, { status: 500 });
  }
}
```

#### Design Guidelines

**Design Thinking**
- Identify the purpose: What problem does this interface solve? Who uses it?
- Choose a tone: minimal/refined, bold/maximalist, playful, editorial, industrial, or soft/approachable
- Create differentiation: What makes this memorable? Avoid generic "AI slop" aesthetics

**Visual Design**
- Typography: Choose fonts with character; avoid overused defaults (Inter, Roboto, Arial)
- Color: Use CSS variables (`--chart-1` through `--chart-5`) for charts; never hardcode colors; support dark mode
- Motion: Add purposeful animations for page loads, hover states, and data transitions
- Layout: Be intentional about density; use consistent spacing tokens (`space-y-6`, `gap-4`/`gap-6`)

**Component Usage**
- Use shadcn/ui components exclusively—never raw HTML tables, inputs, or buttons
- Cards: `Card` for content containers
- Tables: `Table` for data display
- Status: `Badge` for indicators
- Loading: `Skeleton` matching layout structure
- Charts: `ChartContainer` with Recharts using `hsl(var(--chart-X))` colors
- Icons: Lucide React consistently

**Avoid**
- Generic purple gradients on white backgrounds
- Cookie-cutter layouts without context-specific character
- Raw HTML elements when shadcn components exist
- Hardcoded colors instead of CSS variables

---

### Step 4: Test Locally (REQUIRED)

```bash
npm run dev
```

App runs at `http://localhost:3000`. Browser opens for SSO on first API request.

#### Verify Using Browser Tool
```
ai_browser(
  initial_url="http://localhost:3000",
  instructions="Verify app loads with REAL Snowflake data and UI is polished."
)
```

#### Build Test
```bash
npm run build
```

**⚠️ STOP:** Ask user to confirm the app looks correct.

---

## Next Steps

To deploy to Snowflake, use the `deploy-to-spcs` skill which handles:
- Compute pool and image repository setup
- Docker image building and pushing
- Service creation and monitoring

## Output
- Working Next.js app with Snowflake connection
- Local development server at localhost:3000
- Production-ready build
