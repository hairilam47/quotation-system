# Quotation Management System — Production-Ready Build Spec

A complete, opinionated implementation guide for an IT services quotation system. Stack: Next.js 14 App Router · TypeScript · Tailwind · shadcn/ui · Postgres · Prisma · NextAuth · `@react-pdf/renderer` · Vercel Blob.

---

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       Next.js 14 (App Router)                    │
│                                                                   │
│  ┌──────────────────────┐      ┌─────────────────────────────┐  │
│  │   /(admin)/*  RSC    │      │  /api/quotations/[id]/pdf   │  │
│  │   Dashboard, forms   │      │  Route Handler → PDF stream │  │
│  │   shadcn/ui + RHF    │      │                             │  │
│  └──────────┬───────────┘      └──────────────┬──────────────┘  │
│             │                                  │                  │
│             ▼                                  ▼                  │
│  ┌──────────────────────┐      ┌─────────────────────────────┐  │
│  │  Server Actions      │      │  @react-pdf/renderer        │  │
│  │  /server/actions/*   │      │  Modern + Classic templates │  │
│  └──────────┬───────────┘      └──────────────┬──────────────┘  │
│             │                                  │                  │
│             └──────────────┬───────────────────┘                 │
│                            ▼                                      │
│                  ┌──────────────────┐                            │
│                  │  Prisma Client   │                            │
│                  └─────────┬────────┘                            │
└────────────────────────────┼─────────────────────────────────────┘
                             ▼
                  ┌──────────────────┐    ┌──────────────────┐
                  │   PostgreSQL     │    │   Vercel Blob    │
                  │  (Neon/Supabase) │    │  (logo upload)   │
                  └──────────────────┘    └──────────────────┘
```

**Key decisions and why:**

- **Server Actions for mutations, Route Handlers for the PDF endpoint.** Server Actions give you end-to-end type safety without tRPC overhead in App Router. The PDF endpoint stays a Route Handler because it returns a binary stream with custom headers (`Content-Disposition`, `Content-Type: application/pdf`).
- **`@react-pdf/renderer` over Puppeteer.** No Chromium, no `@sparticuz/chromium`, smaller bundle, faster cold starts on Vercel, and crucially: native `<Link>` component produces real PDF hyperlink annotations. Puppeteer's print-to-PDF is inconsistent for clickable links and bloats the deployment.
- **Snapshotting client + company data on the quotation.** Once a quote is sent, the client's address or your company logo might change. The PDF should always reproduce what the client saw. We store JSON snapshots on the quotation row.
- **Decimal columns, never floats.** Money math on floats is a bug factory. Prisma's `Decimal` maps to Postgres `numeric`.

---

## 2. Project Structure

```
quotation-manager/
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── public/
├── src/
│   ├── app/
│   │   ├── (admin)/
│   │   │   ├── layout.tsx                  # auth gate + sidebar
│   │   │   ├── dashboard/page.tsx
│   │   │   ├── clients/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [id]/edit/page.tsx
│   │   │   ├── quotations/
│   │   │   │   ├── page.tsx                # list + filters
│   │   │   │   ├── new/page.tsx
│   │   │   │   └── [id]/
│   │   │   │       ├── page.tsx            # detail + preview
│   │   │   │       └── edit/page.tsx
│   │   │   └── settings/page.tsx           # company profile + logo
│   │   ├── api/
│   │   │   ├── auth/[...nextauth]/route.ts
│   │   │   └── quotations/[id]/pdf/route.ts
│   │   ├── login/page.tsx
│   │   ├── layout.tsx
│   │   └── globals.css
│   ├── components/
│   │   ├── ui/                             # shadcn primitives
│   │   ├── quotation-form.tsx
│   │   ├── line-items-editor.tsx
│   │   ├── client-picker.tsx
│   │   └── status-badge.tsx
│   ├── lib/
│   │   ├── db.ts                           # Prisma singleton
│   │   ├── auth.ts                         # NextAuth config
│   │   ├── blob.ts                         # logo upload helper
│   │   ├── format.ts                       # currency, dates
│   │   ├── validation.ts                   # Zod schemas
│   │   └── pdf/
│   │       ├── render.ts                   # entrypoint: data → Buffer
│   │       ├── calculations.ts             # totals, tax, discount
│   │       ├── qrcode.ts
│   │       └── templates/
│   │           ├── types.ts                # shared template props
│   │           ├── modern.tsx
│   │           └── classic.tsx
│   └── server/
│       └── actions/
│           ├── quotations.ts
│           ├── clients.ts
│           └── settings.ts
├── .env.example
├── next.config.mjs
├── package.json
├── tailwind.config.ts
└── tsconfig.json
```

---

## 3. Feature List & User Stories

### Core features

- **Authentication.** Single-admin or multi-admin via NextAuth credentials provider. All `/(admin)/*` routes gated.
- **Company profile / Settings.** Singleton record: name, address, contact, website, tax number, logo (uploaded), default tax rate, default T&Cs, default payment URL, default template, currency.
- **Clients.** CRUD. Email + name required. Reusable across quotations.
- **Quotations.** CRUD + duplicate. Auto-generated quote number (`Q-2026-0001`). Status workflow. Line items inline. Discount (% or fixed) and tax. Per-quote payment URL override. Per-quote template choice.
- **PDF generation.** Server-side, two templates (Modern, Classic), embedded logo, clickable Pay Now button via real PDF link annotation, optional QR code beside it.
- **Status tracking.** `DRAFT → SENT → ACCEPTED → PAID` (plus `REJECTED`, `EXPIRED`). Timestamps captured per transition.
- **Dashboard.** Counts by status, recent quotations, total outstanding.

### User stories

| As a... | I want to... | So that... |
|---|---|---|
| Procurement manager | log in to a secure dashboard | Only I can access pricing data |
| Procurement manager | save my company logo and details once | They appear on every quote without re-entry |
| Procurement manager | create a quote with an itemised service list | I can bill clients accurately for hours/days/fixed work |
| Procurement manager | apply a discount and tax | The total reflects negotiated terms |
| Procurement manager | paste a Stripe/PayPal link per quote | The client can pay directly from the PDF |
| Procurement manager | choose Modern or Classic template per quote | Tone matches the client (startup vs. enterprise) |
| Procurement manager | duplicate a previous quote | I don't re-type recurring engagements |
| Procurement manager | mark a quote as Sent / Accepted / Paid | I can track my pipeline |
| Procurement manager | download a polished A4 PDF with a clickable Pay Now button and QR code | Clients can pay from desktop or by scanning on mobile |
| Procurement manager | see a list of quotes filterable by client and status | I can chase outstanding ones |

### Non-functional

- Type safety end-to-end (Zod schemas shared between server actions and forms).
- Server-side input validation on every mutation (never trust the client).
- Decimal money arithmetic (no floats).
- A4 PDF with consistent margins; no content overflow at any line-item count.
- Responsive admin UI (collapses to single column at `<md`).
- Idempotent PDF rendering (same input → identical PDF).

---

## 4. Database Schema (Prisma)

`prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum QuotationStatus {
  DRAFT
  SENT
  ACCEPTED
  REJECTED
  PAID
  EXPIRED
}

enum DiscountType {
  PERCENTAGE
  FIXED
}

enum TemplateStyle {
  MODERN
  CLASSIC
}

model User {
  id           String   @id @default(cuid())
  email        String   @unique
  name         String?
  passwordHash String
  role         String   @default("ADMIN")
  createdAt    DateTime @default(now())
}

/// Singleton — enforce at app level (only one row).
model CompanySettings {
  id                String        @id @default(cuid())
  name              String
  addressLine1      String
  addressLine2      String?
  city              String
  region            String?
  postalCode        String
  country           String
  phone             String?
  email             String
  website           String?
  taxNumber         String?
  logoUrl           String?
  currency          String        @default("USD")
  defaultTaxRate    Decimal       @default(0) @db.Decimal(5, 2)
  defaultTerms      String?       @db.Text
  defaultNotes      String?       @db.Text
  defaultTemplate   TemplateStyle @default(MODERN)
  defaultPaymentUrl String?
  updatedAt         DateTime      @updatedAt
}

model Client {
  id           String      @id @default(cuid())
  name         String
  company      String?
  email        String
  phone        String?
  addressLine1 String?
  addressLine2 String?
  city         String?
  region       String?
  postalCode   String?
  country      String?
  notes        String?     @db.Text
  createdAt    DateTime    @default(now())
  updatedAt    DateTime    @updatedAt
  quotations   Quotation[]

  @@index([email])
}

model Quotation {
  id     String          @id @default(cuid())
  number String          @unique // e.g. Q-2026-0001
  status QuotationStatus @default(DRAFT)

  issueDate  DateTime @default(now())
  validUntil DateTime

  clientId String
  client   Client @relation(fields: [clientId], references: [id], onDelete: Restrict)

  /// Frozen copies for stable PDF reproduction after send
  clientSnapshot  Json?
  companySnapshot Json?

  currency String @default("USD")

  discountType  DiscountType?
  discountValue Decimal       @default(0) @db.Decimal(12, 2)
  taxRate       Decimal       @default(0) @db.Decimal(5, 2)

  // Stored totals — recomputed on every save, persisted for fast list views
  subtotal       Decimal @default(0) @db.Decimal(12, 2)
  discountAmount Decimal @default(0) @db.Decimal(12, 2)
  taxAmount      Decimal @default(0) @db.Decimal(12, 2)
  total          Decimal @default(0) @db.Decimal(12, 2)

  notes String? @db.Text
  terms String? @db.Text

  paymentUrl String?
  showQrCode Boolean @default(true)

  template TemplateStyle @default(MODERN)

  lineItems LineItem[]

  createdAt  DateTime  @default(now())
  updatedAt  DateTime  @updatedAt
  sentAt     DateTime?
  acceptedAt DateTime?
  paidAt     DateTime?

  @@index([clientId])
  @@index([status])
  @@index([number])
}

model LineItem {
  id          String    @id @default(cuid())
  quotationId String
  quotation   Quotation @relation(fields: [quotationId], references: [id], onDelete: Cascade)
  description String    @db.Text
  quantity    Decimal   @db.Decimal(10, 2)
  unit        String    @default("hours") // hours | days | fixed
  unitPrice   Decimal   @db.Decimal(12, 2)
  lineTotal   Decimal   @db.Decimal(12, 2)
  position    Int       @default(0)

  @@index([quotationId])
}
```

A note on the snapshots: the first time a quote leaves `DRAFT` (transitions to `SENT`), the server action copies the current client + company settings into `clientSnapshot` / `companySnapshot`. PDF generation always prefers the snapshot when present, so a sent quote renders identically forever even if you later rename your company or update the client's address.

---

## 5. PDF Generation Strategy

### Why `@react-pdf/renderer`

- **Real PDF link annotations.** Wrap any node in `<Link src="https://...">` and you get a true clickable region in the PDF (works in Acrobat, Preview, Chrome's PDF viewer, mobile). Puppeteer's print-to-PDF often produces flat raster regions or mishandles styled buttons.
- **Server-rendered to Buffer.** `renderToBuffer(<Doc/>)` works fine in a Vercel serverless function. No browser binary, no temp filesystem.
- **Deterministic.** Same JSON in → byte-identical PDF out.

### Layout strategy

A4 is `595.28 × 841.89` points. We use:

- **Page padding:** `40pt` on all sides (≈14mm).
- **Header (`fixed`):** logo + company block, repeats on overflow pages.
- **Footer (`fixed`):** page numbers via the `render` prop + a thin branding line.
- **Body:** flex column with `flexGrow: 1` so the totals block sits at sensible vertical position even on short quotes.
- **Line items table:** a `View` with `wrap` enabled (default), each row is `wrap={false}` so a single row never splits across pages.
- **Pay Now block:** `wrap={false}` so the button + QR never split.

### Logos

Logos can be:

1. **Vercel Blob URL** (recommended): you upload a PNG/SVG/JPG once via the Settings page. The URL goes in `companySettings.logoUrl`. At PDF render time, `@react-pdf/renderer` fetches it server-side and embeds it.
2. **Pre-fetched buffer:** for ultra-determinism, fetch the logo once in the Route Handler and pass it as a base64 data URL or a Node `Buffer`. This avoids any network call during PDF assembly.

I'll use option 2 in the example — it's only a few extra lines and removes a flaky dependency.

### Hyperlinks (Pay Now button)

```tsx
<Link src={paymentUrl}>
  <View style={styles.payButton}>
    <Text style={styles.payButtonText}>Click here to pay online →</Text>
  </View>
</Link>
```

That's it. Anywhere the user taps inside the rendered button rectangle in the PDF triggers the URL.

### QR codes

The `qrcode` npm package renders to PNG data URL server-side. We embed it via `<Image src={dataUrl} />`. We render the QR only when `paymentUrl` is set and `showQrCode` is true.

### Multiple templates

Both `modern.tsx` and `classic.tsx` export a default React component receiving the same `TemplateProps`. A registry in `render.ts` picks one based on `quotation.template`.

---

## 6. Implementation Guide

### 6.1 Setup

Initialize the project:

```bash
npx create-next-app@latest quotation-manager \
  --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
cd quotation-manager

# Core deps
npm install @prisma/client prisma zod react-hook-form @hookform/resolvers \
            @react-pdf/renderer qrcode date-fns next-auth bcryptjs \
            @vercel/blob

# UI
npm install class-variance-authority clsx tailwind-merge lucide-react
npx shadcn@latest init -d
npx shadcn@latest add button input textarea label select form table \
                       card badge dialog dropdown-menu toast separator

# Types
npm install -D @types/bcryptjs @types/qrcode

# Init Prisma
npx prisma init --datasource-provider postgresql
```

`.env.example`:

```bash
# Database
DATABASE_URL="postgresql://user:pass@host:5432/quotation_manager?sslmode=require"

# Auth
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="run: openssl rand -base64 32"

# Vercel Blob (logo storage). Leave empty to fall back to /public uploads.
BLOB_READ_WRITE_TOKEN=""

# Optional: app-wide brand
NEXT_PUBLIC_APP_NAME="Quotation Manager"
```

### 6.2 Prisma client + seed

`src/lib/db.ts`:

```ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === "development" ? ["query", "error"] : ["error"],
  });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

`prisma/seed.ts` (optional — creates initial admin + company):

```ts
import { PrismaClient } from "@prisma/client";
import bcrypt from "bcryptjs";

const prisma = new PrismaClient();

async function main() {
  const email = process.env.SEED_ADMIN_EMAIL ?? "admin@example.com";
  const password = process.env.SEED_ADMIN_PASSWORD ?? "changeme";

  const passwordHash = await bcrypt.hash(password, 12);

  await prisma.user.upsert({
    where: { email },
    update: {},
    create: { email, name: "Admin", passwordHash },
  });

  const existing = await prisma.companySettings.findFirst();
  if (!existing) {
    await prisma.companySettings.create({
      data: {
        name: "Your Company Ltd",
        addressLine1: "123 Example St",
        city: "Auckland",
        postalCode: "1010",
        country: "New Zealand",
        email: "hello@example.com",
        currency: "NZD",
        defaultTaxRate: "15.00",
      },
    });
  }
  console.log(`Seeded admin: ${email} / ${password}`);
}

main().finally(() => prisma.$disconnect());
```

Add to `package.json`:

```json
{
  "prisma": { "seed": "tsx prisma/seed.ts" },
  "scripts": {
    "db:push": "prisma db push",
    "db:migrate": "prisma migrate dev",
    "db:seed": "prisma db seed",
    "db:studio": "prisma studio"
  }
}
```

### 6.3 Validation schemas (shared)

`src/lib/validation.ts`:

```ts
import { z } from "zod";

export const lineItemSchema = z.object({
  id: z.string().optional(),
  description: z.string().min(1, "Description required").max(2000),
  quantity: z.coerce.number().positive("Quantity > 0"),
  unit: z.enum(["hours", "days", "fixed"]).default("hours"),
  unitPrice: z.coerce.number().nonnegative(),
  position: z.number().int().nonnegative().default(0),
});

export const quotationSchema = z.object({
  clientId: z.string().min(1, "Pick a client"),
  issueDate: z.coerce.date(),
  validUntil: z.coerce.date(),
  currency: z.string().length(3).default("USD"),
  discountType: z.enum(["PERCENTAGE", "FIXED"]).nullable().optional(),
  discountValue: z.coerce.number().nonnegative().default(0),
  taxRate: z.coerce.number().min(0).max(100).default(0),
  notes: z.string().max(5000).optional().nullable(),
  terms: z.string().max(5000).optional().nullable(),
  paymentUrl: z.string().url().optional().nullable().or(z.literal("")),
  showQrCode: z.boolean().default(true),
  template: z.enum(["MODERN", "CLASSIC"]).default("MODERN"),
  lineItems: z.array(lineItemSchema).min(1, "At least one line item"),
}).refine((d) => d.validUntil >= d.issueDate, {
  message: "Valid-until must be on or after issue date",
  path: ["validUntil"],
});

export type QuotationInput = z.infer<typeof quotationSchema>;
export type LineItemInput = z.infer<typeof lineItemSchema>;

export const clientSchema = z.object({
  name: z.string().min(1).max(200),
  company: z.string().max(200).optional().nullable(),
  email: z.string().email(),
  phone: z.string().max(50).optional().nullable(),
  addressLine1: z.string().max(200).optional().nullable(),
  addressLine2: z.string().max(200).optional().nullable(),
  city: z.string().max(100).optional().nullable(),
  region: z.string().max(100).optional().nullable(),
  postalCode: z.string().max(20).optional().nullable(),
  country: z.string().max(100).optional().nullable(),
  notes: z.string().max(5000).optional().nullable(),
});

export type ClientInput = z.infer<typeof clientSchema>;

export const settingsSchema = z.object({
  name: z.string().min(1),
  addressLine1: z.string().min(1),
  addressLine2: z.string().optional().nullable(),
  city: z.string().min(1),
  region: z.string().optional().nullable(),
  postalCode: z.string().min(1),
  country: z.string().min(1),
  phone: z.string().optional().nullable(),
  email: z.string().email(),
  website: z.string().url().optional().nullable().or(z.literal("")),
  taxNumber: z.string().optional().nullable(),
  logoUrl: z.string().url().optional().nullable().or(z.literal("")),
  currency: z.string().length(3),
  defaultTaxRate: z.coerce.number().min(0).max(100),
  defaultTerms: z.string().optional().nullable(),
  defaultNotes: z.string().optional().nullable(),
  defaultTemplate: z.enum(["MODERN", "CLASSIC"]),
  defaultPaymentUrl: z.string().url().optional().nullable().or(z.literal("")),
});

export type SettingsInput = z.infer<typeof settingsSchema>;
```

### 6.4 Money math

`src/lib/pdf/calculations.ts`:

```ts
import { Decimal } from "@prisma/client/runtime/library";

export type LineForCalc = {
  quantity: number | string | Decimal;
  unitPrice: number | string | Decimal;
};

export type Totals = {
  subtotal: Decimal;
  discountAmount: Decimal;
  taxableBase: Decimal;
  taxAmount: Decimal;
  total: Decimal;
  lineTotals: Decimal[];
};

const D = (v: number | string | Decimal) => new Decimal(v ?? 0);

export function computeTotals(
  lines: LineForCalc[],
  discount: { type: "PERCENTAGE" | "FIXED" | null | undefined; value: number | string | Decimal },
  taxRatePct: number | string | Decimal,
): Totals {
  const lineTotals = lines.map((l) => D(l.quantity).mul(D(l.unitPrice)).toDecimalPlaces(2));
  const subtotal = lineTotals.reduce((acc, x) => acc.add(x), new Decimal(0));

  let discountAmount = new Decimal(0);
  if (discount.type === "PERCENTAGE") {
    discountAmount = subtotal.mul(D(discount.value)).div(100).toDecimalPlaces(2);
  } else if (discount.type === "FIXED") {
    discountAmount = D(discount.value).toDecimalPlaces(2);
  }
  if (discountAmount.gt(subtotal)) discountAmount = subtotal;

  const taxableBase = subtotal.sub(discountAmount);
  const taxAmount = taxableBase.mul(D(taxRatePct)).div(100).toDecimalPlaces(2);
  const total = taxableBase.add(taxAmount).toDecimalPlaces(2);

  return { subtotal, discountAmount, taxableBase, taxAmount, total, lineTotals };
}
```

This runs both server-side (in actions) and as the source of truth — never duplicate this logic in the client.

### 6.5 Auth (NextAuth)

`src/lib/auth.ts`:

```ts
import NextAuth, { type NextAuthOptions } from "next-auth";
import Credentials from "next-auth/providers/credentials";
import bcrypt from "bcryptjs";
import { prisma } from "@/lib/db";

export const authOptions: NextAuthOptions = {
  session: { strategy: "jwt" },
  providers: [
    Credentials({
      name: "Credentials",
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      async authorize(creds) {
        if (!creds?.email || !creds?.password) return null;
        const user = await prisma.user.findUnique({ where: { email: creds.email } });
        if (!user) return null;
        const ok = await bcrypt.compare(creds.password, user.passwordHash);
        if (!ok) return null;
        return { id: user.id, email: user.email, name: user.name ?? undefined };
      },
    }),
  ],
  pages: { signIn: "/login" },
  callbacks: {
    async session({ session, token }) {
      if (session.user && token.sub) (session.user as any).id = token.sub;
      return session;
    },
  },
};

export const { handlers, auth } = {
  handlers: NextAuth(authOptions),
  auth: async () => (await import("next-auth")).getServerSession(authOptions),
};
```

`src/app/api/auth/[...nextauth]/route.ts`:

```ts
import NextAuth from "next-auth";
import { authOptions } from "@/lib/auth";
const handler = NextAuth(authOptions);
export { handler as GET, handler as POST };
```

`middleware.ts` (project root):

```ts
import { withAuth } from "next-auth/middleware";
export default withAuth({ pages: { signIn: "/login" } });
export const config = { matcher: ["/dashboard/:path*", "/clients/:path*", "/quotations/:path*", "/settings/:path*"] };
```

### 6.6 Server Actions — Quotations CRUD

`src/server/actions/quotations.ts`:

```ts
"use server";

import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { Decimal } from "@prisma/client/runtime/library";
import { prisma } from "@/lib/db";
import { auth } from "@/lib/auth";
import { quotationSchema, type QuotationInput } from "@/lib/validation";
import { computeTotals } from "@/lib/pdf/calculations";

async function requireAuth() {
  const session = await auth();
  if (!session?.user) throw new Error("Unauthorized");
  return session;
}

async function nextQuoteNumber(): Promise<string> {
  const year = new Date().getFullYear();
  const prefix = `Q-${year}-`;
  const last = await prisma.quotation.findFirst({
    where: { number: { startsWith: prefix } },
    orderBy: { number: "desc" },
    select: { number: true },
  });
  const n = last ? parseInt(last.number.slice(prefix.length), 10) + 1 : 1;
  return `${prefix}${String(n).padStart(4, "0")}`;
}

export async function createQuotation(input: QuotationInput) {
  await requireAuth();
  const data = quotationSchema.parse(input);

  const totals = computeTotals(
    data.lineItems,
    { type: data.discountType ?? null, value: data.discountValue },
    data.taxRate,
  );

  const number = await nextQuoteNumber();

  const quote = await prisma.quotation.create({
    data: {
      number,
      clientId: data.clientId,
      issueDate: data.issueDate,
      validUntil: data.validUntil,
      currency: data.currency,
      discountType: data.discountType ?? null,
      discountValue: new Decimal(data.discountValue),
      taxRate: new Decimal(data.taxRate),
      subtotal: totals.subtotal,
      discountAmount: totals.discountAmount,
      taxAmount: totals.taxAmount,
      total: totals.total,
      notes: data.notes ?? null,
      terms: data.terms ?? null,
      paymentUrl: data.paymentUrl || null,
      showQrCode: data.showQrCode,
      template: data.template,
      lineItems: {
        create: data.lineItems.map((li, i) => ({
          description: li.description,
          quantity: new Decimal(li.quantity),
          unit: li.unit,
          unitPrice: new Decimal(li.unitPrice),
          lineTotal: totals.lineTotals[i],
          position: i,
        })),
      },
    },
  });

  revalidatePath("/quotations");
  redirect(`/quotations/${quote.id}`);
}

export async function updateQuotation(id: string, input: QuotationInput) {
  await requireAuth();
  const data = quotationSchema.parse(input);

  const totals = computeTotals(
    data.lineItems,
    { type: data.discountType ?? null, value: data.discountValue },
    data.taxRate,
  );

  await prisma.$transaction([
    prisma.lineItem.deleteMany({ where: { quotationId: id } }),
    prisma.quotation.update({
      where: { id },
      data: {
        clientId: data.clientId,
        issueDate: data.issueDate,
        validUntil: data.validUntil,
        currency: data.currency,
        discountType: data.discountType ?? null,
        discountValue: new Decimal(data.discountValue),
        taxRate: new Decimal(data.taxRate),
        subtotal: totals.subtotal,
        discountAmount: totals.discountAmount,
        taxAmount: totals.taxAmount,
        total: totals.total,
        notes: data.notes ?? null,
        terms: data.terms ?? null,
        paymentUrl: data.paymentUrl || null,
        showQrCode: data.showQrCode,
        template: data.template,
        lineItems: {
          create: data.lineItems.map((li, i) => ({
            description: li.description,
            quantity: new Decimal(li.quantity),
            unit: li.unit,
            unitPrice: new Decimal(li.unitPrice),
            lineTotal: totals.lineTotals[i],
            position: i,
          })),
        },
      },
    }),
  ]);

  revalidatePath("/quotations");
  revalidatePath(`/quotations/${id}`);
}

export async function duplicateQuotation(id: string) {
  await requireAuth();
  const src = await prisma.quotation.findUnique({
    where: { id },
    include: { lineItems: true },
  });
  if (!src) throw new Error("Not found");

  const number = await nextQuoteNumber();

  const dup = await prisma.quotation.create({
    data: {
      number,
      status: "DRAFT",
      clientId: src.clientId,
      issueDate: new Date(),
      validUntil: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
      currency: src.currency,
      discountType: src.discountType,
      discountValue: src.discountValue,
      taxRate: src.taxRate,
      subtotal: src.subtotal,
      discountAmount: src.discountAmount,
      taxAmount: src.taxAmount,
      total: src.total,
      notes: src.notes,
      terms: src.terms,
      paymentUrl: src.paymentUrl,
      showQrCode: src.showQrCode,
      template: src.template,
      lineItems: {
        create: src.lineItems.map((li) => ({
          description: li.description,
          quantity: li.quantity,
          unit: li.unit,
          unitPrice: li.unitPrice,
          lineTotal: li.lineTotal,
          position: li.position,
        })),
      },
    },
  });

  revalidatePath("/quotations");
  redirect(`/quotations/${dup.id}/edit`);
}

export async function changeStatus(
  id: string,
  status: "DRAFT" | "SENT" | "ACCEPTED" | "REJECTED" | "PAID" | "EXPIRED",
) {
  await requireAuth();

  const updates: Record<string, unknown> = { status };
  const now = new Date();
  if (status === "SENT") updates.sentAt = now;
  if (status === "ACCEPTED") updates.acceptedAt = now;
  if (status === "PAID") updates.paidAt = now;

  // Snapshot client + company on first SENT transition.
  if (status === "SENT") {
    const q = await prisma.quotation.findUnique({
      where: { id },
      include: { client: true },
    });
    if (q && !q.clientSnapshot) {
      const settings = await prisma.companySettings.findFirst();
      updates.clientSnapshot = q.client;
      updates.companySnapshot = settings;
    }
  }

  await prisma.quotation.update({ where: { id }, data: updates });
  revalidatePath("/quotations");
  revalidatePath(`/quotations/${id}`);
}

export async function deleteQuotation(id: string) {
  await requireAuth();
  await prisma.quotation.delete({ where: { id } });
  revalidatePath("/quotations");
  redirect("/quotations");
}
```

### 6.7 Logo upload (Vercel Blob)

`src/lib/blob.ts`:

```ts
import { put, del } from "@vercel/blob";

export async function uploadLogo(file: File): Promise<string> {
  if (!process.env.BLOB_READ_WRITE_TOKEN) {
    throw new Error("BLOB_READ_WRITE_TOKEN not configured");
  }
  if (file.size > 2 * 1024 * 1024) throw new Error("Logo must be <2MB");
  if (!/^image\/(png|jpeg|jpg|svg\+xml|webp)$/.test(file.type)) {
    throw new Error("Logo must be PNG, JPG, SVG, or WEBP");
  }
  const ext = file.name.split(".").pop() ?? "png";
  const blob = await put(`logos/${Date.now()}.${ext}`, file, {
    access: "public",
    contentType: file.type,
  });
  return blob.url;
}

export async function deleteBlob(url: string) {
  try {
    await del(url);
  } catch {
    /* ignore */
  }
}
```

`src/server/actions/settings.ts` accepts `FormData` so logo upload + text fields go through one action:

```ts
"use server";

