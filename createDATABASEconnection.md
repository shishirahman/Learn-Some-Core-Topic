

## Step 1: Install Dependencies

```bash
npm install mongoose
npm install -D @types/node tsx

# or
yarn add mongoose
yarn add -D @types/node tsx
```

**Why each package?**
- `mongoose`: MongoDB ODM (Object Data Modeling) library - provides schema validation, type safety, and cleaner API
- `@types/node`: TypeScript types for Node.js (for `process`, `global`, etc.)
- `tsx`: TypeScript executor for running seed scripts

---

## Step 2: Environment Variables

Create `.env.local`:

```env
MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/your-database-name?retryWrites=true&w=majority
MONGODB_DB=next_isr_blog
NODE_ENV=development
```

**Why this format?**
- `MONGODB_URI`: Full connection string with authentication
- `MONGODB_DB`: Database name (optional, can be in URI)
- `NODE_ENV`: Tells us if we're in development (with HMR) or production

---

## Step 3: Create Connection Utility `lib/mongodb.ts`

```typescript
import mongoose from 'mongoose';

// 1. GET THE CONNECTION STRING
// ==============================
// Ensure the URI exists in environment variables
if (!process.env.MONGODB_URI) {
  throw new Error('Please add your Mongo URI to .env.local');
}

const MONGODB_URI: string = process.env.MONGODB_URI;

// 2. DEFINE CACHE STRUCTURE
// ==========================
// This interface describes what we're caching
interface MongooseCache {
  conn: typeof mongoose | null;  // The actual mongoose instance
  promise: Promise<typeof mongoose> | null;  // The connection promise
}

// 3. EXTEND GLOBAL TYPE
// ======================
// Tell TypeScript that global can have a mongoose property
declare global {
  var mongoose: MongooseCache | undefined;
}

// 4. GET OR CREATE CACHE
// =======================
// If global.mongoose exists (from previous HMR), use it
// Otherwise, create a new cache object
let cached: MongooseCache = global.mongoose || { 
  conn: null, 
  promise: null 
};

// 5. STORE CACHE GLOBALLY (FIRST TIME ONLY)
// ==========================================
// If this is the first run, save our cache to global
// This makes it survive Hot Module Reloads
if (!global.mongoose) {
  global.mongoose = cached;
}

// 6. MAIN CONNECTION FUNCTION
// ============================
async function connectDB(): Promise<typeof mongoose> {
  // ✅ FAST PATH: If we already have a connection, return it immediately
  if (cached.conn) {
    console.log('🔄 Using cached database connection');
    return cached.conn;
  }

  // ✅ PREVENT DUPLICATE CONNECTIONS: If connection is in progress, wait for it
  if (!cached.promise) {
    console.log('🔌 Creating new database connection...');
    
    const opts = {
      bufferCommands: false, // Disable mongoose buffering (fail fast if not connected)
    };

    // Start connecting and store the promise
    cached.promise = mongoose.connect(MONGODB_URI, opts).then((mongoose) => {
      console.log('✅ Database connected successfully');
      return mongoose;
    });
  }

  // Wait for the connection to complete
  try {
    cached.conn = await cached.promise;
  } catch (e) {
    // If connection fails, clear the promise so we can retry
    cached.promise = null;
    console.error('❌ Database connection failed:', e);
    throw e;
  }

  return cached.conn;
}

export default connectDB;
```

### **Why Each Part?**

#### **Part 1: Get Connection String**
```typescript
if (!process.env.MONGODB_URI) {
  throw new Error('Please add your Mongo URI to .env.local');
}
```
- **Purpose**: Fail fast if environment variable is missing
- **Why**: Better to crash at startup than get cryptic errors later

#### **Part 2: Define Cache Structure**
```typescript
interface MongooseCache {
  conn: typeof mongoose | null;
  promise: Promise<typeof mongoose> | null;
}
```
- **`conn`**: Stores the actual mongoose connection once established
- **`promise`**: Stores the connection promise while connecting
- **Why both?**: 
  - `promise` prevents duplicate connection attempts
  - `conn` is the final resolved connection

#### **Part 3: Extend Global Type**
```typescript
declare global {
  var mongoose: MongooseCache | undefined;
}
```
- **Purpose**: Tell TypeScript that `global` can have a `mongoose` property
- **Why**: TypeScript doesn't allow arbitrary properties on `global` without declaration

