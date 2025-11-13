# 🗓️ Day 7–8 — Database Schema + Listings API (Final Implementation)

**Sprint Theme:** Core Data Model + Listings Retrieval  
**Duration:** 2 Days (16 hours)  
**Developers:** Dev A (Schema + Seed), Dev B (API + Tests)

---

## 🎯 Objectives
- Finalize Drizzle schema for all entities
- Run migrations and verify table relationships  
- Seed realistic data
- Implement paginated listings API with filters and search
- Secure routes using Clerk
- Ensure type safety, performance, and query correctness

---

## ⚙️ Stack Alignment
| Layer | Tech |
|-------|------|
| Frontend | Next.js 15 (App Router + TS) |
| Backend | Express 5 (ESM + tsx runtime) |
| ORM | Drizzle ORM (PostgreSQL + Neon) |
| Auth | Clerk |
| Package Manager | pnpm 9.x |
| Testing | Vitest + Supertest |
| Validation | Zod |

---

## 🗄️ Day 7 — Database Schema + Seed (8h)

### Morning (4h): Schema Design & Migration Execution

#### 📁 `apps/api/db/schema.ts`
```typescript
import { pgTable, uuid, text, timestamp, integer, boolean, index, varchar } from "drizzle-orm/pg-core";
import { relations } from "drizzle-orm";

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  clerkId: varchar("clerk_id", { length: 255 }).notNull().unique(),
  email: varchar("email", { length: 255 }).notNull(),
  name: varchar("name", { length: 255 }),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

export const listings = pgTable(
  "listings",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    title: varchar("title", { length: 255 }).notNull(),
    description: text("description").notNull(),
    price: integer("price").notNull(),
    status: varchar("status", { length: 50 }).default("active"),
    city: varchar("city", { length: 100 }),
    categoryId: uuid("category_id").references(() => categories.id),
    ownerId: uuid("owner_id").references(() => users.id),
    createdAt: timestamp("created_at").defaultNow().notNull(),
  },
  (table) => ({
    cityStatusIdx: index("listings_city_status_idx").on(table.city, table.status),
    searchIdx: index("listings_search_idx").on(table.title),
  })
);

export const categories = pgTable("categories", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 100 }).notNull(),
  slug: varchar("slug", { length: 100 }).unique().notNull(),
});

export const images = pgTable("images", {
  id: uuid("id").primaryKey().defaultRandom(),
  listingId: uuid("listing_id").references(() => listings.id).notNull(),
  url: text("url").notNull(),
  isPrimary: boolean("is_primary").default(false),
});

export const listingsRelations = relations(listings, ({ one, many }) => ({
  owner: one(users, { fields: [listings.ownerId], references: [users.id] }),
  category: one(categories, { fields: [listings.categoryId], references: [categories.id] }),
  images: many(images),
}));```

#### 🚀 Migration Commands

```# Generate migrations
pnpm drizzle-kit generate:pg

# Push to database
pnpm drizzle-kit push

# Open Drizzle Studio
pnpm drizzle-kit studio```
```
### Afternoon (4h): Seed Data & Verification
#### 📁 `apps/api/scripts/seed.ts`

```import { db } from "../db/client";
import { users, categories, listings, images } from "../db/schema";
import { eq } from "drizzle-orm";

async function main() {
  console.log("🌱 Starting seed...");

  const categoryData = [
    { name: "Electronics", slug: "electronics" },
    { name: "Furniture", slug: "furniture" },
    { name: "Vehicles", slug: "vehicles" },
  ];

  await db.insert(categories).values(categoryData).onConflictDoNothing();

  const user = await db
    .insert(users)
    .values({
      clerkId: "user_123",
      email: "demo@stufflux.com",
      name: "Demo User",
    })
    .returning({ id: users.id });

  await db.insert(listings).values([
    {
      title: "iPhone 14 Pro",
      description: "Mint condition, 128GB storage.",
      price: 250000,
      city: "Lahore",
      categoryId: (await db.select().from(categories).where(eq(categories.slug, "electronics")))[0].id,
      ownerId: user[0].id,
    },
  ]);

  console.log("✅ Seed complete!");
  process.exit(0);
}

main().catch((err) => {
  console.error("❌ Seed failed:", err);
  process.exit(1);
});```
```
#### 📦 Package.json Scripts
```{
  "scripts": {
    "db:generate": "drizzle-kit generate:pg",
    "db:push": "drizzle-kit push",
    "db:studio": "drizzle-kit studio",
    "db:seed": "tsx scripts/seed.ts"
  }
}
```
## 🚀 Day 8 — Listings API (8h)

### Morning (4h): API Routes + Validation