import { revalidatePath } from "next/cache";
import { prisma } from "@/lib/db";
import { auth } from "@/lib/auth";
import { settingsSchema } from "@/lib/validation";
import { uploadLogo, deleteBlob } from "@/lib/blob";

export async function updateSettings(formData: FormData) {
  const session = await auth();
  if (!session?.user) throw new Error("Unauthorized");

  const file = formData.get("logo") as File | null;
  const raw = Object.fromEntries(formData.entries());
  delete (raw as any).logo;

  // Coerce checkbox-like / numeric fields before parse
  if (raw.defaultTaxRate) raw.defaultTaxRate = Number(raw.defaultTaxRate) as any;

  const parsed = settingsSchema.parse(raw);

  let newLogoUrl = parsed.logoUrl ?? undefined;
  if (file && file.size > 0) {
    newLogoUrl = await uploadLogo(file);
  }

  const existing = await prisma.companySettings.findFirst();

  if (existing) {
    if (newLogoUrl && existing.logoUrl && existing.logoUrl !== newLogoUrl) {
      await deleteBlob(existing.logoUrl);
    }
    await prisma.companySettings.update({
      where: { id: existing.id },
      data: { ...parsed, logoUrl: newLogoUrl ?? existing.logoUrl ?? null },
    });
  } else {
    await prisma.companySettings.create({
      data: { ...parsed, logoUrl: newLogoUrl ?? null },
    });
  }

  revalidatePath("/settings");
}
```

### 6.8 Quotation form (client component)

`src/components/quotation-form.tsx`:

```tsx
"use client";