#### **Part 4: Get or Create Cache**
```typescript
let cached: MongooseCache = global.mongoose || { conn: null, promise: null };
```
- **First time**: `global.mongoose` is `undefined`, so create new cache
- **After HMR**: `global.mongoose` exists, so reuse it
- **Why**: This is the magic that preserves connections across HMR

#### **Part 5: Store Cache Globally**
```typescript
if (!global.mongoose) {
  global.mongoose = cached;
}
```
- **Purpose**: Save cache to `global` so it survives module reloads
- **Why only if empty?**: If it already exists, we're reusing it from HMR

#### **Part 6: Connection Function - Fast Path**
```typescript
if (cached.conn) {
  return cached.conn;
}
```
- **What**: If connection exists, return immediately
- **Why**: Avoid unnecessary async operations
- **Performance**: Returns in microseconds instead of milliseconds

#### **Part 7: Prevent Duplicate Connections**
```typescript
if (!cached.promise) {
  cached.promise = mongoose.connect(MONGODB_URI, opts);
}
```
- **Scenario**: Two API routes call `connectDB()` simultaneously
- **Without this**: Two connections start
- **With this**: Second call waits for first connection promise
- **Result**: Only one connection is made

#### **Part 8: Error Handling**
```typescript
try {
  cached.conn = await cached.promise;
} catch (e) {
  cached.promise = null; // ← CRITICAL
  throw e;
}
```
- **Why clear promise?**: If connection fails, we want to retry next time
- **Without clearing**: Failed promise would block all future attempts

---

## Step 4: Create Mongoose Schema and Model

Create `models/Post.ts`:

```typescript
import mongoose, { Schema, Model, Document } from 'mongoose';

// 1. DEFINE TYPESCRIPT INTERFACE
// ================================
// This is for TypeScript type checking
export interface IPost {
  slug: string;
  title: string;
  content: string;
  author: string;
  createdAt?: Date;
  updatedAt?: Date;
}

// 2. EXTEND WITH DOCUMENT
// ========================
// IPostDocument includes MongoDB properties like _id
export interface IPostDocument extends IPost, Document {
  _id: string;
  createdAt: Date;
  updatedAt: Date;
}

// 3. CREATE MONGOOSE SCHEMA
// ==========================
// This defines validation rules and database structure
const PostSchema: Schema<IPostDocument> = new Schema(
  {
    slug: {
      type: String,
      required: [true, 'Slug is required'],
      unique: true, // Ensures no duplicate slugs
      trim: true, // Removes whitespace
      lowercase: true, // Converts to lowercase
      match: [/^[a-z0-9-]+$/, 'Slug can only contain lowercase letters, numbers, and hyphens'],
    },
    title: {
      type: String,
      required: [true, 'Title is required'],
      trim: true,
      maxlength: [200, 'Title cannot exceed 200 characters'],
    },
    content: {
      type: String,
      required: [true, 'Content is required'],
    },
    author: {
      type: String,
      required: [true, 'Author is required'],
      trim: true,
    },
  },
  {
    // Schema options
    timestamps: true, // Automatically adds createdAt and updatedAt
    collection: 'posts', // Explicit collection name (default would be 'posts' anyway)
  }
);

// 4. ADD INDEXES FOR PERFORMANCE
// ================================
// Index on slug for fast lookups
PostSchema.index({ slug: 1 });

// Index on createdAt for sorting by date
PostSchema.index({ createdAt: -1 });

// 5. CREATE OR REUSE MODEL
// =========================
// In development with HMR, models can be registered multiple times
// This prevents "OverwriteModelError"
const Post: Model<IPostDocument> = 
  mongoose.models.Post || mongoose.model<IPostDocument>('Post', PostSchema);

export default Post;
```

### **Why Each Part?**

#### **Part 1: TypeScript Interface**
```typescript
export interface IPost {
  slug: string;
  title: string;
  // ...
}
```
- **Purpose**: Type safety in your application code
- **Why**: TypeScript can catch errors before runtime
- **Example**: `post.titl` would be caught (typo)

