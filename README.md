# Vanta API (vanta-api) 🚀

Advanced API features and security configuration for Node.js & MongoDB/Mongoose. 
(ویژگی‌های پیشرفته API و پیکربندی امنیتی برای Node.js و MongoDB)

[English Documentation](#english-documentation) | [مستندات فارسی](#persian-documentation)

---

<h2 id="english-documentation">🇬🇧 English Documentation</h2>

### 📦 Installation
```bash
npm install vanta-api
```
*Note: A default `security-config.js` file will be automatically generated in your project root upon installation.*

### 🚀 Getting Started

Vanta API operates by converting standard HTTP queries into powerful MongoDB Aggregation pipelines, taking care of security, type-casting, and deep relations.

```javascript
import express from 'express';
import ApiFeatures, { catchAsync, catchError } from 'vanta-api';
import User from './models/User.js';

const app = express();

app.get('/users', catchAsync(async (req, res, next) => {
  // 1. Initialize ApiFeatures with your Mongoose Model, the req.query, and the user's role
  const features = new ApiFeatures(User, req.query, 'admin')
    .filter()
    .search(['name', 'email', 'bio'])
    .sort()
    .limitFields()
    .populate('role,profile')
    .paginate();

  // 2. Execute the aggregation pipeline
  const result = await features.execute();

  res.status(200).json({
    success: true,
    count: result.count,
    data: result.data // Will be result.cursor if options.useCursor = true
  });
}));

app.use(catchError); // Global Error Handler
```

### 🛠️ API Features & Methods

#### 1. `filter()`
Filters documents based on the URL query string. Automatically handles Mongoose operators (`$eq`, `$gt`, `$in`, etc.) and casts values to `Numbers` or `ObjectIds` appropriately.

**URL Examples:**
- `GET /users?status=active` (Exact match)
- `GET /users?age[gte]=18&age[lt]=30` (Age >= 18 and Age < 30)
- `GET /users?categoryIds=id1,id2` (Matches documents where category is id1 OR id2 - `$in`)

**Code:**
```javascript
features.filter();
```

#### 2. `search(fields)`
Performs a case-insensitive Regex search across the specified fields when the `q` parameter is present in the URL.

**URL Example:**
- `GET /users?q=john`

**Code:**
```javascript
features.search(['firstName', 'lastName', 'email']);
```

#### 3. `sort()`
Sorts the documents based on the `sort` query parameter. Use a minus `-` prefix for descending order. Supports sorting on nested fields.

**URL Examples:**
- `GET /users?sort=createdAt` (Ascending)
- `GET /users?sort=-price,rating` (Descending price, ascending rating)

**Code:**
```javascript
features.sort();
```

#### 4. `limitFields()`
Selects or excludes specific fields to be returned in the response (Projection). It strictly respects `forbiddenFields` from your security config (e.g., passwords are never returned).

**URL Examples:**
- `GET /users?fields=name,email` (Only return name and email)
- `GET /users?fields=-phone,-address` (Return everything EXCEPT phone and address)

**Code:**
```javascript
features.limitFields();
```

#### 5. `paginate()`
Paginates the results using `page` and `limit` query parameters. Limits are bounded by the `maxLimit` defined in the security configuration for the current user role.

**URL Example:**
- `GET /users?page=2&limit=20`

**Code:**
```javascript
features.paginate();
```

#### 6. `populate(input)`
Simulates Mongoose's `.populate()` inside the aggregation pipeline. Supports deep nesting and specific field selection for related documents. Arrays and Objects are correctly populated.

**URL Example:**
- `GET /users?populate=comments,profile`

**Code:**
```javascript
// You can populate from the URL or hardcode it in the backend:
features.populate([
  { path: 'author', select: 'name avatar' },
  { path: 'comments', populate: { path: 'likes' } }
]);
```

#### 7. `addManualFilters(filters)`
Allows you to forcefully inject backend-side filters that the client cannot override.

**Code:**
```javascript
features.addManualFilters({ 
  isDeleted: false,
  tenantId: req.user.tenantId
});
```

#### 8. `execute(options)`
Runs the aggregation pipeline and fetches the total document count.
Supports standard execution or optimized cursors for massive datasets.

**Code:**
```javascript
const result = await features.execute({
  useCursor: false, // Set to true to return a MongoDB cursor instead of an array
  batchSize: 100,
  allowDiskUse: true
});
// result.data contains the documents
// result.count contains the total number of matched documents
```

### ⚙️ Security Configuration (`security-config.js`)
At the root of your project, you can configure roles, forbidden fields, and pipeline limits:
```javascript
export const securityConfig = {
  forbiddenFields: ["password", "resetToken"],
  accessLevels: {
    guest: { maxLimit: 20, allowedPopulate: ["category"] },
    user: { maxLimit: 50, allowedPopulate: ["*"] },
    admin: { maxLimit: 1000, allowedPopulate: ["*"] },
  }
};
```

---

<h2 id="persian-documentation">🇮🇷 مستندات فارسی (Persian Documentation)</h2>

### 📦 نصب و راه‌اندازی
```bash
npm install vanta-api
```
*نکته: پس از نصب، فایل `security-config.js` به صورت خودکار در روت پروژه شما ساخته می‌شود تا بتوانید تنظیمات امنیتی را شخصی‌سازی کنید.*

### 🚀 شروع کار (نحوه استفاده)

پکیج Vanta API کوئری‌های ساده HTTP را دریافت کرده و آن‌ها را به Aggregation Pipeline های قدرتمند MongoDB تبدیل می‌کند و همزمان امنیت، تبدیل نوع داده‌ها (Type-Casting) و روابط تو در تو را مدیریت می‌کند.

```javascript
import express from 'express';
import ApiFeatures, { catchAsync, catchError } from 'vanta-api';
import User from './models/User.js';

const app = express();

app.get('/users', catchAsync(async (req, res, next) => {
  // ۱. کلاس ApiFeatures را با مدل مانگوس، کوئری‌های URL و نقش کاربر مقداردهی کنید
  const features = new ApiFeatures(User, req.query, 'admin')
    .filter()
    .search(['name', 'email', 'bio'])
    .sort()
    .limitFields()
    .populate('role,profile')
    .paginate();

  // ۲. پایپ‌لاین ساخته شده را اجرا کنید
  const result = await features.execute();

  res.status(200).json({
    success: true,
    count: result.count,
    data: result.data
  });
}));

app.use(catchError); // میدلور مدیریت خطاهای سراسری
```

### 🛠️ متدها و ویژگی‌ها

#### ۱. متد `filter()`
داکیومنت‌ها را بر اساس پارامترهای URL فیلتر می‌کند. به صورت خودکار اپراتورهای مانگوس (مثل `$gt`, `$in`) را تشخیص داده و رشته‌ها را به عدد یا `ObjectId` تبدیل می‌کند.

**نمونه‌های URL:**
- `GET /users?status=active` (جستجوی دقیق)
- `GET /users?age[gte]=18&age[lt]=30` (سن بزرگتر مساوی ۱۸ و کمتر از ۳۰)
- `GET /users?categoryIds=id1,id2` (یافتن رکوردهایی که دسته‌بندی آن‌ها id1 یا id2 باشد)

**نحوه استفاده در کد:**
```javascript
features.filter();
```

#### ۲. متد `search(fields)`
در صورتی که پارامتر `q` در URL وجود داشته باشد، یک جستجوی متنی (Regex) بدون حساسیت به حروف کوچک و بزرگ روی فیلدهای مشخص شده انجام می‌دهد.

**نمونه URL:**
- `GET /users?q=john`

**نحوه استفاده در کد:**
```javascript
features.search(['firstName', 'lastName', 'email']);
```

#### ۳. متد `sort()`
داکیومنت‌ها را بر اساس پارامتر `sort` مرتب می‌کند. برای مرتب‌سازی نزولی (برعکس) از علامت منفی `-` استفاده کنید.

**نمونه‌های URL:**
- `GET /users?sort=createdAt` (صعودی - از قدیم به جدید)
- `GET /users?sort=-price,rating` (مرتب‌سازی نزولی قیمت، و سپس صعودی امتیاز)

**نحوه استفاده در کد:**
```javascript
features.sort();
```

#### ۴. متد `limitFields()`
فیلدهای خاصی را برای بازگرداندن در خروجی انتخاب یا حذف می‌کند. این متد به شدت به فیلدهای ممنوعه (`forbiddenFields`) در تنظیمات امنیتی شما پایبند است (مثلاً پسورد تحت هیچ شرایطی برگردانده نمی‌شود).

**نمونه‌های URL:**
- `GET /users?fields=name,email` (فقط نام و ایمیل را برگردان)
- `GET /users?fields=-phone,-address` (همه چیز را برگردان به جز شماره تماس و آدرس)

**نحوه استفاده در کد:**
```javascript
features.limitFields();
```

#### ۵. متد `paginate()`
نتایج را با استفاده از پارامترهای `page` و `limit` صفحه‌بندی می‌کند. محدودیت‌ها هرگز نمی‌توانند از سقف تعیین شده (`maxLimit`) برای نقش آن کاربر در فایل کانفیگ فراتر بروند.

**نمونه URL:**
- `GET /users?page=2&limit=20` (صفحه دوم، ۲۰ رکورد)

**نحوه استفاده در کد:**
```javascript
features.paginate();
```

#### ۶. متد `populate(input)`
رفتار متد `.populate()` مانگوس را در داخل Aggregation شبیه‌سازی می‌کند. از روابط عمیق (Nested) و انتخاب فیلدهای خاص در جداول دیگر پشتیبانی می‌کند.

**نمونه URL:**
- `GET /users?populate=comments,profile`

**نحوه استفاده در کد:**
```javascript
// می‌توانید مقادیر را از URL بگیرید یا به صورت دستی در کد وارد کنید:
features.populate([
  { path: 'author', select: 'name avatar' },
  { path: 'comments', populate: { path: 'likes' } }
]);
```

#### ۷. متد `addManualFilters(filters)`
به شما اجازه می‌دهد فیلترهای هاردکد شده در سمت بک‌اند را به کوئری تزریق کنید. کاربر نمی‌تواند این فیلترها را از طریق URL دور بزند.

**نحوه استفاده در کد:**
```javascript
features.addManualFilters({ 
  isDeleted: false,
  tenantId: req.user.tenantId
});
```

#### ۸. متد `execute(options)`
پایپ‌لاین نهایی را در دیتابیس اجرا کرده و همزمان تعداد کل رکوردهای یافت شده (Count) را محاسبه می‌کند.
در داده‌های بسیار عظیم، از Cursor ها پشتیبانی می‌کند تا سرور دچار افت سرعت یا پر شدن حافظه نشود.

**نحوه استفاده در کد:**
```javascript
const result = await features.execute({
  useCursor: false, // برای دریافت مستقیم آبجکت کرسر این مقدار را true کنید
  batchSize: 100,
  allowDiskUse: true
});
// داکیومنت‌ها در result.data قرار می‌گیرند
// تعداد کل رکوردها در result.count است
```

### ⚙️ تنظیمات امنیتی (`security-config.js`)
در روت پروژه شما یک فایل کانفیگ وجود دارد که می‌توانید در آن سطوح دسترسی، فیلدهای ممنوعه و محدودیت‌ها را تعریف کنید:
```javascript
export const securityConfig = {
  // فیلدهایی که هرگز نباید به کاربر کلاینت ارسال شوند
  forbiddenFields: ["password", "resetToken"],
  accessLevels: {
    guest: { maxLimit: 20, allowedPopulate: ["category"] }, // مهمان
    user: { maxLimit: 50, allowedPopulate: ["*"] }, // کاربر عادی
    admin: { maxLimit: 1000, allowedPopulate: ["*"] }, // ادمین
  }
};
```