import { useForm, useFieldArray, Controller } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { useState, useTransition } from "react";
import { quotationSchema, type QuotationInput } from "@/lib/validation";
import { computeTotals } from "@/lib/pdf/calculations";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Label } from "@/components/ui/label";
import {
  Select, SelectContent, SelectItem, SelectTrigger, SelectValue,
} from "@/components/ui/select";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Trash2, Plus } from "lucide-react";

type Client = { id: string; name: string; company: string | null };

type Props = {
  clients: Client[];
  defaults?: Partial<QuotationInput>;
  onSubmit: (input: QuotationInput) => Promise<void>;
  submitLabel?: string;
};

export function QuotationForm({ clients, defaults, onSubmit, submitLabel = "Save" }: Props) {
  const [pending, startTransition] = useTransition();
  const [error, setError] = useState<string | null>(null);

  const form = useForm<QuotationInput>({
    resolver: zodResolver(quotationSchema),
    defaultValues: {
      clientId: defaults?.clientId ?? "",
      issueDate: defaults?.issueDate ?? new Date(),
      validUntil:
        defaults?.validUntil ?? new Date(Date.now() + 30 * 24 * 3600 * 1000),
      currency: defaults?.currency ?? "USD",
      discountType: defaults?.discountType ?? null,
      discountValue: defaults?.discountValue ?? 0,
      taxRate: defaults?.taxRate ?? 0,
      notes: defaults?.notes ?? "",
      terms: defaults?.terms ?? "",
      paymentUrl: defaults?.paymentUrl ?? "",
      showQrCode: defaults?.showQrCode ?? true,
      template: defaults?.template ?? "MODERN",
      lineItems: defaults?.lineItems?.length
        ? defaults.lineItems
        : [{ description: "", quantity: 1, unit: "hours", unitPrice: 0, position: 0 }],
    },
  });

  const { fields, append, remove } = useFieldArray({
    control: form.control, name: "lineItems",
  });

  const watched = form.watch();
  const totals = computeTotals(
    watched.lineItems,
    { type: watched.discountType ?? null, value: watched.discountValue },
    watched.taxRate,
  );

  const submit = form.handleSubmit((values) => {
    setError(null);
    startTransition(async () => {
      try {
        await onSubmit(values);
      } catch (e) {
        setError(e instanceof Error ? e.message : "Save failed");
      }
    });
  });

  return (
    <form onSubmit={submit} className="space-y-6">
      <Card>
        <CardHeader><CardTitle>Quotation</CardTitle></CardHeader>
        <CardContent className="grid gap-4 md:grid-cols-2">
          <div>
            <Label>Client *</Label>
            <Controller control={form.control} name="clientId" render={({ field }) => (
              <Select value={field.value} onValueChange={field.onChange}>
                <SelectTrigger><SelectValue placeholder="Select a client" /></SelectTrigger>
                <SelectContent>
                  {clients.map((c) => (
                    <SelectItem key={c.id} value={c.id}>
                      {c.name}{c.company ? ` — ${c.company}` : ""}
                    </SelectItem>
                  ))}
                </SelectContent>
              </Select>
            )}/>
          </div>
          <div>
            <Label>Template</Label>
            <Controller control={form.control} name="template" render={({ field }) => (
              <Select value={field.value} onValueChange={field.onChange}>
                <SelectTrigger><SelectValue /></SelectTrigger>
                <SelectContent>
                  <SelectItem value="MODERN">Modern</SelectItem>
                  <SelectItem value="CLASSIC">Classic</SelectItem>
                </SelectContent>
              </Select>
            )}/>
          </div>
          <div>
            <Label>Issue date *</Label>
            <Input type="date"
              {...form.register("issueDate", { valueAsDate: true })}/>
          </div>
          <div>
            <Label>Valid until *</Label>
            <Input type="date"
              {...form.register("validUntil", { valueAsDate: true })}/>
          </div>
          <div>
            <Label>Currency</Label>
            <Input maxLength={3} {...form.register("currency")} />
          </div>
        </CardContent>
      </Card>

      <Card>
        <CardHeader><CardTitle>Line items</CardTitle></CardHeader>
        <CardContent className="space-y-3">
          {fields.map((f, i) => (
            <div key={f.id} className="grid gap-2 md:grid-cols-12 items-start">
              <div className="md:col-span-5">
                <Textarea rows={2} placeholder="Service description"
                  {...form.register(`lineItems.${i}.description`)} />
              </div>
              <div className="md:col-span-2">
                <Input type="number" step="0.25" placeholder="Qty"
                  {...form.register(`lineItems.${i}.quantity`)} />
              </div>
              <div className="md:col-span-1">
                <Controller control={form.control} name={`lineItems.${i}.unit`} render={({ field }) => (
                  <Select value={field.value} onValueChange={field.onChange}>
                    <SelectTrigger><SelectValue /></SelectTrigger>
                    <SelectContent>
                      <SelectItem value="hours">hr</SelectItem>
                      <SelectItem value="days">d</SelectItem>
                      <SelectItem value="fixed">fix</SelectItem>
                    </SelectContent>
                  </Select>
                )}/>
              </div>
              <div className="md:col-span-2">
                <Input type="number" step="0.01" placeholder="Unit price"
                  {...form.register(`lineItems.${i}.unitPrice`)} />
              </div>
              <div className="md:col-span-1 text-right pt-2 text-sm">
                {totals.lineTotals[i]?.toFixed(2)}
              </div>
              <div className="md:col-span-1">
                <Button type="button" variant="ghost" size="icon"
                  onClick={() => remove(i)} disabled={fields.length === 1}>
                  <Trash2 className="h-4 w-4" />
                </Button>
              </div>
            </div>
          ))}
          <Button type="button" variant="outline" size="sm"
            onClick={() => append({ description: "", quantity: 1, unit: "hours", unitPrice: 0, position: fields.length })}>
            <Plus className="h-4 w-4 mr-1" /> Add item
          </Button>
        </CardContent>
      </Card>

      <Card>
        <CardHeader><CardTitle>Discount, tax & totals</CardTitle></CardHeader>
        <CardContent className="grid gap-4 md:grid-cols-3">
          <div>
            <Label>Discount type</Label>
            <Controller control={form.control} name="discountType" render={({ field }) => (
              <Select value={field.value ?? "NONE"} onValueChange={(v) => field.onChange(v === "NONE" ? null : v)}>
                <SelectTrigger><SelectValue /></SelectTrigger>
                <SelectContent>
                  <SelectItem value="NONE">None</SelectItem>
                  <SelectItem value="PERCENTAGE">Percentage</SelectItem>
                  <SelectItem value="FIXED">Fixed amount</SelectItem>
                </SelectContent>
              </Select>
            )}/>
          </div>
          <div>
            <Label>Discount value</Label>
            <Input type="number" step="0.01" {...form.register("discountValue")} />
          </div>
          <div>
            <Label>Tax rate (%)</Label>
            <Input type="number" step="0.01" {...form.register("taxRate")} />
          </div>
          <div className="md:col-span-3 mt-2 ml-auto text-right text-sm space-y-1">
            <div>Subtotal: <strong>{totals.subtotal.toFixed(2)}</strong></div>
            <div>Discount: −{totals.discountAmount.toFixed(2)}</div>
            <div>Tax: {totals.taxAmount.toFixed(2)}</div>
            <div className="text-base">Total: <strong>{totals.total.toFixed(2)}</strong></div>
          </div>
        </CardContent>
      </Card>

      <Card>
        <CardHeader><CardTitle>Payment & notes</CardTitle></CardHeader>
        <CardContent className="space-y-4">
          <div>
            <Label>Payment URL</Label>
            <Input placeholder="https://buy.stripe.com/... or https://paypal.me/..."
              {...form.register("paymentUrl")} />
            <p className="text-xs text-muted-foreground mt-1">
              Will appear as a clickable Pay Now button in the PDF.
            </p>
          </div>
          <div className="flex items-center gap-2">
            <input id="qr" type="checkbox" {...form.register("showQrCode")} />
            <Label htmlFor="qr">Show QR code beside the Pay Now button</Label>
          </div>
          <div>
            <Label>Notes</Label>
            <Textarea rows={3} {...form.register("notes")} />
          </div>
          <div>
            <Label>Terms & conditions</Label>
            <Textarea rows={4} {...form.register("terms")} />
          </div>
        </CardContent>
      </Card>

      {error && <p className="text-sm text-destructive">{error}</p>}
      <Button type="submit" disabled={pending}>
        {pending ? "Saving…" : submitLabel}
      </Button>
    </form>
  );
}
```

### 6.9 PDF Route Handler

`src/app/api/quotations/[id]/pdf/route.ts`:

```ts
import { NextRequest, NextResponse } from "next/server";
import { prisma } from "@/lib/db";
import { auth } from "@/lib/auth";
import { renderQuotationPdf } from "@/lib/pdf/render";

