# 🚀 Vanta API - Complete NPM Package Documentation

**Vanta API** is a powerful, production-ready utility toolkit for building secure, scalable APIs with **Express.js** and **Mongoose**. It provides advanced data filtering, searching, sorting, pagination, and population with built-in security features and error handling.

---

## Table of Contents

1. [Features](#-features)
2. [Installation](#-installation)
3. [Quick Start](#quick-start)
4. [Core Concepts](#core-concepts)
5. [API Reference](#api-reference)
   - [Constructor](#constructor)
   - [filter()](#filter)
   - [addManualFilters()](#addmanualfilters)
   - [search()](#search)
   - [sort()](#sort)
   - [limitFields()](#limitfields)
   - [populate()](#populate)
   - [paginate()](#paginate)
   - [execute()](#execute)
6. [Real-World Examples](#real-world-examples)
7. [Error Handling](#error-handling)
8. [Security Configuration](#security-configuration)
9. [Best Practices](#best-practices)

---

## ✨ Features

| Feature | Description |
|---------|-------------|
| **Advanced Filtering** | Complex query operators with automatic type conversion |
| **Server-Side Filters** | Enforce backend conditions without exposing to users |
| **Logical Operators** | Recursive `$and`, `$or`, `$nor` support with dot notation |
| **Automatic Type Conversion** | Convert strings to ObjectId, booleans, and numbers intelligently |
| **Full-Text Search** | Case-insensitive regex search across multiple fields |
| **Sorting** | Multi-field sorting with `-` prefix for descending |
| **Field Projection** | Include/exclude fields with forbidden field protection |
| **Pagination** | Cursor-based and skip-limit pagination with role-based limits |
| **Aggregation-Based Populate** | MongoDB `$lookup` with nested populate support |
| **Nested Populate** | Deep population with select and role-based access control |
| **Role-Based Security** | Access levels with max limits and allowed operations |
| **Forbidden Field Protection** | Automatically block sensitive fields from all results |
| **Request Sanitization** | Prevent NoSQL injection attacks |
| **Async Error Handling** | Centralized error management with custom error classes |
| **Express Integration** | Seamless integration with Express error middleware |

---

## 📦 Installation

```bash
npm install vanta-api
```

### Prerequisites

Your project must have these dependencies:

```bash
npm install express mongoose pluralize winston
```

### Peer Dependencies

```json
{
  "peerDependencies": {
    "mongoose": "^7 || ^8 || ^9"
  }
}
```

---

## Quick Start

### 1. Basic Setup

```javascript
import express from "express";
import ApiFeatures, { catchAsync, catchError } from "vanta-api";
import Product from "./models/Product.js";

const app = express();
app.use(express.json());

// Simple endpoint
app.get(
  "/api/products",
  catchAsync(async (req, res) => {
    const result = await new ApiFeatures(Product, req.query, req.user?.role)
      .filter()
      .search(["name", "description"])
      .sort()
      .limitFields()
      .paginate()
      .execute();

    res.json(result);
  })
);

// Global error handler (must be last)
app.use(catchError);

app.listen(3000);
```

### 2. Create Security Config (optional)

In your project root, create `security-config.js`:

```javascript
export const securityConfig = {
  forbiddenFields: ["password", "refreshToken"],
  allowedOperators: ["eq", "ne", "gt", "gte", "lt", "lte", "in", "nin", "regex"],
  accessLevels: {
    guest: { maxLimit: 50, allowedPopulate: ["*"] },
    user: { maxLimit: 100, allowedPopulate: ["*"] },
    admin: { maxLimit: 1000, allowedPopulate: ["*"] }
  }
};
```

### 3. Make Your First Request

```bash
# Basic filter
curl "http://localhost:3000/api/products?category=electronics"

# With sorting and pagination
curl "http://localhost:3000/api/products?sort=-createdAt&page=1&limit=10"

# With search
curl "http://localhost:3000/api/products?q=iphone"

# With field limiting
curl "http://localhost:3000/api/products?fields=name,price,-_id"
```

---

## Core Concepts

### Method Chaining

All methods return `this` for chainable API:

```javascript
const result = await new ApiFeatures(Model, req.query, userRole)
  .method1()
  .method2()
  .method3()
  .execute();
```

### Recommended Execution Order

```javascript
const result = await new ApiFeatures(Model, req.query, userRole)
  .addManualFilters(serverFilters)    // 1. Add backend filters
  .filter()                            // 2. Apply URL filters
  .populate(populateOptions)           // 3. Join related data
  .search(["name", "description"])     // 4. Full-text search
  .sort()                              // 5. Order results
  .limitFields()                       // 6. Project fields
  .paginate()                          // 7. Apply pagination
  .execute();                          // 8. Run aggregation
```

### MongoDB Aggregation Pipeline

ApiFeatures internally builds a MongoDB aggregation pipeline:

```javascript
[
  { $match: { category: "electronics" } },
  { $lookup: { from: "brands", ... } },
  { $match: { $or: [...search conditions...] } },
  { $sort: { createdAt: -1 } },
  { $project: { name: 1, price: 1 } },
  { $skip: 0 },
  { $limit: 10 }
]
```

---

## API Reference

### Constructor

Creates a new ApiFeatures instance for processing a query.

```javascript
new ApiFeatures(model, query, userRole)
```

**Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `model` | Mongoose Model | ✅ Yes | - | The Mongoose model to query |
| `query` | Object | ❌ No | `{}` | Query object (typically `req.query`) |
| `userRole` | String | ❌ No | `"guest"` | User's role for security rules |

**Examples:**

```javascript
// Basic usage
const features = new ApiFeatures(User, req.query);

// With user role
const features = new ApiFeatures(Post, req.query, req.user?.role);

// With empty query
const features = new ApiFeatures(Product, {}, "admin");
```

**Properties Initialized:**

- `this.model` - The Mongoose model
- `this.query` - Copy of the input query object
- `this.pipeline` - MongoDB aggregation pipeline (empty array)
- `this.manualFilters` - Backend-enforced filters
- `this.userRole` - Determined security role
- `this.useCursor` - Cursor mode flag

---

### `filter()`

Builds MongoDB `$match` stage from query parameters.

```javascript
.filter()
```

**Returns:** `this` (for chaining)

**Features:**
- Parses query parameters into MongoDB filters
- Applies security filtering (removes forbidden fields)
- Normalizes logical operators
- Converts types intelligently
- Blocks injection attacks

**Examples:**

#### Simple Equality Filter

```javascript
// URL: GET /api/products?category=electronics&brand=apple

new ApiFeatures(Product, req.query)
  .filter()
  .execute();

// Generated $match:
// { category: "electronics", brand: "apple" }
```

#### Comparison Operators

```javascript
// URL: GET /api/products?price[gte]=100&price[lte]=500

new ApiFeatures(Product, req.query)
  .filter()
  .execute();

// Generated $match:
// { price: { $gte: 100, $lte: 500 } }
```

**Supported Operators:** `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `in`, `nin`, `regex`, `exists`, `size`, `or`, `and`

#### Multiple Values (CSV)

```javascript
// URL: GET /api/products?tags=electronics,phone,cheap

new ApiFeatures(Product, req.query)
  .filter()
  .execute();

// Generated $match:
// { tags: ["electronics", "phone", "cheap"] }
```

#### ObjectId Conversion

```javascript
// URL: GET /api/products?userId=665f0f6f4e7d9a2e2c123456

new ApiFeatures(Product, req.query)
  .filter()
  .execute();

// Generated $match:
// { userId: ObjectId("665f0f6f4e7d9a2e2c123456") }
```

**Auto-converts these fields to ObjectId:**
- `_id`, `id`
- Any field ending with `id` (e.g., `userId`, `productId`)
- Fields in `$eq`, `$ne`, `$in`, `$nin` operators

#### Type Conversions

```javascript
// URL: GET /api/products?isActive=true&price=99&code=00123&date=null

new ApiFeatures(Product, req.query)
  .filter()
  .execute();

// Generated $match:
// { isActive: true, price: 99, code: "00123", date: null }
```

**Conversion Rules:**
- `"true"` → `true`
- `"false"` → `false`
- `"null"` → `null`
- Numbers without leading zeros → `Number`
- Numbers with leading zeros → `String` (preserved)

#### Reserved Keys (Ignored)

These keys are never treated as filters:
```javascript
["page", "limit", "sort", "fields", "populate", "q"]
```

---

### `addManualFilters()`

Adds backend-enforced filters that cannot be bypassed by users.

```javascript
.addManualFilters(filters)
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `filters` | Object | MongoDB filter object to merge |

**Returns:** `this` (for chaining)

**Use Cases:**
- Restrict data by user ownership
- Enforce role-based visibility
- Add automatic status filters
- Secure multi-tenant queries

**Examples:**

#### Filter by Current User

```javascript
app.get(
  "/api/my-posts",
  catchAsync(async (req, res) => {
    const result = await new ApiFeatures(Post, req.query, req.user?.role)
      .addManualFilters({ userId: req.user._id })  // Only user's posts
      .filter()
      .execute();

    res.json(result);
  })
);
```

#### Combine Multiple Manual Filters

```javascript
const result = await new ApiFeatures(Order, req.query)
  .addManualFilters({
    userId: req.user._id,      // User's orders only
    isDeleted: false,           // Not deleted
    status: { $ne: "cancelled" } // Not cancelled
  })
  .filter()
  .execute();
```

#### Using Logical Operators

```javascript
const result = await new ApiFeatures(Product, req.query, "admin")
  .addManualFilters({
    $and: [
      { isActive: true },
      { stock: { $gt: 0 } }
    ]
  })
  .filter()
  .execute();
```

#### Nested ObjectId in Logical Operators

```javascript
const result = await new ApiFeatures(Post, req.query)
  .addManualFilters({
    $or: [
      { userId: "665f0f6f4e7d9a2e2c123456" },
      { sharedWith: "665f0f6f4e7d9a2e2c123456" }
    ]
  })
  .filter()
  .execute();

// ObjectIds in $or are automatically converted!
```

#### `$in` Operator

```javascript
const result = await new ApiFeatures(Product, req.query)
  .addManualFilters({
    _id: { $in: ["665f0f6f4e7d9a2e2c123456", "665f0f6f4e7d9a2e2c654321"] }
  })
  .filter()
  .execute();

// ObjectIds are converted automatically
```

#### `$nor` Operator

```javascript
const result = await new ApiFeatures(Product, req.query)
  .addManualFilters({
    $nor: [
      { status: "blocked" },
      { isDeleted: true }
    ]
  })
  .filter()
  .execute();
```

#### Multiple Calls (Merged)

```javascript
const result = await new ApiFeatures(Post, req.query)
  .addManualFilters({ isPublished: true })
  .addManualFilters({ userId: req.user._id })
  .filter()
  .execute();

// Both filters are merged
```

---

### `search()`

Performs case-insensitive full-text search using regex.

```javascript
.search(fields)
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `fields` | Array | Field names to search in |

**Returns:** `this` (for chaining)

**Features:**
- Case-insensitive search
- Searches multiple fields
- Uses efficient regex
- Escapes special characters
- Creates `$or` conditions

**Examples:**

#### Simple Search

```javascript
// URL: GET /api/products?q=iphone

new ApiFeatures(Product, req.query)
  .filter()
  .search(["name", "description"])
  .execute();

// Generated $or:
// {
//   $or: [
//     { name: { $regex: "iphone", $options: "i" } },
//     { description: { $regex: "iphone", $options: "i" } }
//   ]
// }
```

#### Search Multiple Fields

```javascript
// URL: GET /api/products?q=samsung

new ApiFeatures(Product, req.query)
  .filter()
  .search(["name", "brand", "model", "description", "specs"])
  .execute();
```

#### Search with Special Characters (Escaped)

```javascript
// URL: GET /api/posts?q=$pecial+char&.^

new ApiFeatures(Post, req.query)
  .search(["title", "content"])
  .execute();

// Special characters are escaped, prevents injection
```

#### No Search Query (Returns Early)

```javascript
// URL: GET /api/products

new ApiFeatures(Product, req.query)
  .search(["name", "description"])  // q not provided, skipped
  .execute();
```

#### Empty Search Fields (Skipped)

```javascript
new ApiFeatures(Product, req.query)
  .search([])  // Empty array, skipped
  .execute();
```

#### Search with Sorting

```javascript
// URL: GET /api/products?q=phone&sort=-popularity,price

new ApiFeatures(Product, req.query)
  .filter()
  .search(["name", "description"])
  .sort()
  .paginate()
  .execute();
```

#### Combined with Filter

```javascript
// URL: GET /api/products?category=electronics&q=iphone&price[lte]=1000

new ApiFeatures(Product, req.query)
  .filter()                 // category + price filters
  .search(["name", "model"]) // q search
  .execute();

// Both filter and search are combined
```

---

### `sort()`

Sorts results by one or more fields.

```javascript
.sort()
```

**Returns:** `this` (for chaining)

**Features:**
- Multi-field sorting
- `-` prefix for descending order
- Validates fields exist in schema
- Ignores invalid fields silently

**Examples:**

#### Single Field Ascending

```javascript
// URL: GET /api/products?sort=price

new ApiFeatures(Product, req.query)
  .sort()
  .execute();

// Generated $sort:
// { price: 1 }
```

#### Single Field Descending

```javascript
// URL: GET /api/products?sort=-createdAt

new ApiFeatures(Product, req.query)
  .sort()
  .execute();

// Generated $sort:
// { createdAt: -1 }
```

#### Multiple Fields

```javascript
// URL: GET /api/products?sort=-popularity,price,name

new ApiFeatures(Product, req.query)
  .sort()
  .execute();

// Generated $sort:
// { popularity: -1, price: 1, name: 1 }
```

#### Invalid Fields Ignored

```javascript
// URL: GET /api/products?sort=price,-invalidField,name

new ApiFeatures(Product, req.query)
  .sort()
  .execute();

// Generated $sort (invalidField removed):
// { price: 1, name: 1 }
```

#### No Sort (Skipped)

```javascript
// URL: GET /api/products

new ApiFeatures(Product, req.query)
  .sort()  // sort not provided, skipped
  .execute();
```

#### Sort with Pagination

```javascript
// URL: GET /api/products?sort=-rating&page=2&limit=20

new ApiFeatures(Product, req.query)
  .sort()
  .paginate()
  .execute();
```

---

### `limitFields()`

Controls which fields are returned (projection).

```javascript
.limitFields(input)
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `input` | String | Optional field list (comma-separated) |

**Returns:** `this` (for chaining)

**Features:**
- Include/exclude fields
- Automatic forbidden field blocking
- Prevents mixing include/exclude
- Always includes `_id` in include mode
- Logs attempts to access forbidden fields

**Examples:**

#### Include Specific Fields

```javascript
// URL: GET /api/products?fields=name,price,category

new ApiFeatures(Product, req.query)
  .limitFields()
  .execute();

// Generated $project:
// { name: 1, price: 1, category: 1, password: 0 }
// (password auto-excluded)
```

#### Exclude Specific Fields

```javascript
// URL: GET /api/products?fields=-password,-refreshToken

new ApiFeatures(Product, req.query)
  .limitFields()
  .execute();

// Generated $project:
// { password: 0, refreshToken: 0 }
```

#### Programmatic Include

```javascript
new ApiFeatures(Product, req.query)
  .limitFields("name,price,category")  // Pass as string
  .execute();
```

#### Programmatic Exclude

```javascript
new ApiFeatures(Product, req.query)
  .limitFields("-password,-token")
  .execute();
```

#### Auto-Include _id

```javascript
// URL: GET /api/products?fields=name,price

new ApiFeatures(Product, req.query)
  .limitFields()
  .execute();

// _id is always included:
// { _id: 1, name: 1, price: 1 }
```

#### Forbidden Fields Always Blocked

```javascript
// URL: GET /api/users?fields=name,email,password,apiKey

// securityConfig.forbiddenFields: ["password", "apiKey"]

new ApiFeatures(User, req.query)
  .limitFields()
  .execute();

// password and apiKey are blocked:
// { _id: 1, name: 1, email: 1, password: 0, apiKey: 0 }
```

#### Forbidden Field Removal in Exclude Mode

```javascript
// URL: GET /api/users?fields=-profilePicture

// securityConfig.forbiddenFields: ["password", "refreshToken"]

new ApiFeatures(User, req.query)
  .limitFields()
  .execute();

// Forbidden fields always excluded:
// { profilePicture: 0, password: 0, refreshToken: 0 }
```

#### Combined with Manual Input

```javascript
new ApiFeatures(Product, req.query)
  .limitFields("name,price")  // Manual input
  .execute();

// Even if user provides fields in URL, manual takes precedence
```

#### Invalid: Mixed Include/Exclude

```javascript
// URL: GET /api/products?fields=name,-password

new ApiFeatures(Product, req.query)
  .limitFields()
  .execute();

// Error: "Cannot mix include and exclude fields"
```

---

### `populate()`

Joins related documents using aggregation `$lookup`.

```javascript
.populate(input)
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `input` | String/Object/Array | Path(s) to populate |

**Returns:** `this` (for chaining)

**Features:**
- Single and multiple populate
- Nested populate support
- Field selection with forbidden field protection
- Dot notation support
- Role-based access control
- Preserves array relationships

**Examples:**

#### Simple Populate

```javascript
// URL: GET /api/posts?populate=user

new ApiFeatures(Post, req.query)
  .populate("user")
  .execute();

// Generated $lookup:
// { $lookup: { from: "users", localField: "userId", foreignField: "_id", as: "user" } }
// { $unwind: { path: "$user", preserveNullAndEmptyArrays: true } }
```

#### Populate from Query

```javascript
// URL: GET /api/posts?populate=user,category

new ApiFeatures(Post, req.query)
  .populate()  // No argument, uses req.query.populate
  .execute();
```

#### Multiple Paths Programmatically

```javascript
new ApiFeatures(Post, req.query)
  .populate(["user", "category", "tags"])
  .execute();
```

#### Nested Populate

```javascript
new ApiFeatures(Post, req.query)
  .populate({
    path: "user",
    populate: {
      path: "company",
      populate: {
        path: "country"
      }
    }
  })
  .execute();

// Generates nested $lookup stages
```

#### Dot Notation (Shorthand)

```javascript
new ApiFeatures(Post, req.query)
  .populate("user.company.country")
  .execute();

// Same as nested populate above, more concise
```

#### Populate with Field Selection

```javascript
new ApiFeatures(Post, req.query)
  .populate({
    path: "user",
    select: "name email profile"
  })
  .execute();

// User documents include only: _id, name, email, profile
```

#### Exclude Fields in Populate

```javascript
new ApiFeatures(Post, req.query)
  .populate({
    path: "user",
    select: "-password -refreshToken"  // Exclude sensitive fields
  })
  .execute();

// Forbidden fields are also excluded
```

#### Combine Include and Exclude (Error)

```javascript
new ApiFeatures(Post, req.query)
  .populate({
    path: "user",
    select: "name -email"  // INVALID
  })
  .execute();

// Error: "Cannot mix include and exclude in populate select"
```

#### Populate Arrays

```javascript
// Post has many comments

new ApiFeatures(Post, req.query)
  .populate({
    path: "comments",
    isArray: true
  })
  .execute();

// Handles array relationships correctly
```

#### Nested Populate with Arrays

```javascript
new ApiFeatures(Post, req.query)
  .populate({
    path: "comments",
    populate: {
      path: "author"
    }
  })
  .execute();

// Each comment's author is populated
```

#### Populate with Complex Nesting

```javascript
new ApiFeatures(Order, req.query)
  .populate([
    {
      path: "user",
      select: "name email"
    },
    {
      path: "items",
      populate: {
        path: "product",
        select: "name price",
        populate: {
          path: "category",
          select: "name"
        }
      }
    },
    {
      path: "shipping",
      select: "-notes"
    }
  ])
  .execute();
```

#### Populate from URL with Select

```javascript
// URL: GET /api/posts?populate=user&user.select=name%20email

new ApiFeatures(Post, req.query)
  .populate()  // Parses both path and select from query
  .execute();
```

#### Access Control on Populate

```javascript
// securityConfig for user role:
// allowedPopulate: ["user", "category"]

new ApiFeatures(Post, req.query, "user")
  .populate("user")       // Allowed
  .populate("category")   // Allowed
  .populate("admin")      // Silently skipped
  .execute();
```

---

### `paginate()`

Adds pagination to results.

```javascript
.paginate()
```

**Returns:** `this` (for chaining)

**Features:**
- Skip-limit pagination
- Role-based max limits
- Defaults to page 1, limit 10
- Prevents negative values
- Caps by user's access level

**Examples:**

#### Default Pagination

```javascript
// URL: GET /api/products

new ApiFeatures(Product, req.query)
  .paginate()
  .execute();

// Default: page 1, limit 10
// Generated $skip: 0, $limit: 10
```

#### Custom Page and Limit

```javascript
// URL: GET /api/products?page=2&limit=20

new ApiFeatures(Product, req.query)
  .paginate()
  .execute();

// Skip: 10, Limit: 20
```

#### Large Page Request (Capped)

```javascript
// URL: GET /api/products?limit=10000 (user is 'guest')
// securityConfig.guest.maxLimit: 50

new ApiFeatures(Product, req.query, "guest")
  .paginate()
  .execute();

// Actual limit: 50 (capped)
```

#### Admin Higher Limits

```javascript
// securityConfig.admin.maxLimit: 1000

const result = await new ApiFeatures(Product, req.query, "admin")
  .paginate()
  .execute();

// Admins can request up to 1000 items
```

#### Negative Page (Normalized)

```javascript
// URL: GET /api/products?page=-5

new ApiFeatures(Product, req.query)
  .paginate()
  .execute();

// page becomes 1 (minimum)
```

#### Pagination with Search and Sort

```javascript
// URL: GET /api/products?q=phone&sort=-rating&page=2&limit=15

new ApiFeatures(Product, req.query)
  .search(["name", "description"])
  .sort()
  .paginate()
  .execute();

// Returns page 2 with 15 items per page
```

#### Calculate Pagination Info

```javascript
const result = await new ApiFeatures(Product, req.query, "user")
  .paginate()
  .execute();

const page = parseInt(req.query.page) || 1;
const limit = parseInt(req.query.limit) || 10;
const totalPages = Math.ceil(result.count / limit);

console.log(`Page ${page} of ${totalPages} (${result.count} total)`);
```

---

### `execute()`

Runs the aggregation pipeline and returns results.

```javascript
await .execute(options)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `options` | Object | `{}` | Execution options |
| `options.debug` | Boolean | `false` | Log pipeline to console |
| `options.useCursor` | Boolean | `false` | Use aggregation cursor |
| `options.batchSize` | Number | `100` | Cursor batch size |
| `options.maxTimeMS` | Number | `10000` | Query timeout in ms |
| `options.allowDiskUse` | Boolean | `false` | Allow MongoDB disk spill |
| `options.readConcern` | String | `"majority"` | MongoDB read concern |

**Returns:** Promise resolving to result object

**Result Format:**

```javascript
{
  success: true,
  count: 25,           // Total documents matching query (before pagination)
  data: [...]          // Documents (after pagination)
}
```

**Examples:**

#### Basic Execution

```javascript
const result = await new ApiFeatures(Product, req.query)
  .filter()
  .execute();

// result = { success: true, count: 123, data: [...] }
```

#### Debug Pipeline

```javascript
const result = await new ApiFeatures(Product, req.query)
  .filter()
  .search(["name"])
  .sort()
  .execute({ debug: true });

// Logs complete pipeline to console
```

#### With Cursor for Large Results

```javascript
const result = await new ApiFeatures(Product, req.query)
  .filter()
  .execute({
    useCursor: true,
    batchSize: 500
  });

// Uses cursor for memory efficiency with large datasets
```

#### Increase Timeout

```javascript
const result = await new ApiFeatures(Post, req.query)
  .search(["content"])
  .execute({ maxTimeMS: 30000 });

// 30-second timeout for slow queries
```

#### Large Sort with Disk Usage

```javascript
const result = await new ApiFeatures(Order, req.query)
  .filter()
  .sort()
  .execute({ allowDiskUse: true });

// Allows MongoDB to use disk for large sorts
```

#### Pipeline Size Limit Check

```javascript
try {
  const result = await new ApiFeatures(Model, req.query)
    .populate(["ref1", "ref2", "ref3"])
    .execute();
} catch (err) {
  if (err.message.includes("Too many pipeline stages")) {
    // Default max: 80 stages (configurable)
  }
}
```

#### Full Options Example

```javascript
const result = await new ApiFeatures(Product, req.query, "admin")
  .addManualFilters({ store: req.user.storeId })
  .filter()
  .populate(["category", "brand"])
  .search(["name", "description"])
  .sort()
  .limitFields()
  .paginate()
  .execute({
    debug: process.env.NODE_ENV === "development",
    useCursor: true,
    batchSize: 200,
    maxTimeMS: 15000,
    allowDiskUse: true,
    readConcern: "majority"
  });

res.json(result);
```

---

## Real-World Examples

### Example 1: E-Commerce Product Listing

```javascript
import express from "express";
import ApiFeatures, { catchAsync, catchError } from "vanta-api";
import Product from "./models/Product.js";

const app = express();

app.get(
  "/api/products",
  catchAsync(async (req, res) => {
    // Filter by active products
    const result = await new ApiFeatures(Product, req.query, req.user?.role)
      .addManualFilters({ isActive: true, stock: { $gt: 0 } })
      .filter()  // User filters: category, brand, price range
      .search(["name", "description", "brand"])
      .sort()    // Sort by price, rating, date
      .limitFields()
      .paginate()
      .execute({ debug: process.env.DEBUG === "true" });

    res.json(result);
  })
);

app.use(catchError);
app.listen(3000);

// Example Requests:
// GET /api/products
// GET /api/products?category=electronics
// GET /api/products?price[gte]=100&price[lte]=500&sort=-popularity
// GET /api/products?q=iphone&fields=name,price&page=1&limit=20
```

### Example 2: Admin Dashboard with Role-Based Access

```javascript
import ApiFeatures, { catchAsync } from "vanta-api";
import Order from "./models/Order.js";

app.get(
  "/api/admin/orders",
  authMiddleware,
  adminMiddleware,
  catchAsync(async (req, res) => {
    const result = await new ApiFeatures(Order, req.query, req.user.role)
      .addManualFilters({
        $or: [
          { userId: req.user._id },  // Own orders
          { adminApproved: true }     // Public orders
        ]
      })
      .filter()
      .populate([
        { path: "user", select: "name email phone" },
        { path: "items.product", select: "name price" }
      ])
      .sort()
      .limitFields()
      .paginate()
      .execute();

    res.json(result);
  })
);
```

### Example 3: API with Multiple Resource Formats

```javascript
import ApiFeatures, { catchAsync } from "vanta-api";
import Post from "./models/Post.js";

app.get(
  "/api/posts",
  catchAsync(async (req, res) => {
    let features = new ApiFeatures(Post, req.query, req.user?.role)
      .addManualFilters({ status: "published" })
      .filter()
      .search(["title", "content"])
      .sort();

    // Different projections based on endpoint
    if (req.path.includes("/preview")) {
      features = features.limitFields("title,excerpt,thumbnail");
    } else if (req.path.includes("/detailed")) {
      features = features.limitFields();
    }

    const result = await features
      .paginate()
      .execute();

    res.json(result);
  })
);
```

### Example 4: Nested Data with Conditional Population

```javascript
import ApiFeatures, { catchAsync } from "vanta-api";
import Comment from "./models/Comment.js";

app.get(
  "/api/comments",
  catchAsync(async (req, res) => {
    let features = new ApiFeatures(Comment, req.query, req.user?.role)
      .filter();

    // Conditionally populate author data based on query
    if (req.query.includeAuthor === "true") {
      features = features.populate({
        path: "author",
        select: "-email"  // Hide email
      });
    }

    // Conditionally populate nested replies
    if (req.query.nested === "true") {
      features = features.populate({
        path: "replies",
        populate: {
          path: "author",
          select: "name avatar"
        }
      });
    }

    const result = await features
      .search(["content"])
      .sort()
      .paginate()
      .execute();

    res.json(result);
  })
);
```

### Example 5: Complex Filtering with Server-Side Rules

```javascript
import ApiFeatures, { catchAsync } from "vanta-api";
import Article from "./models/Article.js";

app.get(
  "/api/articles",
  catchAsync(async (req, res) => {
    // Build dynamic server-side filters
    let manualFilters = { status: "published" };

    // Role-based visibility
    if (req.user?.role === "subscriber") {
      manualFilters.visibility = { $in: ["public", "subscriber"] };
    } else if (req.user?.role === "premium") {
      manualFilters.visibility = { $in: ["public", "subscriber", "premium"] };
    } else {
      manualFilters.visibility = "public";
    }

    const result = await new ApiFeatures(Article, req.query, req.user?.role)
      .addManualFilters(manualFilters)
      .filter()
      .populate({
        path: "author",
        select: "name bio avatar"
      })
      .search(["title", "content", "author.name"])
      .sort()
      .limitFields()
      .paginate()
      .execute();

    res.json(result);
  })
);
```

---

## Error Handling

### Using `catchAsync`

Automatically catches async errors:

```javascript
import { catchAsync, HandleERROR } from "vanta-api";

app.get(
  "/api/products/:id",
  catchAsync(async (req, res, next) => {
    const product = await Product.findById(req.params.id);
    if (!product) {
      throw new HandleERROR("Product not found", 404);
    }
    res.json(product);
  })
);
```

### Global Error Middleware

Must be registered last:

```javascript
import { catchError } from "vanta-api";

// All routes here...

// Error middleware (must be last)
app.use(catchError);
```

### Error Response Format

```json
{
  "status": "fail",
  "success": false,
  "message": "Invalid field in query"
}
```

### Custom Error Handler

```javascript
app.use((err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  const status = err.status || "error";

  res.status(statusCode).json({
    status,
    success: false,
    message: err.message,
    ...(process.env.NODE_ENV === "development" && { stack: err.stack })
  });
});
```

---

## Security Configuration

### Default Configuration

```javascript
// src/security-default-config.js
export const securityConfig = {
  allowedOperators: [
    "eq", "ne", "gt", "gte",
    "lt", "lte", "in", "nin",
    "regex", "exists", "size", "or", "and"
  ],
  forbiddenFields: ["password"],
  maxPipelineStages: 80,
  accessLevels: {
    guest: { maxLimit: 50, allowedPopulate: ["*"] },
    user: { maxLimit: 100, allowedPopulate: ["*"] },
    admin: { maxLimit: 1000, allowedPopulate: ["*"] }
  }
};
```

### Custom Configuration

Create `security-config.js` in project root:

```javascript
export const securityConfig = {
  allowedOperators: [
    "eq", "ne", "gt", "gte", "lt", "lte",
    "in", "nin", "regex", "exists"
  ],
  forbiddenFields: [
    "password",
    "refreshToken",
    "resetPasswordToken",
    "apiKey",
    "secretKey"
  ],
  maxPipelineStages: 100,
  accessLevels: {
    guest: {
      maxLimit: 20,
      allowedPopulate: ["category", "brand"]
    },
    user: {
      maxLimit: 100,
      allowedPopulate: ["*"]
    },
    moderator: {
      maxLimit: 500,
      allowedPopulate: ["*"]
    },
    admin: {
      maxLimit: 5000,
      allowedPopulate: ["*"]
    },
    superAdmin: {
      maxLimit: 10000,
      allowedPopulate: ["*"]
    }
  }
};
```

### Configuration Options

| Option | Type | Description |
|--------|------|-------------|
| `allowedOperators` | Array | MongoDB operators allowed in queries |
| `forbiddenFields` | Array | Fields automatically blocked from all responses |
| `maxPipelineStages` | Number | Maximum aggregation pipeline stages |
| `accessLevels` | Object | Role-based access definitions |

### Forbidden Fields Example

```javascript
forbiddenFields: ["password", "apiKey", "refreshToken", "creditCard"]

// All queries will automatically exclude these fields
// Even if user explicitly requests: ?fields=name,password
// Result: password is still excluded
```

---

## Best Practices

### 1. Always Use Manual Filters for Security

```javascript
// ✅ GOOD - Secure
const result = await new ApiFeatures(Post, req.query)
  .addManualFilters({ userId: req.user._id })  // User can only see own posts
  .filter()
  .execute();

// ❌ BAD - Insecure
const result = await new ApiFeatures(Post, req.query)
  .filter()  // User could filter other users' posts
  .execute();
```

### 2. Chain Methods in Recommended Order

```javascript
// ✅ GOOD
const result = await new ApiFeatures(Model, req.query)
  .addManualFilters(backendFilters)
  .filter()
  .populate(populateOptions)
  .search(searchFields)
  .sort()
  .limitFields()
  .paginate()
  .execute();
```

### 3. Validate User Input

```javascript
// ✅ GOOD
if (!Array.isArray(req.query.ids) || req.query.ids.length > 100) {
  throw new HandleERROR("Invalid request", 400);
}

const result = await new ApiFeatures(Product, req.query)
  .addManualFilters({ _id: { $in: req.query.ids } })
  .execute();
```

### 4. Use Forbidden Fields for Sensitive Data

```javascript
// ✅ GOOD
export const securityConfig = {
  forbiddenFields: [
    "password",
    "refreshToken",
    "apiKey",
    "ssn",
    "creditCard"
  ]
};
```

### 5. Limit Pagination for Performance

```javascript
// ✅ GOOD
accessLevels: {
  guest: { maxLimit: 50 },      // 50 items max
  user: { maxLimit: 100 },      // 100 items max
  admin: { maxLimit: 1000 }     // 1000 items max
}
```

### 6. Use Debug Mode in Development Only

```javascript
// ✅ GOOD
const result = await features.execute({
  debug: process.env.NODE_ENV === "development"
});
```

### 7. Handle Errors Gracefully

```javascript
// ✅ GOOD
try {
  const result = await features.execute();
  res.json(result);
} catch (err) {
  // catchAsync will forward to error middleware
  throw err;
}
```

### 8. Optimize Populate Paths

```javascript
// ✅ GOOD - Selective fields
.populate({
  path: "user",
  select: "name email -_id"  // Only needed fields
})

// ❌ BAD - All fields
.populate("user")  // Includes all user fields
```

---

## API Exports

```javascript
import ApiFeatures from "vanta-api";           // Main class
import { catchAsync } from "vanta-api";        // Async wrapper
import { catchError } from "vanta-api";        // Error middleware
import { HandleERROR } from "vanta-api";       // Error class
```

---

## License

MIT © Alireza Aghaee

---

## Support

For issues, feature requests, or contributions, visit the repository.