#### 📁 `apps/api/src/utils/asyncHandler.ts`
```
import { Request, Response, NextFunction } from "express";

export const asyncHandler =
  (fn: Function) =>
  (req: Request, res: Response, next: NextFunction) =>
    Promise.resolve(fn(req, res, next)).catch(next);

export default asyncHandler;
```
#### 📁 `apps/api/src/routes/listings.ts`
```import express from "express";
import { db } from "../../db/client";
import { listings, images, categories, users } from "../../db/schema";
import { z } from "zod";
import { eq, ilike, and, desc, sql } from "drizzle-orm";
import asyncHandler from "../utils/asyncHandler";
import { requireAuth } from "@clerk/express";

const router = express.Router();

const querySchema = z.object({
  city: z.string().optional(),
  category: z.string().optional(),
  status: z.enum(["active", "inactive"]).optional(),
  search: z.string().optional(),
  limit: z.number().max(50).default(10),
  offset: z.number().default(0),
});

router.get(
  "/",
  asyncHandler(async (req, res) => {
    try {
      const params = querySchema.parse(req.query);

      const filters = [];
      if (params.city) filters.push(eq(listings.city, params.city));
      if (params.status) filters.push(eq(listings.status, params.status));
      if (params.category) filters.push(eq(categories.slug, params.category));

      const data = await db
        .select({
          id: listings.id,
          title: listings.title,
          price: listings.price,
          city: listings.city,
          status: listings.status,
          createdAt: listings.createdAt,
          category: categories.name,
          primaryImageUrl: sql<string>`(SELECT url FROM images WHERE images.listing_id = listings.id AND images.is_primary = true LIMIT 1)`,
        })
        .from(listings)
        .leftJoin(categories, eq(categories.id, listings.categoryId))
        .where(and(...filters))
        .limit(params.limit)
        .offset(params.offset)
        .orderBy(desc(listings.createdAt));

      res.json({ success: true, data });
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          error: "Invalid query parameters",
          details: error.errors.map((e) => ({
            field: e.path.join("."),
            message: e.message,
          })),
        });
      }
      throw error;
    }
  })
);

const bodySchema = z.object({
  title: z.string().min(3),
  description: z.string().min(10),
  price: z.number().positive(),
  city: z.string(),
  categoryId: z.string().uuid(),
});

router.post(
  "/",
  requireAuth(),
  asyncHandler(async (req, res) => {
    const { userId } = req.auth;
    const body = bodySchema.parse(req.body);

    const result = await db
      .insert(listings)
      .values({ ...body, ownerId: userId })
      .returning({ id: listings.id });

    res.status(201).json({ success: true, id: result[0].id });
  })
);

export default router;
```
### Afternoon (4h): Testing & Validation

#### 📁 `vitest.config.ts`

```import { defineConfig } from 'vitest/config';

process.env.DATABASE_URL = process.env.TEST_DATABASE_URL || "postgres://localhost/stufflux_test";

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./src/test-setup.ts'],
  },
});
```
#### 📁 `apps/api/tests/listings.test.ts`
```import request from "supertest";
import app from "../src/app";

describe("GET /api/listings", () => {
  it("returns listings with primary images", async () => {
    const res = await request(app).get("/api/listings");
    expect(res.status).toBe(200);
    expect(res.body.data[0]).toHaveProperty("primaryImageUrl");
  });

  it("handles large limit parameter gracefully", async () => {
    const res = await request(app).get("/api/listings?limit=100");
    expect(res.status).toBe(400);
  });

  it("filters by multiple criteria", async () => {
    const res = await request(app).get("/api/listings?city=Lahore&status=active");
    expect(res.status).toBe(200);
  });
});
```
#### 📦 Test Scripts
```{
  "scripts": {
    "test": "DATABASE_URL=$TEST_DATABASE_URL vitest",
    "test:watch": "DATABASE_URL=$TEST_DATABASE_URL vitest --watch",
    "test:coverage": "DATABASE_URL=$TEST_DATABASE_URL vitest --coverage"
  }
}
```
### Day 7 Deliverables

- All database tables created with proper relationships
    
- Migrations executed successfully
    
- Seed script populates database with realistic data
    
- Composite indexes added for performance
    
- Data integrity verified in Drizzle Studio
    

### Day 8 Deliverables

- GET /api/listings with pagination and filters
    
- POST /api/listings with Clerk authentication
    
- Comprehensive error handling and validation
    
- Unit tests passing with >80% coverage
    
- Test database isolation implemented

## 🧪 Verification Checklist

### Database Verification
```# Check tables
pnpm db:studio

# Run seed
pnpm db:seed

# Verify data
SELECT COUNT(*) FROM listings; -- Should return seeded count
```
### API Verification
```# Test endpoints
curl http://localhost:3001/api/listings
curl "http://localhost:3001/api/listings?city=Lahore&limit=5"
curl -X POST http://localhost:3001/api/listings -H "Authorization: Bearer {token}" -d '{"title":"Test",...}'

# Run tests
pnpm test
```