export const runtime = "nodejs";
export const dynamic = "force-dynamic";

export async function GET(
  _req: NextRequest,
  { params }: { params: { id: string } },
) {
  const session = await auth();
  if (!session?.user) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const quote = await prisma.quotation.findUnique({
    where: { id: params.id },
    include: {
      client: true,
      lineItems: { orderBy: { position: "asc" } },
    },
  });
  if (!quote) {
    return NextResponse.json({ error: "Not found" }, { status: 404 });
  }

  const settings = await prisma.companySettings.findFirst();
  if (!settings) {
    return NextResponse.json(
      { error: "Configure company settings first" },
      { status: 400 },
    );
  }

  try {
    const buffer = await renderQuotationPdf({ quote, client: quote.client, settings });
    return new NextResponse(buffer, {
      status: 200,
      headers: {
        "Content-Type": "application/pdf",
        "Content-Disposition": `inline; filename="${quote.number}.pdf"`,
        "Cache-Control": "private, no-store",
      },
    });
  } catch (e) {
    console.error("[pdf]", e);
    return NextResponse.json({ error: "PDF render failed" }, { status: 500 });
  }
}
```

---

## 7. The PDF Template (full code)

### 7.1 Render entrypoint

`src/lib/pdf/render.ts`:

```ts
import { renderToBuffer } from "@react-pdf/renderer";
import type { Client, CompanySettings, LineItem, Quotation } from "@prisma/client";
import { ModernTemplate } from "./templates/modern";
import { ClassicTemplate } from "./templates/classic";
import { generateQrDataUrl } from "./qrcode";
import type { TemplateProps } from "./templates/types";