#### **Part 2: Document Extension**
```typescript
export interface IPostDocument extends IPost, Document {
  _id: string;
  createdAt: Date;
  updatedAt: Date;
}
```
- **What it adds**: MongoDB-specific properties
- **Why**: When you fetch from DB, you get `_id`, `createdAt`, etc.
- **Type safety**: You can use `post._id` without TypeScript errors

#### **Part 3: Mongoose Schema**
```typescript
const PostSchema: Schema<IPostDocument> = new Schema({
  slug: {
    type: String,
    required: true,
    unique: true,
    // ...
  }
})
```
- **Purpose**: Defines how data is validated and stored
- **Why Mongoose vs raw MongoDB?**:
  - ✅ Automatic validation
  - ✅ Type casting
  - ✅ Middleware (pre/post hooks)
  - ✅ Better error messages

**Validation Example**:
```typescript
// ❌ This would fail validation
await Post.create({ title: 'Test' }); // Missing required 'slug'
// Error: Post validation failed: slug: Slug is required

// ✅ This would pass
await Post.create({ 
  slug: 'test-post',
  title: 'Test Post',
  content: 'Content here',
  author: 'John'
});
```

#### **Part 4: Indexes**
```typescript
PostSchema.index({ slug: 1 });
PostSchema.index({ createdAt: -1 });
```
- **Why indexes?**: Speed up queries
- **`{ slug: 1 }`**: Ascending index on slug
- **`{ createdAt: -1 }`**: Descending index (newest first)

**Performance impact**:
```typescript
// Without index: Scans all documents (slow)
await Post.findOne({ slug: 'hello-isr' }); // 500ms for 1M docs

// With index: Direct lookup (fast)
await Post.findOne({ slug: 'hello-isr' }); // 5ms for 1M docs
```

#### **Part 5: Model Creation with HMR Protection**
```typescript
const Post: Model<IPostDocument> = 
  mongoose.models.Post || mongoose.model<IPostDocument>('Post', PostSchema);
```
- **Why `mongoose.models.Post ||`?**: 
  - First time: `mongoose.models.Post` is undefined, create model
  - After HMR: `mongoose.models.Post` exists, reuse it
- **Without this**: "OverwriteModelError" on every file save

---

## Step 5: Create Database Helper `lib/db.ts`

```typescript
import connectDB from './mongodb';
import Post, { IPostDocument } from '@/models/Post';

// 1. GET DATABASE CONNECTION
// ===========================
export async function getDatabase() {
  const mongoose = await connectDB();
  return mongoose.connection.db;
}

// 2. GET POST MODEL (MONGOOSE WAY)
// =================================
// This is simpler with Mongoose - no need for separate collection getter
export async function getPostsCollection() {
  await connectDB(); // Ensure connection exists
  return Post; // Return the Mongoose model
}

// 3. HELPER FUNCTIONS FOR COMMON OPERATIONS
// ==========================================

/**
 * Get all posts sorted by creation date (newest first)
 */
export async function getAllPosts(): Promise<IPostDocument[]> {
  await connectDB();
  return Post.find({})
    .sort({ createdAt: -1 })
    .lean() // Returns plain JavaScript objects (faster)
    .exec();
}

/**
 * Get a single post by slug
 */
export async function getPostBySlug(slug: string): Promise<IPostDocument | null> {
  await connectDB();
  return Post.findOne({ slug })
    .lean()
    .exec();
}

/**
 * Create a new post
 */
export async function createPost(postData: Partial<IPostDocument>): Promise<IPostDocument> {
  await connectDB();
  const post = await Post.create(postData);
  return post.toObject(); // Convert to plain object
}

/**
 * Update a post by slug
 */
export async function updatePost(
  slug: string, 
  updates: Partial<IPostDocument>
): Promise<IPostDocument | null> {
  await connectDB();
  return Post.findOneAndUpdate(
    { slug },
    { ...updates, updatedAt: new Date() },
    { new: true, runValidators: true } // Return updated doc & validate
  )
    .lean()
    .exec();
}

/**
 * Delete a post by slug
 */
export async function deletePost(slug: string): Promise<boolean> {
  await connectDB();
  const result = await Post.deleteOne({ slug });
  return result.deletedCount > 0;
}
```

### **Why Each Part?**

