# FurnishFlow — Complete Technical Requirements

**Version:** 1.0  
**Date:** May 2026  
**Status:** Draft  

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Technology Stack](#3-technology-stack)
4. [Repository & Project Structure](#4-repository--project-structure)
5. [Environment & Configuration](#5-environment--configuration)
6. [Database Schema](#6-database-schema)
7. [Telegram Bot Service](#7-telegram-bot-service)
8. [AI Processing Pipeline](#8-ai-processing-pipeline)
9. [Image Processing Service](#9-image-processing-service)
10. [3D Generation Service](#10-3d-generation-service)
11. [Backend REST API](#11-backend-rest-api)
12. [Authentication](#12-authentication)
13. [Frontend — Product Pages](#13-frontend--product-pages)
14. [Frontend — Owner Dashboard](#14-frontend--owner-dashboard)
15. [File Storage](#15-file-storage)
16. [Job Queue System](#16-job-queue-system)
17. [Notification System](#17-notification-system)
18. [Analytics](#18-analytics)
19. [Subscription & Billing](#19-subscription--billing)
20. [Security Requirements](#20-security-requirements)
21. [Performance Requirements](#21-performance-requirements)
22. [Error Handling & Logging](#22-error-handling--logging)
23. [Testing Requirements](#23-testing-requirements)
24. [Deployment & CI/CD](#24-deployment--cicd)
25. [Phase-by-Phase Build Plan](#25-phase-by-phase-build-plan)

---

## 1. Project Overview

### 1.1 What FurnishFlow Does

FurnishFlow is a SaaS platform that allows small furniture business owners to publish professional product catalog pages by simply sending structured text and photos to a Telegram bot. The system automatically:

- Parses the product details from free-form structured text
- Downloads and enhances the product images
- Generates 3D model views of the product
- Publishes a public product page with a 3D viewer, image gallery, and full spec table
- Sends the owner a confirmation with the live URL — all within 2–3 minutes

### 1.2 Core User Flow

```
Owner sends text + images to Telegram bot
         ↓
Bot stores raw input, pushes job to queue
         ↓
Worker picks up job → Claude API parses text → structured JSON
         ↓
Image pipeline → background removal → enhancement → WebP compression
         ↓
3D pipeline → Meshy.ai API → .glb model → uploaded to R2
         ↓
Product record created in database
         ↓
Next.js static product page generated
         ↓
Bot sends owner the live URL
```

### 1.3 Three User-Facing Surfaces

| Surface | URL Pattern | Audience |
|---|---|---|
| Telegram bot | t.me/furnishflowbot | Business owners (input) |
| Product page | furnishflow.com/p/[slug] | End customers (public) |
| Owner dashboard | furnishflow.com/dashboard | Business owners (management) |
| Catalog page | furnishflow.com/c/[shop-slug] | End customers (public) |

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                         │
│  Telegram App   │   Product Page (Next.js)  │  Dashboard    │
└────────┬────────┴──────────────┬────────────┴──────┬────────┘
         │                       │                   │
         ▼                       ▼                   ▼
┌────────────────┐    ┌─────────────────┐   ┌───────────────┐
│  Telegram Bot  │    │   Vercel Edge   │   │  Vercel Edge  │
│  Webhook       │    │   (product SSG) │   │  (dashboard)  │
│  (Railway)     │    └────────┬────────┘   └───────┬───────┘
└───────┬────────┘             │                    │
        │                      └──────────┬─────────┘
        ▼                                 ▼
┌───────────────────────────────────────────────────────────┐
│                    BACKEND API (Railway)                   │
│               Node.js + Fastify + TypeScript              │
└───────────────────────┬───────────────────────────────────┘
                        │
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Supabase    │ │ Upstash Redis│ │ Cloudflare   │
│  PostgreSQL  │ │ Job Queue    │ │ R2 Storage   │
└──────────────┘ └──────┬───────┘ └──────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────────┐
│                  WORKER SERVICE (Railway)                  │
│         Pulls jobs → runs AI pipeline → updates DB        │
└──────────┬────────────────────┬──────────────────────────┘
           ▼                    ▼
┌─────────────────┐   ┌──────────────────────┐
│  Anthropic API  │   │  Meshy.ai 3D API     │
│  (Claude Vision)│   │  (image-to-3D)       │
└─────────────────┘   └──────────────────────┘
```

### 2.2 Services Overview

| Service | Runtime | Host | Responsibility |
|---|---|---|---|
| `bot-service` | Node.js | Railway | Telegram webhook, user sessions |
| `api-service` | Node.js (Fastify) | Railway | REST API for dashboard + product pages |
| `worker-service` | Node.js | Railway | Processes jobs from queue (AI pipeline) |
| `image-service` | Python (FastAPI) | Railway | rembg background removal, ESRGAN upscale |
| `frontend` | Next.js 15 | Vercel | Dashboard + product pages + catalog pages |

---

## 3. Technology Stack

### 3.1 Backend

| Category | Choice | Reason |
|---|---|---|
| Runtime | Node.js 20 LTS | Stable, wide ecosystem |
| Language | TypeScript 5 | Type safety across all services |
| API framework | Fastify 4 | Faster than Express, schema validation built-in |
| ORM | Prisma 5 | Type-safe DB access, auto-migrations |
| Validation | Zod | Runtime schema validation, pairs with TypeScript |
| Job queue | BullMQ | Redis-backed, retries, priorities, dead-letter |
| HTTP client | ky | Lightweight, fetch-based, works in Node + browser |

### 3.2 Frontend

| Category | Choice | Reason |
|---|---|---|
| Framework | Next.js 15 (App Router) | SSG for product pages, RSC for dashboard |
| Styling | Tailwind CSS 4 | Utility-first, fast iteration |
| UI components | shadcn/ui | Headless, accessible, customizable |
| 3D viewer | @google/model-viewer | Web component, .glb support, AR built-in |
| State | Zustand | Lightweight client state for dashboard |
| Data fetching | TanStack Query | Cache, revalidate, optimistic updates |
| Forms | React Hook Form + Zod | Type-safe forms with validation |

### 3.3 AI & Processing

| Category | Choice | Version/Notes |
|---|---|---|
| LLM | Anthropic Claude | claude-sonnet-4-20250514 |
| 3D generation | Meshy.ai API | image-to-3D endpoint |
| Background removal | rembg | Python, runs locally on Railway |
| Image upscaling | Real-ESRGAN | Python, 2× upscale |
| Image processing | sharp | Node.js, WebP compression |

### 3.4 Infrastructure

| Category | Choice | Notes |
|---|---|---|
| Frontend hosting | Vercel | Next.js native, edge CDN |
| Backend hosting | Railway | Node.js services + Python worker |
| Database | Supabase (PostgreSQL 15) | Managed, real-time, row-level security |
| File storage | Cloudflare R2 | S3-compatible, zero egress fees |
| CDN | Cloudflare | Global, free tier, cache rules |
| Queue / cache | Upstash Redis | Serverless Redis, per-request billing |
| Email | Resend | Transactional emails |
| Payments | Stripe | Subscriptions + billing portal |
| Monitoring | Better Stack (Logtail) | Logs + uptime monitoring |

---

## 4. Repository & Project Structure

### 4.1 Monorepo Layout

```
furnishflow/
├── apps/
│   ├── bot/                    # Telegram bot service
│   │   ├── src/
│   │   │   ├── handlers/       # Message handlers
│   │   │   ├── commands/       # /start /help /new /edit /delete
│   │   │   ├── middleware/     # Auth, rate limit, session
│   │   │   ├── templates/      # Message templates
│   │   │   └── index.ts        # Entry point
│   │   ├── package.json
│   │   └── Dockerfile
│   │
│   ├── api/                    # REST API service
│   │   ├── src/
│   │   │   ├── routes/         # Route handlers
│   │   │   ├── middleware/     # Auth, CORS, rate limit
│   │   │   ├── services/       # Business logic
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── Dockerfile
│   │
│   ├── worker/                 # Background job processor
│   │   ├── src/
│   │   │   ├── jobs/           # Job handlers per type
│   │   │   ├── ai/             # Claude API integration
│   │   │   ├── storage/        # R2 upload helpers
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── Dockerfile
│   │
│   ├── image-service/          # Python image processing
│   │   ├── app/
│   │   │   ├── main.py         # FastAPI app
│   │   │   ├── routes/
│   │   │   │   ├── remove_bg.py
│   │   │   │   └── upscale.py
│   │   │   └── utils/
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   │
│   └── web/                    # Next.js frontend
│       ├── app/
│       │   ├── (public)/
│       │   │   ├── p/[slug]/   # Product page
│       │   │   └── c/[slug]/   # Catalog page
│       │   ├── (auth)/
│       │   │   └── dashboard/  # Owner dashboard
│       │   └── api/            # Next.js API routes (auth callbacks)
│       ├── components/
│       │   ├── product/        # ProductCard, ProductViewer, SpecTable
│       │   ├── dashboard/      # CatalogTable, ProductEditor, Analytics
│       │   └── ui/             # Shared UI components
│       ├── lib/
│       │   ├── api.ts          # API client
│       │   └── auth.ts         # Auth helpers
│       └── package.json
│
├── packages/
│   ├── database/               # Prisma schema + client
│   │   ├── prisma/
│   │   │   └── schema.prisma
│   │   └── index.ts
│   ├── types/                  # Shared TypeScript types
│   │   └── index.ts
│   └── config/                 # Shared config (Zod env schemas)
│       └── index.ts
│
├── package.json                # pnpm workspace root
├── pnpm-workspace.yaml
└── turbo.json                  # Turborepo config
```

### 4.2 Tooling

- **Package manager:** pnpm 9 with workspaces
- **Monorepo orchestration:** Turborepo
- **Linting:** ESLint + Prettier
- **Git hooks:** Husky + lint-staged
- **Commit format:** Conventional Commits

---

## 5. Environment & Configuration

### 5.1 Required Environment Variables

```env
# ── Telegram ──────────────────────────────────────────────
TELEGRAM_BOT_TOKEN=                  # From @BotFather
TELEGRAM_WEBHOOK_SECRET=             # Random 256-bit string
TELEGRAM_BOT_USERNAME=               # e.g. furnishflowbot

# ── Database ──────────────────────────────────────────────
DATABASE_URL=                        # Supabase PostgreSQL connection string
DIRECT_URL=                          # Supabase direct URL (for Prisma migrations)

# ── Redis ─────────────────────────────────────────────────
REDIS_URL=                           # Upstash Redis REST URL
REDIS_TOKEN=                         # Upstash Redis token

# ── Cloudflare R2 ─────────────────────────────────────────
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET_NAME=furnishflow-media
R2_PUBLIC_URL=                       # CDN URL prefix for public assets

# ── Anthropic ─────────────────────────────────────────────
ANTHROPIC_API_KEY=

# ── Meshy.ai ──────────────────────────────────────────────
MESHY_API_KEY=

# ── Stripe ────────────────────────────────────────────────
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_PRICE_STARTER=                # Stripe Price ID for Starter plan
STRIPE_PRICE_PRO=                    # Stripe Price ID for Pro plan
STRIPE_PRICE_BUSINESS=               # Stripe Price ID for Business plan

# ── Auth ──────────────────────────────────────────────────
JWT_SECRET=                          # 256-bit secret for signing JWTs
JWT_EXPIRES_IN=7d

# ── Image Service ─────────────────────────────────────────
IMAGE_SERVICE_URL=                   # Internal Railway URL for Python service
IMAGE_SERVICE_SECRET=                # Shared secret for internal auth

# ── App ───────────────────────────────────────────────────
APP_URL=https://furnishflow.com
NODE_ENV=production
PORT=3000
```

### 5.2 Configuration Validation

All environment variables must be validated at startup using a Zod schema. The app must crash with a descriptive error if any required variable is missing or malformed.

```typescript
// packages/config/src/index.ts
import { z } from 'zod'

const envSchema = z.object({
  TELEGRAM_BOT_TOKEN: z.string().min(1),
  DATABASE_URL: z.string().url(),
  ANTHROPIC_API_KEY: z.string().startsWith('sk-ant-'),
  // ... all variables
})

export const env = envSchema.parse(process.env)
```

---

## 6. Database Schema

### 6.1 Prisma Schema

```prisma
// packages/database/prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}

// ── Users ──────────────────────────────────────────────────────────────────

model User {
  id                String    @id @default(uuid())
  telegramId        BigInt    @unique @map("telegram_id")
  telegramUsername  String?   @map("telegram_username")
  firstName         String?   @map("first_name")
  lastName          String?   @map("last_name")
  shopName          String?   @map("shop_name")
  shopDescription   String?   @map("shop_description")
  catalogSlug       String?   @unique @map("catalog_slug")
  logoUrl           String?   @map("logo_url")
  contactPhone      String?   @map("contact_phone")
  contactEmail      String?   @map("contact_email")
  location          String?
  plan              Plan      @default(STARTER)
  planExpiresAt     DateTime? @map("plan_expires_at")
  stripeCustomerId  String?   @unique @map("stripe_customer_id")
  stripeSubId       String?   @unique @map("stripe_sub_id")
  createdAt         DateTime  @default(now()) @map("created_at")
  updatedAt         DateTime  @updatedAt @map("updated_at")

  products  Product[]
  sessions  Session[]

  @@map("users")
}

enum Plan {
  STARTER
  PROFESSIONAL
  BUSINESS
}

// ── Sessions ───────────────────────────────────────────────────────────────

model Session {
  id        String   @id @default(uuid())
  userId    String   @map("user_id")
  token     String   @unique
  expiresAt DateTime @map("expires_at")
  createdAt DateTime @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("sessions")
}

// ── Products ───────────────────────────────────────────────────────────────

model Product {
  id            String        @id @default(uuid())
  userId        String        @map("user_id")
  slug          String        @unique
  name          String
  price         Decimal?      @db.Decimal(10, 2)
  currency      String        @default("USD") @db.VarChar(3)
  dimensionW    Float?        @map("dimension_w")
  dimensionH    Float?        @map("dimension_h")
  dimensionD    Float?        @map("dimension_d")
  dimensionUnit String?       @default("cm") @map("dimension_unit")
  material      String?
  color         String?
  weight        Float?
  stockStatus   StockStatus   @default(IN_STOCK) @map("stock_status")
  description   String?       @db.Text
  rawInput      String?       @map("raw_input") @db.Text
  status        ProductStatus @default(PROCESSING)
  telegramMsgId BigInt?       @map("telegram_msg_id")
  viewCount     Int           @default(0) @map("view_count")
  createdAt     DateTime      @default(now()) @map("created_at")
  updatedAt     DateTime      @updatedAt @map("updated_at")

  user    User           @relation(fields: [userId], references: [id], onDelete: Cascade)
  media   ProductMedia[]
  events  Event[]
  inquiries Inquiry[]

  @@map("products")
}

enum StockStatus {
  IN_STOCK
  OUT_OF_STOCK
  MADE_TO_ORDER
}

enum ProductStatus {
  PROCESSING
  LIVE
  DRAFT
  FAILED
  DELETED
}

// ── Product Media ──────────────────────────────────────────────────────────

model ProductMedia {
  id          String     @id @default(uuid())
  productId   String     @map("product_id")
  type        MediaType
  url         String
  cdnUrl      String?    @map("cdn_url")
  mimeType    String?    @map("mime_type")
  sizeBytes   Int?       @map("size_bytes")
  width       Int?
  height      Int?
  sortOrder   Int        @default(0) @map("sort_order")
  createdAt   DateTime   @default(now()) @map("created_at")

  product Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@map("product_media")
}

enum MediaType {
  ORIGINAL
  ENHANCED
  THUMBNAIL
  MODEL_3D
  AI_RENDER
}

// ── Events (Analytics) ────────────────────────────────────────────────────

model Event {
  id          String    @id @default(uuid())
  productId   String    @map("product_id")
  eventType   EventType @map("event_type")
  sessionId   String?   @map("session_id")
  userAgent   String?   @map("user_agent")
  country     String?   @db.VarChar(2)
  referer     String?
  createdAt   DateTime  @default(now()) @map("created_at")

  product Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@map("events")
}

enum EventType {
  PAGE_VIEW
  GALLERY_CLICK
  MODEL_INTERACT
  INQUIRY_OPEN
  INQUIRY_SUBMIT
  SHARE_CLICK
  CATALOG_CLICK
}

// ── Inquiries ─────────────────────────────────────────────────────────────

model Inquiry {
  id          String   @id @default(uuid())
  productId   String   @map("product_id")
  name        String
  phone       String?
  message     String?  @db.Text
  notified    Boolean  @default(false)
  createdAt   DateTime @default(now()) @map("created_at")

  product Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@map("inquiries")
}

// ── Jobs (audit log) ──────────────────────────────────────────────────────

model JobLog {
  id          String    @id @default(uuid())
  jobId       String    @map("job_id")
  productId   String?   @map("product_id")
  type        String
  status      String
  attempts    Int       @default(0)
  error       String?   @db.Text
  startedAt   DateTime? @map("started_at")
  completedAt DateTime? @map("completed_at")
  createdAt   DateTime  @default(now()) @map("created_at")

  @@map("job_logs")
}
```

### 6.2 Indexes

```sql
-- Performance indexes (run after migration)
CREATE INDEX idx_products_user_id ON products(user_id);
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_products_created ON products(created_at DESC);
CREATE INDEX idx_media_product_id ON product_media(product_id);
CREATE INDEX idx_events_product_id ON events(product_id);
CREATE INDEX idx_events_created ON events(created_at DESC);
CREATE INDEX idx_inquiries_product_id ON inquiries(product_id);
```

---

## 7. Telegram Bot Service

### 7.1 Bot Commands

| Command | Action |
|---|---|
| `/start` | Onboarding message + show template |
| `/help` | Show help and all commands |
| `/new` | Prompt user to send a new product |
| `/myproducts` | List last 10 published products with links |
| `/edit [slug]` | Enter edit mode for a product |
| `/delete [slug]` | Confirm and delete a product |
| `/catalog` | Send link to owner's catalog page |
| `/plan` | Show current plan and usage |

### 7.2 Message Handling Logic

```
Incoming Telegram message
        ↓
Is it a command? → route to command handler
        ↓
Does it contain text? → check if it matches product template pattern
        ↓
Does it contain photos? → check if there is a pending text submission
        ↓
Is it a reply to a bot message? → context-aware action (edit flow)
        ↓
Unknown message → send help prompt
```

### 7.3 New Product Message Format

The bot accepts free-form structured text. Claude handles the parsing, so exact casing and field order don't matter. Minimum required fields are **name** and at least **one image**.

```
Name: Luna Velvet Sofa
Price: 1200
Dimensions: 180x85x75 cm
Material: Velvet, oak legs
Color: Cream
Stock: in stock
Notes: Available in 3 colors
```

The bot must handle:
- Text and images sent in the same message
- Text sent first, images sent as a reply within 5 minutes
- Images sent first, text sent as a reply within 5 minutes
- Missing optional fields (price, dimensions) — bot asks once, then proceeds with what is available

### 7.4 Session State Machine

Each Telegram user has a session state stored in Redis:

```typescript
type BotSessionState =
  | { type: 'idle' }
  | { type: 'awaiting_images'; text: string; expiresAt: number }
  | { type: 'awaiting_text'; fileIds: string[]; expiresAt: number }
  | { type: 'confirming_delete'; productId: string }
  | { type: 'editing'; productId: string; field: string }
```

Session state TTL: 5 minutes. After expiry the bot resets to idle and informs the user.

### 7.5 Webhook Setup

```
POST https://api.telegram.org/bot{TOKEN}/setWebhook
  url: https://bot.furnishflow.com/webhook
  secret_token: {TELEGRAM_WEBHOOK_SECRET}
  allowed_updates: ["message", "callback_query"]
```

The webhook endpoint must:
1. Validate the `X-Telegram-Bot-Api-Secret-Token` header
2. Return HTTP 200 immediately (within 100ms)
3. Push the update to the job queue for async processing

### 7.6 Rate Limiting

- Per user: max 10 product submissions per hour
- Per user: max 50 bot messages per hour
- Global: max 500 webhook requests per minute
- Enforce using Upstash Redis sliding window

---

## 8. AI Processing Pipeline

### 8.1 Job Types

| Job Type | Trigger | Description |
|---|---|---|
| `product.parse` | New product submission | Claude parses text + images |
| `product.enhance_images` | After parse completes | Background removal + upscale |
| `product.generate_3d` | After image enhancement | Meshy.ai 3D generation |
| `product.generate_description` | After parse | Claude generates marketing copy |
| `product.publish` | After all above complete | Creates product page, notifies owner |

### 8.2 Text Parsing with Claude

```typescript
// apps/worker/src/ai/parseProduct.ts

const PARSE_SYSTEM_PROMPT = `
You are a product data extraction assistant for a furniture catalog platform.
Extract structured product information from the user's message.

Return ONLY valid JSON matching this schema exactly:
{
  "name": string,
  "price": number | null,
  "currency": string | null,         // ISO 4217 code, default "USD"
  "dimensions": {
    "w": number | null,
    "h": number | null, 
    "d": number | null,
    "unit": "cm" | "m" | "inch" | null
  } | null,
  "material": string | null,
  "color": string | null,
  "weight": number | null,
  "stockStatus": "IN_STOCK" | "OUT_OF_STOCK" | "MADE_TO_ORDER",
  "notes": string | null,
  "confidence": number              // 0.0–1.0, how confident you are in the extraction
}

Rules:
- If a field is not mentioned, set it to null
- Parse dimensions from formats like "180x85x75", "180 x 85 x 75 cm", "180/85/75"
- Parse prices from formats like "1200", "$1,200", "1.200", "AZN 500"
- Default stockStatus to IN_STOCK if not mentioned
- The name field is required — if missing set confidence below 0.5
`

async function parseProductText(
  text: string, 
  imageBase64Array: string[]
): Promise<ParsedProduct> {
  const content: MessageParam['content'] = [
    { type: 'text', text },
    ...imageBase64Array.map(img => ({
      type: 'image' as const,
      source: {
        type: 'base64' as const,
        media_type: 'image/jpeg' as const,
        data: img,
      },
    })),
    { type: 'text', text: 'Extract the product information from the text and images above.' },
  ]

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    system: PARSE_SYSTEM_PROMPT,
    messages: [{ role: 'user', content }],
  })

  const json = JSON.parse(response.content[0].text)
  return ParsedProductSchema.parse(json)
}
```

### 8.3 Description Generation with Claude

After data extraction, generate a marketing product description:

```typescript
const DESCRIPTION_PROMPT = `
Write a concise, professional 2–3 sentence product description for this furniture item.
Tone: warm, professional, suitable for a furniture catalog.
Do not invent specifications not provided. Do not use superlatives like "amazing" or "stunning".
Output only the description text, no quotes, no formatting.

Product data:
{PRODUCT_JSON}
`
```

### 8.4 Slug Generation

```typescript
function generateSlug(name: string, existingSlugs: string[]): string {
  const base = name
    .toLowerCase()
    .replace(/[^a-z0-9\s-]/g, '')
    .replace(/\s+/g, '-')
    .slice(0, 50)
  
  let slug = base
  let counter = 1
  while (existingSlugs.includes(slug)) {
    slug = `${base}-${counter++}`
  }
  return slug
}
```

---

## 9. Image Processing Service

### 9.1 Python FastAPI Service

The image service runs as a separate Railway service. It exposes two internal endpoints.

```python
# apps/image-service/app/main.py
from fastapi import FastAPI, UploadFile, Header, HTTPException
from .routes import remove_bg, upscale

app = FastAPI()

def verify_secret(x_internal_secret: str = Header(...)):
    if x_internal_secret != settings.IMAGE_SERVICE_SECRET:
        raise HTTPException(status_code=401)
```

### 9.2 Endpoints

**POST /remove-background**

- Input: image file (JPEG/PNG/WebP, max 20MB)
- Output: PNG with transparent background
- Library: `rembg` with `u2net` model
- Processing time target: < 3 seconds

**POST /upscale**

- Input: image file
- Output: 2× resolution JPEG, quality 85
- Library: `Real-ESRGAN` with `RealESRGAN_x2plus` model
- Processing time target: < 10 seconds
- Skip upscale if input is already > 2000px on longest side

### 9.3 Image Processing Pipeline (Node.js worker side)

For each input image:

```
1. Download from R2 (original)
         ↓
2. Send to /remove-background → PNG with no background
         ↓
3. Send to /upscale → 2× upscaled PNG
         ↓
4. Compress with sharp → WebP, quality 82, max 1600px wide
         ↓
5. Generate thumbnail → WebP, 400×400 cover crop
         ↓
6. Upload both to R2 under /enhanced/ and /thumbnails/
         ↓
7. Update product_media records
```

### 9.4 Image Constraints

| Type | Max size | Format | Dimensions |
|---|---|---|---|
| Original | 20 MB | JPEG/PNG/WebP/HEIC | Any |
| Enhanced | 2 MB | WebP | Max 1600px wide |
| Thumbnail | 100 KB | WebP | 400×400 (cover) |
| 3D model | 10 MB | .glb | N/A |

---

## 10. 3D Generation Service

### 10.1 Meshy.ai Integration

Meshy.ai provides image-to-3D generation. The process is asynchronous.

```typescript
// apps/worker/src/jobs/generate3D.ts

async function generate3DModel(imageUrl: string, productId: string) {
  // Step 1 — Submit task
  const createRes = await ky.post('https://api.meshy.ai/v1/image-to-3d', {
    headers: { Authorization: `Bearer ${env.MESHY_API_KEY}` },
    json: {
      image_url: imageUrl,
      enable_pbr: true,          // physically-based rendering
      ai_model: 'meshy-4',
    },
  }).json<{ result: string }>()

  const taskId = createRes.result

  // Step 2 — Poll for completion (max 10 min, poll every 15s)
  let model_urls: { glb: string } | null = null
  const deadline = Date.now() + 10 * 60 * 1000

  while (Date.now() < deadline) {
    await sleep(15_000)
    const statusRes = await ky.get(
      `https://api.meshy.ai/v1/image-to-3d/${taskId}`,
      { headers: { Authorization: `Bearer ${env.MESHY_API_KEY}` } }
    ).json<MeshyTaskStatus>()

    if (statusRes.status === 'SUCCEEDED') {
      model_urls = statusRes.model_urls
      break
    }
    if (statusRes.status === 'FAILED') {
      throw new Error(`Meshy task ${taskId} failed: ${statusRes.task_error?.message}`)
    }
  }

  if (!model_urls) throw new Error('3D generation timed out')

  // Step 3 — Download and re-upload to R2
  const glbBuffer = await ky.get(model_urls.glb).arrayBuffer()
  const r2Key = `models/${productId}/model.glb`
  await uploadToR2(r2Key, Buffer.from(glbBuffer), 'model/gltf-binary')

  return `${env.R2_PUBLIC_URL}/${r2Key}`
}
```

### 10.2 Fallback Strategy

If Meshy.ai fails or times out:
1. Log the failure, mark 3D generation as skipped for this product
2. Publish the product page without the 3D viewer (falls back to image gallery)
3. Retry 3D generation in the background once more after 30 minutes
4. If second attempt fails, mark product as `model_3d_unavailable` — product page stays live

---

## 11. Backend REST API

### 11.1 API Structure

Base URL: `https://api.furnishflow.com`

All authenticated routes require `Authorization: Bearer {JWT}` header.

### 11.2 Endpoints

#### Authentication

```
POST   /auth/telegram          Telegram Login Widget callback → issue JWT
POST   /auth/refresh           Refresh JWT using refresh token
POST   /auth/logout            Invalidate session
```

#### Products

```
GET    /products               List owner's products (paginated)
  Query: ?status=live&page=1&limit=20&sort=created_at&order=desc

GET    /products/:id           Get single product (owner only)

PATCH  /products/:id           Update product fields
  Body: { name?, price?, material?, color?, stockStatus?, description? }

DELETE /products/:id           Soft delete (status → DELETED)

POST   /products/:id/reprocess Trigger re-processing (re-run AI pipeline)
```

#### Public Product Data

```
GET    /public/products/:slug  Full product data for rendering (cached 60s)
GET    /public/catalog/:slug   All live products for a catalog (cached 30s)
```

#### Analytics

```
GET    /analytics              Overview stats (total views, inquiries, top products)
  Query: ?from=2026-01-01&to=2026-12-31

GET    /analytics/products/:id Per-product analytics
  Query: ?period=7d|30d|90d

POST   /events                 Track event (public, no auth)
  Body: { productId, eventType, sessionId? }
```

#### Inquiries

```
GET    /inquiries              List inquiries for owner's products
  Query: ?productId=&page=1&limit=20

POST   /public/inquiries       Submit inquiry (public, no auth)
  Body: { productId, name, phone?, message? }
```

#### User / Shop

```
GET    /user/me                Get current user profile
PATCH  /user/me                Update shop name, description, contact info
POST   /user/me/logo           Upload shop logo
```

#### Billing

```
GET    /billing/portal         Create Stripe billing portal session → redirect URL
POST   /billing/webhook        Stripe webhook handler (no auth, validated by signature)
```

### 11.3 Response Format

All responses follow a consistent envelope:

```typescript
// Success
{
  "success": true,
  "data": { ... }
}

// Paginated
{
  "success": true,
  "data": [...],
  "meta": {
    "total": 142,
    "page": 1,
    "limit": 20,
    "totalPages": 8
  }
}

// Error
{
  "success": false,
  "error": {
    "code": "PRODUCT_NOT_FOUND",
    "message": "No product found with that ID"
  }
}
```

### 11.4 Error Codes

| Code | HTTP Status | Meaning |
|---|---|---|
| `UNAUTHORIZED` | 401 | Missing or invalid JWT |
| `FORBIDDEN` | 403 | Authenticated but no access to resource |
| `NOT_FOUND` | 404 | Resource does not exist |
| `VALIDATION_ERROR` | 422 | Request body failed validation |
| `RATE_LIMITED` | 429 | Too many requests |
| `PLAN_LIMIT` | 402 | Action exceeds current plan limits |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## 12. Authentication

### 12.1 Telegram Login Widget Flow

```
1. User visits dashboard → sees "Login with Telegram" button
2. Telegram widget redirects to /api/auth/telegram callback with:
   { id, first_name, last_name, username, hash, auth_date }
3. Server validates HMAC-SHA256 hash using bot token
4. Server upserts user in DB (creates if new)
5. Server issues JWT (7d expiry) + refresh token (30d) in httpOnly cookies
6. User is redirected to dashboard
```

### 12.2 JWT Payload

```typescript
interface JWTPayload {
  sub: string        // user.id (UUID)
  tgId: string       // user.telegramId
  plan: Plan         // current plan
  iat: number
  exp: number
}
```

### 12.3 Plan Limits Enforcement

The API enforces plan limits on product creation:

| Plan | Max Products | 3D Generation | Custom Domain |
|---|---|---|---|
| STARTER | 30 | No | No |
| PROFESSIONAL | 150 | Yes | Yes |
| BUSINESS | Unlimited | Yes (priority) | Yes |

---

## 13. Frontend — Product Pages

### 13.1 Product Page (`/p/[slug]`)

Built with Next.js SSG. Re-generated on demand via ISR when product is updated.

**Page sections (top to bottom):**

1. **Hero** — 3D model viewer (`@google/model-viewer`) or fallback image carousel
2. **Product header** — Name, price, stock badge, share button
3. **Image gallery** — Horizontal scroll strip of enhanced photos
4. **Description** — AI-generated marketing copy
5. **Spec table** — Dimensions, material, color, weight, care instructions
6. **Inquiry form** — Name, phone (optional), message → POST /public/inquiries
7. **Shop footer** — Shop name, logo, location, "See full catalog" link

**SEO requirements:**
- `<title>`: `{Product Name} — {Shop Name} | FurnishFlow`
- `<meta name="description">`: First sentence of AI-generated description
- Open Graph: `og:title`, `og:description`, `og:image` (first enhanced photo), `og:url`
- Structured data: `Product` schema (schema.org) with price, availability, image

### 13.2 3D Viewer Implementation

```tsx
// components/product/ProductViewer.tsx
'use client'

// @google/model-viewer is a web component — import the script
import '@google/model-viewer'

interface ProductViewerProps {
  modelUrl: string      // .glb file URL
  posterUrl: string     // fallback image while loading
}

export function ProductViewer({ modelUrl, posterUrl }: ProductViewerProps) {
  return (
    <model-viewer
      src={modelUrl}
      poster={posterUrl}
      alt="3D product view"
      camera-controls
      auto-rotate
      ar
      ar-modes="webxr scene-viewer quick-look"
      shadow-intensity="1"
      style={{ width: '100%', height: '400px' }}
    />
  )
}
```

### 13.3 Catalog Page (`/c/[slug]`)

Grid of product cards for all live products from one shop.

- Card shows: thumbnail, name, price, material
- Click → navigates to product page
- Branded header with shop name, logo, description
- Filter by: all / in stock / made to order
- Sort by: newest / price low to high / price high to low
- SEO: indexed, shop name in title

---

## 14. Frontend — Owner Dashboard

### 14.1 Dashboard Routes

```
/dashboard                    Redirect to /dashboard/products
/dashboard/products           Product list with status, views, quick actions
/dashboard/products/[id]      Product editor
/dashboard/analytics          Analytics charts and summary
/dashboard/settings           Shop profile, branding, catalog slug
/dashboard/billing            Plan info, upgrade, billing portal link
```

### 14.2 Product List Requirements

- Table with columns: thumbnail, name, status badge, views, created date, actions
- Status badges: Processing (yellow), Live (green), Draft (gray), Failed (red)
- Quick actions per row: View page, Edit, Toggle visibility, Delete
- Pagination: 20 per page
- Search: filter by name (client-side)
- Sort: created date, views, name
- Real-time status updates via Supabase Realtime (so "Processing" updates to "Live" without refresh)

### 14.3 Product Editor Requirements

- Edit all text fields inline
- Re-order images via drag and drop
- Delete individual images
- Upload additional images (max 8 total)
- Preview what the product page looks like (sidebar preview or link to live page)
- Re-trigger 3D generation if it failed
- Save changes → PATCH /products/:id → triggers ISR revalidation

### 14.4 Analytics Dashboard Requirements

- Summary cards: total views (7d), total inquiries (7d), most viewed product
- Line chart: daily page views over last 30 days (per product or total)
- Bar chart: top 10 products by views
- Table: recent inquiries with product name, customer name, phone, timestamp
- Date range picker: last 7d / 30d / 90d / custom

---

## 15. File Storage

### 15.1 R2 Bucket Structure

```
furnishflow-media/
├── originals/
│   └── {userId}/{productId}/{filename}.jpg
├── enhanced/
│   └── {userId}/{productId}/{filename}.webp
├── thumbnails/
│   └── {userId}/{productId}/{filename}.webp
├── models/
│   └── {productId}/model.glb
└── logos/
    └── {userId}/logo.webp
```

### 15.2 Upload Helper

```typescript
// packages/storage/src/r2.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3'

const r2 = new S3Client({
  region: 'auto',
  endpoint: `https://${env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: env.R2_ACCESS_KEY_ID,
    secretAccessKey: env.R2_SECRET_ACCESS_KEY,
  },
})

export async function uploadToR2(
  key: string,
  body: Buffer,
  contentType: string
): Promise<string> {
  await r2.send(new PutObjectCommand({
    Bucket: env.R2_BUCKET_NAME,
    Key: key,
    Body: body,
    ContentType: contentType,
    CacheControl: 'public, max-age=31536000, immutable',
  }))
  return `${env.R2_PUBLIC_URL}/${key}`
}
```

### 15.3 File Naming

All file names use a deterministic pattern. Never trust original filenames from users (security risk). Use UUIDs or hash-based names.

```typescript
const key = `enhanced/${userId}/${productId}/${randomUUID()}.webp`
```

---

## 16. Job Queue System

### 16.1 Queue Names

```
furnishflow:product:parse           High priority — user is waiting
furnishflow:product:images          Medium priority
furnishflow:product:3d              Low priority — can wait
furnishflow:product:description     Medium priority
furnishflow:product:publish         High priority — final step
furnishflow:notifications           High priority
```

### 16.2 Job Data Structure

```typescript
interface ProductJob {
  jobId: string
  productId: string
  userId: string
  telegramChatId: number
  attempt: number
  createdAt: string
}

interface ParseJob extends ProductJob {
  rawText: string
  imageR2Keys: string[]
}

interface Generate3DJob extends ProductJob {
  imageUrl: string
}
```

### 16.3 Retry Policy

| Queue | Max retries | Backoff | Dead-letter |
|---|---|---|---|
| product:parse | 3 | exponential 2s | Yes |
| product:images | 3 | exponential 5s | Yes |
| product:3d | 2 | fixed 30s | Yes |
| product:publish | 5 | exponential 1s | Yes |

On dead-letter: mark product as FAILED, notify owner via bot with error message and retry option.

### 16.4 Job Dependencies (Pipeline Orchestration)

```typescript
// Worker orchestrates the pipeline by chaining jobs
// Each job, on success, enqueues the next job

parseJob.on('completed', (result) => {
  imageQueue.add('enhance', { productId, imageKeys })
  descriptionQueue.add('generate', { productId, parsedData })
})

imageJob.on('completed', () => {
  threeDQueue.add('generate', { productId, bestImageUrl })
})

// Publish waits for both images and description (but not necessarily 3D)
// Uses a completion counter stored in Redis
```

---

## 17. Notification System

### 17.1 Owner Notifications via Telegram

All owner notifications are sent via the Telegram Bot API using `sendMessage`.

| Event | Message |
|---|---|
| Product received | "Got it! Processing {name}..." |
| Processing complete | "Your product is live! {url}" |
| Processing failed | "Something went wrong with {name}. Reply /retry_{id} to try again." |
| New inquiry | "New inquiry for {product name}: {customer name} — {phone}" |
| Plan about to expire | "Your plan expires in 3 days. Renew here: {billing_url}" |

### 17.2 Inquiry Notification Format

```typescript
async function notifyInquiry(inquiry: Inquiry, product: Product, owner: User) {
  const text = [
    `📩 New inquiry for *${escapeMarkdown(product.name)}*`,
    ``,
    `Name: ${inquiry.name}`,
    inquiry.phone ? `Phone: ${inquiry.phone}` : '',
    inquiry.message ? `Message: ${inquiry.message}` : '',
    ``,
    `Product: ${env.APP_URL}/p/${product.slug}`,
  ].filter(Boolean).join('\n')

  await bot.sendMessage(owner.telegramId.toString(), text, {
    parse_mode: 'MarkdownV2',
  })
}
```

---

## 18. Analytics

### 18.1 Event Tracking

Events are tracked by a lightweight public endpoint. No user PII is stored. Session IDs are anonymous UUIDs stored in localStorage on the product page.

```typescript
// Client-side tracking (product pages)
async function trackEvent(eventType: EventType, productId: string) {
  const sessionId = getOrCreateSessionId()
  await fetch('/api/events', {
    method: 'POST',
    body: JSON.stringify({ productId, eventType, sessionId }),
  })
}
```

Track on product page load:
- `PAGE_VIEW` on mount
- `GALLERY_CLICK` on image click
- `MODEL_INTERACT` on first 3D model interaction
- `INQUIRY_OPEN` on inquiry form open
- `INQUIRY_SUBMIT` on successful inquiry submission
- `SHARE_CLICK` on share button click

### 18.2 Analytics Aggregation Queries

```sql
-- Daily views for a product over last 30 days
SELECT
  DATE(created_at) AS day,
  COUNT(*) AS views
FROM events
WHERE product_id = $1
  AND event_type = 'PAGE_VIEW'
  AND created_at > NOW() - INTERVAL '30 days'
GROUP BY DATE(created_at)
ORDER BY day;

-- Top products by views
SELECT
  p.id, p.name, p.slug,
  COUNT(e.id) AS view_count
FROM products p
LEFT JOIN events e ON e.product_id = p.id AND e.event_type = 'PAGE_VIEW'
WHERE p.user_id = $1
  AND p.status = 'LIVE'
GROUP BY p.id
ORDER BY view_count DESC
LIMIT 10;
```

---

## 19. Subscription & Billing

### 19.1 Stripe Products

Create three Stripe Products with monthly and annual prices:

| Plan | Monthly | Annual |
|---|---|---|
| Starter | $29/mo | $290/yr |
| Professional | $79/mo | $790/yr |
| Business | $199/mo | $1,990/yr |

### 19.2 Webhook Events to Handle

```
customer.subscription.created   → update user plan + expires_at
customer.subscription.updated   → update user plan
customer.subscription.deleted   → downgrade user to free/suspended
invoice.payment_succeeded        → extend plan_expires_at
invoice.payment_failed           → notify user via Telegram, keep plan active for 3 days
```

### 19.3 Plan Enforcement Middleware

```typescript
// Middleware on product creation endpoint
async function enforcePlanLimits(userId: string) {
  const user = await db.user.findUnique({ where: { id: userId } })
  const productCount = await db.product.count({
    where: { userId, status: { not: 'DELETED' } }
  })
  
  const limits = { STARTER: 30, PROFESSIONAL: 150, BUSINESS: Infinity }
  if (productCount >= limits[user.plan]) {
    throw new ApiError('PLAN_LIMIT', 'Upgrade your plan to add more products')
  }
}
```

---

## 20. Security Requirements

### 20.1 Input Validation

- All incoming request bodies validated with Zod schemas
- All URL parameters validated and sanitized
- File uploads: validate MIME type by reading magic bytes, not just extension
- Maximum file sizes enforced at the API gateway level
- Telegram webhook: always verify `X-Telegram-Bot-Api-Secret-Token` header

### 20.2 Authentication & Authorization

- JWTs signed with RS256 (asymmetric) — not HS256
- JWTs expire in 7 days, refresh tokens in 30 days
- Refresh tokens stored in DB — can be revoked
- All owner-facing API routes check that the resource belongs to the authenticated user
- Row Level Security (RLS) enabled on Supabase tables as an extra safety layer

### 20.3 API Security

- CORS: allow only `furnishflow.com` and `*.furnishflow.com`
- Rate limiting per IP and per user (via Upstash Redis)
- Helmet.js: set all security headers (CSP, HSTS, X-Frame-Options, etc.)
- No stack traces in production error responses
- SQL injection: prevented by Prisma parameterized queries
- XSS: all user content escaped on render

### 20.4 File Security

- File uploads scanned for MIME type mismatch
- Original filenames never used — replaced with UUID-based names
- R2 bucket is private — only CDN-proxied public URLs served
- Presigned URLs used for any direct uploads (max 15 min expiry)
- Never serve user-uploaded files from the API domain (MIME confusion attacks)

### 20.5 Secrets Management

- No secrets in code or git history
- All secrets in Railway environment variables
- Rotate Telegram webhook secret monthly
- Use separate API keys per environment (dev/staging/prod)

---

## 21. Performance Requirements

### 21.1 Targets

| Metric | Target |
|---|---|
| Product page LCP | < 1.5s (Vercel edge + CDN) |
| Dashboard initial load | < 2s |
| API p95 response time | < 200ms |
| Product submission → bot confirmation | < 1 second |
| Full processing pipeline | < 3 minutes |
| 3D model load time on product page | < 5s (progressive loading with poster) |

### 21.2 Caching Strategy

| Data | Cache layer | TTL |
|---|---|---|
| Product page HTML | Vercel Edge CDN | 60s, ISR on update |
| Catalog page HTML | Vercel Edge CDN | 30s |
| Product data (API) | Redis | 30s |
| Analytics aggregates | Redis | 5 minutes |
| R2 media files | Cloudflare CDN | 1 year (immutable) |

### 21.3 Next.js ISR

When a product is updated via the dashboard, the API triggers ISR revalidation:

```typescript
// After PATCH /products/:id
await fetch(`${env.APP_URL}/api/revalidate`, {
  method: 'POST',
  headers: { 'x-revalidate-secret': env.REVALIDATE_SECRET },
  body: JSON.stringify({ paths: [`/p/${product.slug}`] }),
})
```

---

## 22. Error Handling & Logging

### 22.1 Logging Strategy

All services log structured JSON to stdout. Collected by Better Stack (Logtail).

```typescript
// Log format
{
  "level": "info" | "warn" | "error",
  "service": "api" | "bot" | "worker",
  "timestamp": "2026-05-08T10:00:00Z",
  "requestId": "uuid",
  "userId": "uuid",
  "message": "...",
  "data": { ... }      // additional context, never log secrets or PII
}
```

### 22.2 Log Levels

- `error`: unhandled exceptions, job failures, external API errors
- `warn`: rate limit hits, plan limit approaching, slow queries
- `info`: job started/completed, product published, user created
- `debug`: disabled in production

### 22.3 Alerting Rules (Better Stack)

- Alert if API error rate > 1% over 5 minutes
- Alert if worker job failure rate > 5% over 10 minutes
- Alert if Meshy.ai API error rate > 10% over 15 minutes
- Alert if any service is down for > 2 minutes

### 22.4 Error Recovery

- All job failures are logged to `job_logs` table with full error context
- Failed products can be manually retried from the owner's dashboard
- The bot proactively notifies the owner when a product fails to process

---

## 23. Testing Requirements

### 23.1 Test Types

| Type | Tool | Coverage target |
|---|---|---|
| Unit tests | Vitest | Core business logic, parsers, utils |
| Integration tests | Vitest + test DB | API routes, DB operations |
| E2E tests | Playwright | Critical flows: bot submission, product page, dashboard |

### 23.2 Critical Test Cases

**Bot parsing:**
- Parse complete product message → correct JSON
- Parse message with missing optional fields → correct nulls
- Parse message in Russian, Azerbaijani, Arabic → correct extraction
- Parse message with non-standard dimension formats
- Reject message with no name and confidence < 0.5

**API:**
- Unauthenticated requests → 401
- Access another owner's product → 403
- Create product over plan limit → 402
- Valid product creation → 201 with correct data

**Product pages:**
- Product with 3D model renders correctly
- Product without 3D model falls back to image gallery
- Inquiry form submits and triggers notification
- SEO meta tags present and correct

### 23.3 Test Database

Use a separate Supabase project for testing. Run Prisma migrations before test suite. Clean up with `prisma db push --force-reset` between test runs (CI only).

---

## 24. Deployment & CI/CD

### 24.1 Environments

| Environment | Purpose | Auto-deploy |
|---|---|---|
| `development` | Local dev with Docker Compose | No |
| `staging` | Preview, integration testing | Yes (on PR) |
| `production` | Live app | Yes (on merge to main) |

### 24.2 GitHub Actions Pipeline

```yaml
# .github/workflows/deploy.yml

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm lint

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
      redis:
        image: redis:7
    steps:
      - uses: actions/checkout@v4
      - run: pnpm test:ci

  deploy-staging:
    needs: [lint, test]
    if: github.event_name == 'pull_request'
    steps:
      - run: railway deploy --environment staging

  deploy-production:
    needs: [lint, test]
    if: github.ref == 'refs/heads/main'
    steps:
      - run: railway deploy --environment production
      - run: vercel deploy --prod
```

### 24.3 Database Migrations

- Migrations run automatically as part of the deploy pipeline before the new code goes live
- `prisma migrate deploy` (not `push`) in staging and production
- Never use `prisma db push` in production
- All migrations must be backwards-compatible (no breaking schema changes in a single deploy)

### 24.4 Local Development

```bash
# Setup
pnpm install
cp .env.example .env.local   # Fill in all variables

# Start all services with Docker Compose
docker compose up -d          # PostgreSQL, Redis, image-service

# Run migrations
pnpm db:migrate

# Start all Node.js services
pnpm dev                      # Turborepo runs all dev servers in parallel
```

---

## 25. Phase-by-Phase Build Plan

### Phase 1 — Weeks 1–4: Working MVP

**Goal:** Owner sends text + images → gets live product page link. No 3D, basic design.

**Week 1**
- [ ] Set up monorepo (pnpm + Turborepo)
- [ ] Configure Railway, Vercel, Supabase, Cloudflare R2, Upstash
- [ ] Register Telegram bot, implement webhook handler
- [ ] Implement bot session state machine (idle / awaiting_images / awaiting_text)
- [ ] Implement `/start` and free-form product message parsing (bot side)

**Week 2**
- [ ] Integrate Claude API — text + image parsing → structured JSON
- [ ] Implement Prisma schema, run first migration
- [ ] Upload images from Telegram to R2
- [ ] Create product record in DB
- [ ] Basic Next.js product page (text-only, no images, no 3D)

**Week 3**
- [ ] Connect product page to real product data
- [ ] Display product images on page
- [ ] Bot sends back live link on completion
- [ ] Implement basic error handling and retry
- [ ] Deploy to staging

**Week 4**
- [ ] Test end-to-end with 5 real furniture shop owners
- [ ] Fix all bugs found in real usage
- [ ] Implement `/myproducts` and `/delete` bot commands
- [ ] Basic product page design (not polished, but readable)
- [ ] Deploy to production (invite-only beta)

---

### Phase 2 — Weeks 5–8: 3D & Quality

**Goal:** Add background removal, image enhancement, 3D generation, and polished product pages.

- [ ] Build Python image-service (FastAPI, rembg, ESRGAN)
- [ ] Integrate image service into worker pipeline
- [ ] Integrate Meshy.ai 3D generation
- [ ] Upload .glb to R2
- [ ] Add `@google/model-viewer` to product page
- [ ] Add image gallery to product page
- [ ] Generate AI product descriptions
- [ ] Auto-generate SEO meta tags
- [ ] Design polished product page (Tailwind, responsive, mobile-first)
- [ ] Add inquiry form to product page
- [ ] Implement Telegram Login Widget
- [ ] Basic dashboard (product list only, no edit)

---

### Phase 3 — Weeks 9–12: Dashboard & Billing

**Goal:** Owners can manage their catalog. Stripe billing goes live.

- [ ] Full product editor (all fields, image management, re-order)
- [ ] Catalog page (`/c/[slug]`)
- [ ] Analytics dashboard (views, inquiries, charts)
- [ ] Inquiry notification via Telegram
- [ ] Shop settings (name, logo, description, catalog slug)
- [ ] Stripe integration (3 plans, billing portal)
- [ ] Plan enforcement (product limits, 3D generation gating)
- [ ] ISR revalidation on product update
- [ ] `/plan` and `/catalog` bot commands
- [ ] Onboarding flow for new users

---

### Phase 4 — Month 4–6: Scale

**Goal:** Grow to paid customers. Add WhatsApp. Custom domains. Multilingual.

- [ ] WhatsApp Business API as second input channel
- [ ] Custom domain support for catalog pages
- [ ] Multilingual product descriptions (Azerbaijani, Russian, Turkish, Arabic)
- [ ] Bulk import via CSV + image ZIP
- [ ] AR viewer (WebXR via `model-viewer` AR mode)
- [ ] Referral program
- [ ] Admin dashboard (internal, for support/ops)
- [ ] Automated email receipts and plan renewal reminders

---

*End of technical requirements document.*
*Next step: begin Phase 1, Week 1 — monorepo setup and Telegram webhook.*