type Args = {
  quote: Quotation & { lineItems: LineItem[] };
  client: Client;
  settings: CompanySettings;
};

async function fetchLogoDataUrl(url: string | null | undefined): Promise<string | undefined> {
  if (!url) return undefined;
  try {
    const res = await fetch(url, { cache: "no-store" });
    if (!res.ok) return undefined;
    const buf = Buffer.from(await res.arrayBuffer());
    const ct = res.headers.get("content-type") ?? "image/png";
    return `data:${ct};base64,${buf.toString("base64")}`;
  } catch {
    return undefined;
  }
}

export async function renderQuotationPdf(args: Args): Promise<Buffer> {
  const { quote, client, settings } = args;

  // Prefer snapshots when present so sent quotes are stable
  const effectiveClient = (quote.clientSnapshot as any) ?? client;
  const effectiveCompany = (quote.companySnapshot as any) ?? settings;

  const logoDataUrl = await fetchLogoDataUrl(effectiveCompany.logoUrl);

  const qrDataUrl =
    quote.showQrCode && quote.paymentUrl
      ? await generateQrDataUrl(quote.paymentUrl)
      : undefined;

  const props: TemplateProps = {
    quote,
    client: effectiveClient,
    company: effectiveCompany,
    logoDataUrl,
    qrDataUrl,
  };

  const Component = quote.template === "CLASSIC" ? ClassicTemplate : ModernTemplate;

  return await renderToBuffer(<Component {...props} />);
}
```

### 7.2 Shared types

`src/lib/pdf/templates/types.ts`:

```ts
import type { Client, CompanySettings, LineItem, Quotation } from "@prisma/client";