#### **Part 1: Get Database Connection**
```typescript
export async function getDatabase() {
  const mongoose = await connectDB();
  return mongoose.connection.db;
}
```
- **Purpose**: Access raw MongoDB database if needed
- **When to use**: For operations not covered by Mongoose models
- **Most of the time**: You'll use models instead

#### **Part 2: Get Model (Mongoose Way)**
```typescript
export async function getPostsCollection() {
  await connectDB();
  return Post;
}
```
- **Difference from raw MongoDB**: Returns Model, not Collection
- **Why simpler**: Mongoose models include connection info
- **Usage**: `const PostModel = await getPostsCollection(); await PostModel.find()`

#### **Part 3: Helper Functions**

**Why create helpers?**
- ✅ **Reusability**: Use same code everywhere
- ✅ **Consistency**: Always connect before querying
- ✅ **Type Safety**: Strong TypeScript types
- ✅ **Cleaner**: API routes become simpler

**`.lean()` explanation**:
```typescript
// Without .lean() - Returns Mongoose Document
const post = await Post.findOne({ slug });
// post has methods: post.save(), post.remove(), etc.
// Heavier in memory

// With .lean() - Returns plain JavaScript object
const post = await Post.findOne({ slug }).lean();
// post is just { _id, slug, title, ... }
// Faster, lighter, perfect for read-only operations
```

---

## Step 6: Test Connection Script

Create `scripts/test-connection.ts`:

```typescript
// Import dotenv to load .env.local
import dotenv from 'dotenv';
import { resolve } from 'path';

// Load environment variables
dotenv.config({ path: resolve(process.cwd(), '.env.local') });

import connectDB from '../lib/mongodb';
import mongoose from 'mongoose';

async function testConnection() {
  try {
    console.log('🔌 Attempting to connect to MongoDB...');
    console.log(`📍 URI: ${process.env.MONGODB_URI?.substring(0, 30)}...`);
    
    // Connect to database
    await connectDB();
    
    console.log('✅ Successfully connected to MongoDB!');
    console.log(`📦 Database: ${mongoose.connection.db.databaseName}`);
    
    // List all collections
    const collections = await mongoose.connection.db.listCollections().toArray();
    console.log(`📚 Collections (${collections.length}):`);
    collections.forEach((col) => {
      console.log(`   - ${col.name}`);
    });
    
    // Get connection stats
    const stats = await mongoose.connection.db.stats();
    console.log(`\n📊 Database Stats:`);
    console.log(`   - Collections: ${stats.collections}`);
    console.log(`   - Documents: ${stats.objects}`);
    console.log(`   - Data Size: ${(stats.dataSize / 1024 / 1024).toFixed(2)} MB`);
    
  } catch (error) {
    console.error('❌ Connection failed:');
    if (error instanceof Error) {
      console.error(`   ${error.message}`);
    }
    process.exit(1);
  } finally {
    // Close connection
    await mongoose.connection.close();
    console.log('\n🔌 Connection closed');
    process.exit(0);
  }
}

testConnection();
```

### **Why Each Part?**

#### **Environment Loading**
```typescript
dotenv.config({ path: resolve(process.cwd(), '.env.local') });
```
- **Why**: Scripts don't auto-load `.env.local` like Next.js does
- **Must run before imports**: `connectDB` needs `process.env.MONGODB_URI`

#### **Error Handling**
```typescript
try {
  // ...
} catch (error) {
  // ...
  process.exit(1); // Non-zero = error
} finally {
  await mongoose.connection.close();
  process.exit(0); // Zero = success
}
```
- **Why exit codes?**: Tell terminal if script succeeded
- **Why close connection?**: Script would hang without it
- **Why finally?**: Always close, even on error

**Run it**:
```bash
npx tsx scripts/test-connection.ts
```

---

## Step 7: Seed Script

Create `scripts/seed.ts`:

```typescript
import dotenv from 'dotenv';
import { resolve } from 'path';

dotenv.config({ path: resolve(process.cwd(), '.env.local') });

import connectDB from '../lib/mongodb';
import Post from '../models/Post';
import mongoose from 'mongoose';

const samplePosts = [
  {
    slug: 'hello-isr',
    title: 'Hello ISR!',
    content: 'This is our first post demonstrating Incremental Static Regeneration. ISR allows us to update this content without rebuilding the entire site.',
    author: 'John Doe',
  },
  {
    slug: 'understanding-isr',
    title: 'Understanding ISR',
    content: 'ISR combines the best of static and dynamic. Pages are pre-rendered but can be updated in the background based on a revalidation period.',
    author: 'Jane Smith',
  },
  {
    slug: 'isr-benefits',
    title: 'Benefits of ISR',
    content: 'ISR provides fast page loads, SEO benefits, and the ability to update content without redeployment. Perfect for blogs, e-commerce, and content sites.',
    author: 'Bob Johnson',
  },
];

async function seed() {
  try {
    console.log('🌱 Starting database seed...');
    
    // Connect to database
    await connectDB();
    console.log('✅ Connected to database');
    
    // Clear existing posts
    const deleteResult = await Post.deleteMany({});
    console.log(`🗑️  Deleted ${deleteResult.deletedCount} existing posts`);
    
    // Insert sample posts
    const insertedPosts = await Post.insertMany(samplePosts);
    console.log(`✅ Inserted ${insertedPosts.length} new posts`);
    
    // Display all posts
    const allPosts = await Post.find({}).sort({ createdAt: -1 });
    console.log('\n📝 Current posts in database:');
    allPosts.forEach((post, index) => {
      console.log(`   ${index + 1}. "${post.title}" by ${post.author}`);
      console.log(`      Slug: /${post.slug}`);
      console.log(`      Created: ${post.createdAt.toLocaleString()}`);
    });
    
    console.log('\n🎉 Seed completed successfully!');
    
  } catch (error) {
    console.error('❌ Seed failed:');
    if (error instanceof Error) {
      console.error(`   ${error.message}`);
    }
    process.exit(1);
  } finally {
    await mongoose.connection.close();
    console.log('🔌 Connection closed');
    process.exit(0);
  }
}

seed();
```

### **Why Mongoose vs Raw MongoDB?**

**Raw MongoDB (your original)**:
```typescript
const posts = await getPostsCollection();
await posts.deleteMany({});
await posts.insertMany(samplePosts);
```

**Mongoose**:
```typescript
await Post.deleteMany({});
await Post.insertMany(samplePosts);
```

**Mongoose advantages**:
- ✅ **Validation**: Checks all fields before inserting
- ✅ **Timestamps**: Auto-adds `createdAt`/`updatedAt`
- ✅ **Type Safety**: TypeScript knows the structure
- ✅ **Cleaner**: No need to get collection first

**Run seed**:
```bash
npx tsx scripts/seed.ts
```

---

## Step 8: Add Scripts to `package.json`

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "test:db": "tsx scripts/test-connection.ts",
    "seed": "tsx scripts/seed.ts"
  }
}
```

**Usage**:
```bash
npm run test:db  # Test connection
npm run seed     # Seed database
```

---

## Step 9: Use in API Routes

Create `app/api/posts/route.ts`:

```typescript
import { NextResponse } from 'next/server';
import { getAllPosts, createPost } from '@/lib/db';

// GET all posts
export async function GET() {
  try {
    const posts = await getAllPosts();
    return NextResponse.json({ success: true, data: posts });
  } catch (error) {
    return NextResponse.json(
      { success: false, error: 'Failed to fetch posts' },
      { status: 500 }
    );
  }
}

// POST create post
export async function POST(request: Request) {
  try {
    const body = await request.json();
    const post = await createPost(body);
    return NextResponse.json({ success: true, data: post }, { status: 201 });
  } catch (error) {
    return NextResponse.json(
      { success: false, error: 'Failed to create post' },
      { status: 500 }
    );
  }
}
```

---

## Complete Flow Summary

```
1. Request comes in
   ↓
2. API route calls helper function (e.g., getAllPosts)
   ↓
3. Helper calls connectDB()
   ↓
4. connectDB checks cached connection
   ↓
   ├─ If exists: Return cached (fast!)
   └─ If not: Create new connection & cache it
   ↓
5. Use Mongoose model (e.g., Post.find())
   ↓
6. Mongoose validates & queries MongoDB
   ↓
7. Return typed data to API route
   ↓
8. Send JSON response
```

This setup gives you a production-ready, type-safe, performant MongoDB connection with Mongoose! 🚀