export type TemplateProps = {
  quote: Quotation & { lineItems: LineItem[] };
  client: Client | any;
  company: CompanySettings | any;
  logoDataUrl?: string;
  qrDataUrl?: string;
};
```

### 7.3 QR code

`src/lib/pdf/qrcode.ts`:

```ts
import QRCode from "qrcode";

export async function generateQrDataUrl(url: string): Promise<string> {
  return await QRCode.toDataURL(url, {
    margin: 1,
    width: 240,
    errorCorrectionLevel: "M",
    color: { dark: "#0f172a", light: "#ffffff" },
  });
}
```

### 7.4 Modern template (full)

`src/lib/pdf/templates/modern.tsx`:

```tsx
import {
  Document, Page, Text, View, Image, Link, StyleSheet,
} from "@react-pdf/renderer";
import type { TemplateProps } from "./types";

const C = {
  ink: "#0f172a", muted: "#475569", line: "#e2e8f0",
  accent: "#2563eb", accentSoft: "#dbeafe", success: "#16a34a",
  bg: "#ffffff", subtle: "#f8fafc",
};

const s = StyleSheet.create({
  page: {
    paddingTop: 40, paddingBottom: 60, paddingHorizontal: 40,
    fontSize: 10, color: C.ink, fontFamily: "Helvetica", backgroundColor: C.bg,
  },
  header: {
    flexDirection: "row", justifyContent: "space-between",
    alignItems: "flex-start", marginBottom: 24,
  },
  logo: { width: 110, height: 40, objectFit: "contain" },
  companyBlock: { textAlign: "right" },
  companyName: { fontSize: 12, fontWeight: 700, marginBottom: 2 },
  meta: { color: C.muted, fontSize: 9, lineHeight: 1.4 },
  titleRow: {
    flexDirection: "row", justifyContent: "space-between",
    alignItems: "flex-end", marginBottom: 16,
  },
  title: { fontSize: 22, fontWeight: 700, color: C.accent, letterSpacing: 1 },
  numberPill: {
    backgroundColor: C.accentSoft, color: C.accent,
    paddingHorizontal: 10, paddingVertical: 4, borderRadius: 4,
    fontSize: 11, fontWeight: 700,
  },
  twoCol: { flexDirection: "row", gap: 16, marginBottom: 16 },
  col: {
    flex: 1, padding: 12, borderRadius: 4,
    backgroundColor: C.subtle, borderLeft: `3pt solid ${C.accent}`,
  },
  colLabel: {
    fontSize: 8, color: C.muted, textTransform: "uppercase",
    letterSpacing: 1, marginBottom: 4,
  },
  bold: { fontWeight: 700 },
  datesRow: {
    flexDirection: "row", justifyContent: "space-between",
    marginBottom: 16, paddingVertical: 10,
    borderTop: `1pt solid ${C.line}`, borderBottom: `1pt solid ${C.line}`,
  },
  dateBlock: { flex: 1 },
  table: { marginBottom: 16 },
  thead: {
    flexDirection: "row", backgroundColor: C.ink, color: "#fff",
    paddingVertical: 8, paddingHorizontal: 8, fontSize: 9,
    fontWeight: 700, textTransform: "uppercase", letterSpacing: 0.5,
  },
  tr: {
    flexDirection: "row", paddingVertical: 8, paddingHorizontal: 8,
    borderBottom: `1pt solid ${C.line}`,
  },
  cDesc: { width: "50%", paddingRight: 8 },
  cQty: { width: "12%", textAlign: "right" },
  cUnit: { width: "13%", textAlign: "right" },
  cPrice: { width: "12%", textAlign: "right" },
  cTotal: { width: "13%", textAlign: "right" },
  totalsWrap: {
    flexDirection: "row", justifyContent: "flex-end", marginBottom: 16,
  },
  totalsBox: { width: 240 },
  totalRow: {
    flexDirection: "row", justifyContent: "space-between",
    paddingVertical: 4,
  },
  grandRow: {
    flexDirection: "row", justifyContent: "space-between",
    paddingVertical: 8, marginTop: 4,
    borderTop: `2pt solid ${C.ink}`, fontSize: 12, fontWeight: 700,
  },
  payWrap: {
    marginTop: 8, marginBottom: 16, padding: 14, borderRadius: 6,
    backgroundColor: C.subtle,
    flexDirection: "row", alignItems: "center", justifyContent: "space-between",
  },
  payTextWrap: { flex: 1, paddingRight: 12 },
  payHeading: { fontSize: 11, fontWeight: 700, marginBottom: 4 },
  payBlurb: { color: C.muted, fontSize: 9, marginBottom: 8 },
  payButton: {
    backgroundColor: C.accent, color: "#fff",
    paddingVertical: 10, paddingHorizontal: 16,
    borderRadius: 4, alignSelf: "flex-start",
  },
  payButtonText: { color: "#fff", fontWeight: 700, fontSize: 11 },
  qr: { width: 80, height: 80 },
  notesBlock: { marginTop: 8, marginBottom: 8 },
  notesLabel: {
    fontSize: 9, color: C.muted, textTransform: "uppercase",
    letterSpacing: 1, marginBottom: 4,
  },
  notesBody: { fontSize: 9.5, lineHeight: 1.5, color: C.muted },
  footer: {
    position: "absolute", bottom: 24, left: 40, right: 40,
    flexDirection: "row", justifyContent: "space-between",
    paddingTop: 8, borderTop: `1pt solid ${C.line}`,
    fontSize: 8, color: C.muted,
  },
});

const fmtMoney = (v: any, currency: string) => {
  const n = typeof v === "number" ? v : Number(v?.toString?.() ?? v ?? 0);
  return new Intl.NumberFormat("en-US", {
    style: "currency", currency, maximumFractionDigits: 2,
  }).format(n);
};
const fmtDate = (d: Date | string) =>
  new Date(d).toLocaleDateString("en-GB", {
    day: "2-digit", month: "short", year: "numeric",
  });

export function ModernTemplate({
  quote, client, company, logoDataUrl, qrDataUrl,
}: TemplateProps) {
  const cur = quote.currency;

  return (
    <Document title={quote.number} author={company.name}>
      <Page size="A4" style={s.page}>
        {/* Fixed header */}
        <View style={s.header} fixed>
          {logoDataUrl ? (
            <Image src={logoDataUrl} style={s.logo} />
          ) : (
            <View>
              <Text style={[s.companyName, { color: C.accent }]}>{company.name}</Text>
            </View>
          )}
          <View style={s.companyBlock}>
            <Text style={s.companyName}>{company.name}</Text>
            <Text style={s.meta}>
              {company.addressLine1}{company.addressLine2 ? `, ${company.addressLine2}` : ""}
            </Text>
            <Text style={s.meta}>
              {company.city}{company.region ? `, ${company.region}` : ""} {company.postalCode}
            </Text>
            <Text style={s.meta}>{company.country}</Text>
            {company.phone && <Text style={s.meta}>{company.phone}</Text>}
            <Text style={s.meta}>{company.email}</Text>
            {company.website && <Text style={s.meta}>{company.website}</Text>}
            {company.taxNumber && <Text style={s.meta}>Tax No: {company.taxNumber}</Text>}
          </View>
        </View>

        {/* Title */}
        <View style={s.titleRow}>
          <Text style={s.title}>QUOTATION</Text>
          <Text style={s.numberPill}>{quote.number}</Text>
        </View>

        {/* Bill-to */}
        <View style={s.twoCol}>
          <View style={s.col}>
            <Text style={s.colLabel}>Bill to</Text>
            <Text style={s.bold}>{client.name}</Text>
            {client.company && <Text>{client.company}</Text>}
            {client.addressLine1 && <Text>{client.addressLine1}</Text>}
            {client.addressLine2 && <Text>{client.addressLine2}</Text>}
            {(client.city || client.postalCode) && (
              <Text>
                {[client.city, client.region, client.postalCode].filter(Boolean).join(" ")}
              </Text>
            )}
            {client.country && <Text>{client.country}</Text>}
            <Text style={{ marginTop: 4 }}>{client.email}</Text>
            {client.phone && <Text>{client.phone}</Text>}
          </View>
          <View style={s.col}>
            <Text style={s.colLabel}>Status</Text>
            <Text style={s.bold}>{quote.status}</Text>
            <Text style={[s.colLabel, { marginTop: 8 }]}>Currency</Text>
            <Text>{cur}</Text>
          </View>
        </View>

        {/* Dates */}
        <View style={s.datesRow}>
          <View style={s.dateBlock}>
            <Text style={s.colLabel}>Issue date</Text>
            <Text style={s.bold}>{fmtDate(quote.issueDate)}</Text>
          </View>
          <View style={s.dateBlock}>
            <Text style={s.colLabel}>Valid until</Text>
            <Text style={s.bold}>{fmtDate(quote.validUntil)}</Text>
          </View>
        </View>

        {/* Line items */}
        <View style={s.table}>
          <View style={s.thead} fixed>
            <Text style={s.cDesc}>Description</Text>
            <Text style={s.cQty}>Qty</Text>
            <Text style={s.cUnit}>Unit</Text>
            <Text style={s.cPrice}>Unit price</Text>
            <Text style={s.cTotal}>Line total</Text>
          </View>
          {quote.lineItems.map((li) => (
            <View key={li.id} style={s.tr} wrap={false}>
              <Text style={s.cDesc}>{li.description}</Text>
              <Text style={s.cQty}>{Number(li.quantity).toFixed(2)}</Text>
              <Text style={s.cUnit}>{li.unit}</Text>
              <Text style={s.cPrice}>{fmtMoney(li.unitPrice, cur)}</Text>
              <Text style={s.cTotal}>{fmtMoney(li.lineTotal, cur)}</Text>
            </View>
          ))}
        </View>

        {/* Totals */}
        <View style={s.totalsWrap} wrap={false}>
          <View style={s.totalsBox}>
            <View style={s.totalRow}>
              <Text>Subtotal</Text>
              <Text>{fmtMoney(quote.subtotal, cur)}</Text>
            </View>
            {Number(quote.discountAmount) > 0 && (
              <View style={s.totalRow}>
                <Text>
                  Discount{quote.discountType === "PERCENTAGE" ? ` (${Number(quote.discountValue)}%)` : ""}
                </Text>
                <Text>−{fmtMoney(quote.discountAmount, cur)}</Text>
              </View>
            )}
            {Number(quote.taxRate) > 0 && (
              <View style={s.totalRow}>
                <Text>Tax ({Number(quote.taxRate)}%)</Text>
                <Text>{fmtMoney(quote.taxAmount, cur)}</Text>
              </View>
            )}
            <View style={s.grandRow}>
              <Text>Total</Text>
              <Text>{fmtMoney(quote.total, cur)}</Text>
            </View>
          </View>
        </View>

        {/* Pay Now */}
        {quote.paymentUrl ? (
          <View style={s.payWrap} wrap={false}>
            <View style={s.payTextWrap}>
              <Text style={s.payHeading}>Ready to proceed?</Text>
              <Text style={s.payBlurb}>
                Pay this quotation securely online — your access link is below.
                {qrDataUrl ? " Scan the QR with your phone, or click the button." : ""}
              </Text>
              <Link src={quote.paymentUrl}>
                <View style={s.payButton}>
                  <Text style={s.payButtonText}>Click here to pay online →</Text>
                </View>
              </Link>
            </View>
            {qrDataUrl && (
              <Link src={quote.paymentUrl}>
                <Image src={qrDataUrl} style={s.qr} />
              </Link>
            )}
          </View>
        ) : null}

        {/* Notes & Terms */}
        {quote.notes && (
          <View style={s.notesBlock} wrap={false}>
            <Text style={s.notesLabel}>Notes</Text>
            <Text style={s.notesBody}>{quote.notes}</Text>
          </View>
        )}
        {quote.terms && (
          <View style={s.notesBlock} wrap={false}>
            <Text style={s.notesLabel}>Terms & Conditions</Text>
            <Text style={s.notesBody}>{quote.terms}</Text>
          </View>
        )}

        {/* Footer */}
        <View style={s.footer} fixed>
          <Text>
            {company.name} · Quotation {quote.number}
          </Text>
          <Text render={({ pageNumber, totalPages }) => `Page ${pageNumber} of ${totalPages}`} />
        </View>
      </Page>
    </Document>
  );
}
```

### 7.5 Classic template (sketch)

`src/lib/pdf/templates/classic.tsx` follows the same data contract with a serif-feeling, more formal layout — centered title, single-column billing block, double-ruled totals box. Drop in different `StyleSheet` values; everything else stays the same. (Skipping the full duplicate to keep this doc focused — clone Modern, swap accent color to `#1f2937`, remove the colored title pill, switch the `<Page>` font to a serif registered via `Font.register`.)

---

## 8. Pages — wiring it together

### Quotation list

`src/app/(admin)/quotations/page.tsx`:

```tsx
import Link from "next/link";
import { prisma } from "@/lib/db";
import { Button } from "@/components/ui/button";

export default async function QuotationsPage() {
  const quotes = await prisma.quotation.findMany({
    include: { client: true },
    orderBy: { createdAt: "desc" },
  });

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-2xl font-semibold">Quotations</h1>
        <Button asChild><Link href="/quotations/new">New quote</Link></Button>
      </div>
      <div className="border rounded-lg overflow-hidden">
        <table className="w-full text-sm">
          <thead className="bg-muted">
            <tr>
              <th className="text-left p-3">Number</th>
              <th className="text-left p-3">Client</th>
              <th className="text-left p-3">Status</th>
              <th className="text-right p-3">Total</th>
              <th className="text-right p-3">Issued</th>
            </tr>
          </thead>
          <tbody>
            {quotes.map((q) => (
              <tr key={q.id} className="border-t hover:bg-muted/50">
                <td className="p-3"><Link className="underline" href={`/quotations/${q.id}`}>{q.number}</Link></td>
                <td className="p-3">{q.client.name}</td>
                <td className="p-3">{q.status}</td>
                <td className="p-3 text-right">{q.currency} {Number(q.total).toFixed(2)}</td>
                <td className="p-3 text-right">{q.issueDate.toLocaleDateString()}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
}
```

### New quotation page

`src/app/(admin)/quotations/new/page.tsx`:

```tsx
import { prisma } from "@/lib/db";
import { QuotationForm } from "@/components/quotation-form";
import { createQuotation } from "@/server/actions/quotations";

export default async function NewQuotePage() {
  const [clients, settings] = await Promise.all([
    prisma.client.findMany({ orderBy: { name: "asc" } }),
    prisma.companySettings.findFirst(),
  ]);

  return (
    <div className="space-y-6 max-w-4xl">
      <h1 className="text-2xl font-semibold">New quotation</h1>
      <QuotationForm
        clients={clients}
        defaults={{
          currency: settings?.currency ?? "USD",
          taxRate: Number(settings?.defaultTaxRate ?? 0),
          terms: settings?.defaultTerms ?? "",
          notes: settings?.defaultNotes ?? "",
          paymentUrl: settings?.defaultPaymentUrl ?? "",
          template: settings?.defaultTemplate ?? "MODERN",
        }}
        onSubmit={createQuotation}
        submitLabel="Create quote"
      />
    </div>
  );
}
```

### Quotation detail page (with PDF link)

`src/app/(admin)/quotations/[id]/page.tsx`:

```tsx
import Link from "next/link";
import { notFound } from "next/navigation";
import { prisma } from "@/lib/db";
import { Button } from "@/components/ui/button";
import { changeStatus, duplicateQuotation, deleteQuotation } from "@/server/actions/quotations";

export default async function QuoteDetail({ params }: { params: { id: string } }) {
  const q = await prisma.quotation.findUnique({
    where: { id: params.id },
    include: { client: true, lineItems: { orderBy: { position: "asc" } } },
  });
  if (!q) return notFound();

  return (
    <div className="space-y-6 max-w-4xl">
      <div className="flex justify-between items-start">
        <div>
          <h1 className="text-2xl font-semibold">{q.number}</h1>
          <p className="text-muted-foreground">{q.client.name} · {q.status}</p>
        </div>
        <div className="flex gap-2">
          <Button asChild variant="outline">
            <a href={`/api/quotations/${q.id}/pdf`} target="_blank" rel="noreferrer">View PDF</a>
          </Button>
          <Button asChild><Link href={`/quotations/${q.id}/edit`}>Edit</Link></Button>
          <form action={duplicateQuotation.bind(null, q.id)}>
            <Button type="submit" variant="outline">Duplicate</Button>
          </form>
        </div>
      </div>

      {/* Status transitions */}
      <div className="flex gap-2 flex-wrap">
        {(["DRAFT","SENT","ACCEPTED","REJECTED","PAID","EXPIRED"] as const).map((s) => (
          <form key={s} action={changeStatus.bind(null, q.id, s)}>
            <Button type="submit" variant={q.status === s ? "default" : "outline"} size="sm">
              Mark {s.toLowerCase()}
            </Button>
          </form>
        ))}
      </div>

      {/* ... totals + line items rendered for review ... */}
    </div>
  );
}
```

---

## 9. Deployment

### Vercel + Neon (recommended)

1. **Postgres:** Create a free Neon project. Copy the pooled connection string into `DATABASE_URL`.
2. **Repo:** Push to GitHub.
3. **Vercel project:** Import the repo. Set environment variables:
   - `DATABASE_URL`
   - `NEXTAUTH_URL` (your prod URL)
   - `NEXTAUTH_SECRET` (`openssl rand -base64 32`)
   - `BLOB_READ_WRITE_TOKEN` (created from the Vercel Storage tab → Blob)
4. **Build settings:** Add `prisma generate && prisma migrate deploy` to the Build Command, e.g.:
   ```
   prisma generate && prisma migrate deploy && next build
   ```
5. **First deploy:** SSH-free; Vercel runs the build command which applies migrations.
6. **Seed:** Run once locally against the prod DB:
   ```bash
   DATABASE_URL="..." npm run db:seed
   ```

### Railway alternative

Railway gives you Postgres + Next.js in one project. Use the same env vars. Add a post-deploy command for `prisma migrate deploy`. Use S3 (or Railway's volume) for logos by swapping `src/lib/blob.ts` to use `@aws-sdk/client-s3`.

### Self-hosted (Docker)

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json prisma ./
RUN npm ci

FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate && npm run build

FROM node:20-alpine AS run
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/.next ./.next
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/prisma ./prisma
COPY --from=build /app/public ./public
COPY package.json ./
EXPOSE 3000
CMD ["sh","-c","npx prisma migrate deploy && npm run start"]
```

---

## 10. Security & Best Practices

- **Server-only secrets.** `DATABASE_URL`, `BLOB_READ_WRITE_TOKEN`, `NEXTAUTH_SECRET` are never imported into client components.
- **All mutations gated.** Every server action calls `requireAuth()` first.
- **Validate before persist.** Zod parses input on the server before DB writes — never trust the form.
- **PDF endpoint is auth-gated.** If you want public "share via link" PDFs later, add a separate signed-token endpoint (HMAC over `quoteId` + `expiresAt`); don't expose the existing `/api/quotations/[id]/pdf` route.
- **File upload limits.** 2MB, MIME-checked. Reject SVGs in production if you can't sanitize them — they're an XSS vector if served directly.
- **Decimal everywhere for money.** No `Number.toFixed` for arithmetic, only for display.
- **Rate-limit the PDF route.** It does work. Use Vercel's `unstable_after` or add Upstash Redis for a per-IP limit if you ever expose it externally.
- **Audit log (later).** Add a `QuotationEvent` table that records status transitions with `userId` + `timestamp` + `meta`.

---

## 11. Implementation order (suggested)

1. Scaffold Next.js + Prisma + Tailwind + shadcn.
2. Schema → migrate → seed admin + company.
3. Auth + login page + middleware.
4. Settings page (logo upload + company fields). **Get this working before quotations** — the PDF needs it.
5. Clients CRUD.
6. Quotation form (no PDF yet, just save).
7. PDF route + Modern template. Test with several edge cases: 0 line items rejected, 100 items wraps cleanly, very long descriptions, missing logo, no payment URL, payment URL with no QR.
8. Status transitions + duplicate.
9. Classic template.
10. Deploy to Vercel.

---

## 12. Edge cases I've already handled in the code above

- Discount > subtotal → clamped to subtotal (no negative bases).
- Empty line items → rejected by Zod.
- `validUntil < issueDate` → rejected by Zod refinement.
- Logo URL fails to fetch → renders text fallback, doesn't crash the PDF.
- No payment URL → Pay Now block hidden entirely (not a sad empty button).
- QR opt-out → respected per quote.
- Very long line item table → header repeats on each page (`fixed` thead), totals/pay block stay together (`wrap={false}`).
- Currency formatting → `Intl.NumberFormat` with the quote's currency, not the company default.
- Sent quotes → snapshotted so later edits to client/company don't rewrite history.

---

That's the full spec. Build it in the order in section 11 and you'll have a working v1 in roughly a long weekend, production-hardened in another week.
