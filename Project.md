# Incremental Static Regeneration (ISR) in Next.js - Complete Guide

Let me guide you through ISR step-by-step with a practical project.

## Overview: What is ISR?

**Incremental Static Regeneration** allows you to:
- Generate static pages at build time (like SSG)
- **Regenerate** those pages in the background after deployment
- Serve stale content instantly while regenerating
- Update pages without rebuilding the entire site

**Key concept**: You set a `revalidate` interval (e.g., 60 seconds). When a request comes after this period, Next.js serves the cached page but triggers a regeneration in the background. The next visitor gets the fresh page.

**When to use ISR:**
- Blog posts that update occasionally
- Product pages with changing inventory
- Content that needs to be fresh but not real-time
- Thousands of pages that would be slow to build all at once

**Our project**: A simple blog that fetches posts from MongoDB. Posts can be added/updated, and ISR will regenerate pages automatically.

---

## Step 1: Project Setup & Dependencies

### What we're doing
Setting up a Next.js project with TypeScript support and installing dependencies for MongoDB and our Express backend.

### Commands

```bash
# Create a new Next.js project
npx create-next-app@latest next-isr-blog
# When prompted, choose:
# - TypeScript: Yes
# - ESLint: Yes
# - Tailwind CSS: Yes
# - src/ directory: No
# - App Router: No (we'll use Pages Router for clearer ISR examples)
# - Import alias: Yes (default @/*)

cd next-isr-blog

# Install dependencies
npm install mongodb express cors
npm install --save-dev @types/express @types/cors
```

### Why these choices?

- **Pages Router**: ISR is more straightforward to understand with `getStaticProps`
- **MongoDB**: Simple document storage for our blog posts
- **Express**: Separate API for content management (simulates a real CMS)
- **TypeScript**: Better developer experience with type safety

### Troubleshooting

- If `create-next-app` fails, ensure Node.js version is 18.17 or higher: `node --version`
- On Windows, use PowerShell or Command Prompt, not Git Bash for `create-next-app`

### Testing this step

```bash
npm run dev
# Visit http://localhost:3000 - you should see the Next.js welcome page
```

**Summary**: ✅ Project created with Pages Router and dependencies installed.

---

## Step 2: Set Up MongoDB Atlas

### What we're doing
Creating a free MongoDB Atlas cluster and getting a connection string. This will store our blog posts.

### Steps

1. **Go to** [MongoDB Atlas](https://www.mongodb.com/cloud/atlas/register)
2. **Sign up** for a free account
3. **Create a cluster**:
   - Choose M0 (Free tier)
   - Select a region close to you
   - Name it `next-isr-cluster`
4. **Set up database access**:
   - Go to "Database Access" → Add New User
   - Username: `next_user`
   - Password: Generate a secure password (save it!)
   - Database User Privileges: "Read and write to any database"
5. **Network Access**:
   - Go to "Network Access" → Add IP Address
   - Click "Allow Access from Anywhere" (0.0.0.0/0)
   - Confirm (only for development!)
6. **Get connection string**:
   - Go to "Database" → Click "Connect"
   - Choose "Connect your application"
   - Copy the connection string (looks like: `mongodb+srv://next_user:<password>@...`)

### Create environment file

In your project root, create `.env.local`:

```bash
MONGODB_URI=mongodb+srv://next_user:YOUR_PASSWORD@next-isr-cluster.xxxxx.mongodb.net/?retryWrites=true&w=majority
MONGODB_DB=next_isr_blog
```

Replace:
- `YOUR_PASSWORD` with your actual password
- The cluster URL with your actual URL from Atlas

### Troubleshooting

- **"IP not whitelisted"**: Double-check Network Access settings
- **"Authentication failed"**: Verify username/password in connection string
- **Connection timeout**: Check your firewall/VPN isn't blocking MongoDB ports

### Testing this step

We'll test the connection in the next step when we create the database utility.

**Summary**: ✅ MongoDB Atlas cluster created and connection string saved to `.env.local`.

---

## Step 3: Create MongoDB Connection Utility

### What we're doing
Creating a reusable MongoDB connection that handles connection pooling efficiently.

### Create the file

Create `lib/mongodb.ts`:

```typescript
import { MongoClient } from 'mongodb';

// Ensure the URI exists
if (!process.env.MONGODB_URI) {
  throw new Error('Please add your Mongo URI to .env.local');
}

const uri = process.env.MONGODB_URI;
const options = {};

let client: MongoClient;
let clientPromise: Promise<MongoClient>;

// In development, use a global variable to preserve the connection across hot reloads
// In production, create a new connection
if (process.env.NODE_ENV === 'development') {
  // Use global variable to avoid multiple connections during hot reload
  let globalWithMongo = global as typeof globalThis & {
    _mongoClientPromise?: Promise<MongoClient>;
  };

  if (!globalWithMongo._mongoClientPromise) {
    client = new MongoClient(uri, options);
    globalWithMongo._mongoClientPromise = client.connect();
  }
  clientPromise = globalWithMongo._mongoClientPromise;
} else {
  // In production, create a new client
  client = new MongoClient(uri, options);
  clientPromise = client.connect();
}

export default clientPromise;
```

### Why this pattern?

- **Connection pooling**: Reuses connections instead of creating new ones
- **Development optimization**: Prevents connection leaks during hot reload
- **Type safety**: TypeScript ensures we're using the client correctly

### Create a helper to get the database

Create `lib/db.ts`:

```typescript
import clientPromise from './mongodb';

export async function getDatabase() {
  const client = await clientPromise;
  const db = client.db(process.env.MONGODB_DB || 'next_isr_blog');
  return db;
}

// Helper to get the posts collection with proper typing
export interface Post {
  _id?: string;
  slug: string;
  title: string;
  content: string;
  author: string;
  createdAt: Date;
  updatedAt: Date;
}

export async function getPostsCollection() {
  const db = await getDatabase();
  return db.collection<Post>('posts');
}
```

### Testing this step

Create a test file `test-db.js` in your project root:

```javascript
require('dotenv').config({ path: '.env.local' });
const { MongoClient } = require('mongodb');

async function testConnection() {
  const client = new MongoClient(process.env.MONGODB_URI);
  
  try {
    await client.connect();
    console.log('✅ Connected to MongoDB!');
    
    const db = client.db(process.env.MONGODB_DB);
    const collections = await db.listCollections().toArray();
    console.log('📦 Collections:', collections.map(c => c.name));
  } catch (error) {
    console.error('❌ Connection failed:', error.message);
  } finally {
    await client.close();
  }
}

testConnection();
```

Run it:

```bash
node test-db.js
```

You should see: "✅ Connected to MongoDB!"

### Troubleshooting

- **"Cannot find module 'dotenv'"**: Run `npm install dotenv --save-dev`
- **Connection errors**: Revisit Step 2, verify `.env.local` format
- **TypeScript errors**: Ensure `tsconfig.json` has `"lib": ["dom", "es2015"]`

**Summary**: ✅ MongoDB connection utility created and tested successfully.

---

## Step 4: Seed Initial Data

### What we're doing
Adding some sample blog posts to MongoDB so we have data to work with.

### Create the seed script

Create `scripts/seed.ts`:

```typescript
import { getPostsCollection } from '../lib/db';

const samplePosts = [
  {
    slug: 'hello-isr',
    title: 'Hello ISR!',
    content: 'This is our first post demonstrating Incremental Static Regeneration. ISR allows us to update this content without rebuilding the entire site.',
    author: 'John Doe',
    createdAt: new Date(),
    updatedAt: new Date(),
  },
  {
    slug: 'understanding-isr',
    title: 'Understanding ISR',
    content: 'ISR combines the best of static and dynamic. Pages are pre-rendered but can be updated in the background based on a revalidation period.',
    author: 'Jane Smith',
    createdAt: new Date(),
    updatedAt: new Date(),
  },
  {
    slug: 'isr-benefits',
    title: 'Benefits of ISR',
    content: 'ISR provides fast page loads, SEO benefits, and the ability to update content without redeployment. Perfect for blogs, e-commerce, and content sites.',
    author: 'Bob Johnson',
    createdAt: new Date(),
    updatedAt: new Date(),
  },
];

async function seed() {
  try {
    const posts = await getPostsCollection();
    
    // Clear existing posts
    await posts.deleteMany({});
    console.log('🗑️  Cleared existing posts');
    
    // Insert sample posts
    const result = await posts.insertMany(samplePosts);
    console.log(`✅ Inserted ${result.insertedCount} posts`);
    
    // Display inserted posts
    const allPosts = await posts.find({}).toArray();
    console.log('\n📝 Current posts:');
    allPosts.forEach(post => {
      console.log(`  - ${post.title} (/${post.slug})`);
    });
    
    process.exit(0);
  } catch (error) {
    console.error('❌ Seed failed:', error);
    process.exit(1);
  }
}

seed();
```

### Add a script command

Update `package.json`, add to `"scripts"`:

```json
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "start": "next start",
  "lint": "next lint",
  "seed": "ts-node --compiler-options {\\\"module\\\":\\\"commonjs\\\"} scripts/seed.ts"
},
```

### Install ts-node

```bash
npm install --save-dev ts-node
```

### Run the seed

```bash
npm run seed
```

You should see:
```
🗑️  Cleared existing posts
✅ Inserted 3 posts

📝 Current posts:
  - Hello ISR! (/hello-isr)
  - Understanding ISR (/understanding-isr)
  - Benefits of ISR (/isr-benefits)
```

### Troubleshooting

- **"Cannot use import statement"**: The `ts-node` config in package.json handles this
- **MongoDB connection timeout**: Ensure `npm run dev` isn't already using the connection
- **Seed runs but no output**: Check `.env.local` is in the project root

### Verify in MongoDB Atlas

1. Go to your cluster in MongoDB Atlas
2. Click "Browse Collections"
3. You should see `next_isr_blog` database with `posts` collection

**Summary**: ✅ Sample blog posts added to MongoDB. Ready to fetch them with ISR!

---

## Step 5: Create the Home Page with ISR

### What we're doing
Building the home page that lists all blog posts using `getStaticProps` with `revalidate` (this is ISR!).

### Create the home page

Replace `pages/index.tsx` with:

```typescript
import { GetStaticProps } from 'next';
import Link from 'next/link';
import { getPostsCollection, Post } from '../lib/db';

interface HomeProps {
  posts: Post[];
  lastBuilt: string;
}

export default function Home({ posts, lastBuilt }: HomeProps) {
  return (
    <div className="min-h-screen bg-gray-50 py-12 px-4">
      <div className="max-w-4xl mx-auto">
        <header className="mb-12">
          <h1 className="text-4xl font-bold text-gray-900 mb-2">
            ISR Blog Demo
          </h1>
          <p className="text-gray-600">
            Demonstrating Incremental Static Regeneration
          </p>
          <p className="text-sm text-gray-500 mt-2">
            Last built: {lastBuilt}
          </p>
        </header>

        <div className="space-y-6">
          {posts.map((post) => (
            <article
              key={post.slug}
              className="bg-white rounded-lg shadow-md p-6 hover:shadow-lg transition-shadow"
            >
              <Link href={`/posts/${post.slug}`}>
                <h2 className="text-2xl font-semibold text-blue-600 hover:text-blue-800 mb-2">
                  {post.title}
                </h2>
              </Link>
              <p className="text-gray-700 mb-4 line-clamp-3">
                {post.content}
              </p>
              <div className="flex justify-between items-center text-sm text-gray-500">
                <span>By {post.author}</span>
                <span>{new Date(post.updatedAt).toLocaleDateString()}</span>
              </div>
            </article>
          ))}
        </div>
      </div>
    </div>
  );
}

// This is ISR in action!
export const getStaticProps: GetStaticProps<HomeProps> = async () => {
  const posts = await getPostsCollection();
  const allPosts = await posts.find({}).sort({ createdAt: -1 }).toArray();

  return {
    props: {
      posts: JSON.parse(JSON.stringify(allPosts)), // Serialize dates
      lastBuilt: new Date().toISOString(),
    },
    revalidate: 10, // Regenerate page every 10 seconds
  };
};
```

### Key concepts explained

**`getStaticProps`**: Runs at build time (and during revalidation) on the server. Never runs in the browser.

**`revalidate: 10`**: The ISR magic! After 10 seconds:
- A new request comes in
- Next.js serves the stale (cached) page instantly
- Triggers regeneration in the background
- Next request gets the fresh page

**`JSON.parse(JSON.stringify())`**: MongoDB returns dates as objects, but props must be JSON-serializable. This converts them.

### Testing locally

```bash
npm run dev
```

Visit `http://localhost:3000`. You should see your three blog posts.

**Note**: In development mode (`npm run dev`), ISR behaves differently:
- Pages regenerate on every request
- `revalidate` is ignored
- This is intentional for better DX

To test real ISR behavior locally:

```bash
# Build for production
npm run build

# Start production server
npm run start
```

Visit `http://localhost:3000`:
1. Note the "Last built" timestamp
2. Wait 10 seconds
3. Refresh the page
4. The timestamp should update (background regeneration happened)

### Troubleshooting

- **Build errors about MongoDB**: Ensure `.env.local` exists and is correct
- **"Cannot read properties of undefined"**: Check MongoDB connection
- **Blank page**: Check browser console for errors
- **TypeScript errors on `line-clamp-3`**: Update `tailwind.config.ts` (we'll do this next)

### Update Tailwind config

Update `tailwind.config.ts` to support line-clamp:

```typescript
import type { Config } from "tailwindcss";

const config: Config = {
  content: [
    "./pages/**/*.{js,ts,jsx,tsx,mdx}",
    "./components/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [
    require('@tailwindcss/line-clamp'),
  ],
};
export default config;
```

Install the plugin:

```bash
npm install -D @tailwindcss/line-clamp
```

**Summary**: ✅ Home page created with ISR! Posts list regenerates every 10 seconds.

---

## Step 6: Create Individual Post Pages with ISR

### What we're doing
Creating dynamic routes for individual posts using `getStaticPaths` and `getStaticProps` with ISR.

### Create the dynamic route

Create `pages/posts/[slug].tsx`:

```typescript
import { GetStaticPaths, GetStaticProps } from 'next';
import Link from 'next/link';
import { useRouter } from 'next/router';
import { getPostsCollection, Post } from '../../lib/db';

interface PostPageProps {
  post: Post;
  lastBuilt: string;
}

export default function PostPage({ post, lastBuilt }: PostPageProps) {
  const router = useRouter();

  // Show a loading state while the page is being generated (fallback: 'blocking')
  if (router.isFallback) {
    return (
      <div className="min-h-screen bg-gray-50 flex items-center justify-center">
        <div className="text-xl text-gray-600">Loading...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-50 py-12 px-4">
      <article className="max-w-3xl mx-auto bg-white rounded-lg shadow-md p-8">
        <Link href="/" className="text-blue-600 hover:text-blue-800 mb-6 inline-block">
          ← Back to home
        </Link>
        
        <header className="mb-8">
          <h1 className="text-4xl font-bold text-gray-900 mb-4">
            {post.title}
          </h1>
          <div className="flex items-center text-gray-600 text-sm space-x-4">
            <span>By {post.author}</span>
            <span>•</span>
            <span>{new Date(post.updatedAt).toLocaleDateString()}</span>
          </div>
        </header>

        <div className="prose prose-lg max-w-none">
          <p className="text-gray-700 leading-relaxed whitespace-pre-wrap">
            {post.content}
          </p>
        </div>

        <footer className="mt-12 pt-6 border-t border-gray-200">
          <p className="text-xs text-gray-500">
            Page last built: {new Date(lastBuilt).toLocaleString()}
          </p>
        </footer>
      </article>
    </div>
  );
}

// Tell Next.js which paths to pre-render at build time
export const getStaticPaths: GetStaticPaths = async () => {
  const posts = await getPostsCollection();
  const allPosts = await posts.find({}).toArray();

  const paths = allPosts.map((post) => ({
    params: { slug: post.slug },
  }));

  return {
    paths,
    fallback: 'blocking', // Generate new pages on-demand if not pre-rendered
  };
};

// Fetch the post data
export const getStaticProps: GetStaticProps<PostPageProps> = async ({ params }) => {
  const posts = await getPostsCollection();
  const post = await posts.findOne({ slug: params?.slug as string });

  if (!post) {
    return {
      notFound: true, // Show 404 page
    };
  }

  return {
    props: {
      post: JSON.parse(JSON.stringify(post)),
      lastBuilt: new Date().toISOString(),
    },
    revalidate: 10, // ISR: Regenerate every 10 seconds
  };
};
```

### Key concepts explained

**`getStaticPaths`**: Tells Next.js which dynamic routes exist at build time.
- Returns an array of `params` (e.g., `{ slug: 'hello-isr' }`)
- These paths are pre-rendered during `npm run build`

**`fallback: 'blocking'`**: 
- If someone visits a slug not pre-rendered (e.g., a new post added after build)
- Next.js generates the page on-demand (server-side)
- The visitor waits (blocked) until generation completes
- The page is then cached for future requests
- This enables true on-demand ISR!

**Alternative fallback options**:
- `false`: Only pre-rendered paths are valid (404 for others)
- `true`: Shows a loading state, then client-side fetches the page
- `'blocking'`: Server-side renders on-demand (better for SEO)

**`revalidate: 10`**: Same as home page - regenerates every 10 seconds.

### Testing

Start the dev server:

```bash
npm run dev
```

1. Visit `http://localhost:3000`
2. Click on a post title
3. You should see the full post page at `/posts/hello-isr` (or another slug)
4. Click "Back to home"

### Test ISR behavior in production mode

```bash
npm run build
npm run start
```

1. Visit a post page: `http://localhost:3000/posts/hello-isr`
2. Note the "Page last built" timestamp
3. Wait 10 seconds and refresh
4. The timestamp should update

### Troubleshooting

- **404 errors**: Ensure MongoDB has posts with the correct slugs
- **"Cannot read property 'slug'"**: Check `params?.slug` is being used
- **Build fails**: Ensure database is accessible during build (check firewall)
- **Fallback page shows forever**: Check MongoDB connection and that the slug exists

**Summary**: ✅ Individual post pages created with ISR and on-demand generation!

---

## Step 7: Create an API to Update Posts

### What we're doing
Building an Express server that provides API endpoints to create/update posts. This simulates a CMS that triggers ISR regeneration.

### Create the Express server

Create `server/index.ts`:

```typescript
import express from 'express';
import cors from 'cors';
import { getPostsCollection } from '../lib/db';

const app = express();
const PORT = process.env.PORT || 4000;

app.use(cors());
app.use(express.json());

// Get all posts
app.get('/api/posts', async (req, res) => {
  try {
    const posts = await getPostsCollection();
    const allPosts = await posts.find({}).sort({ createdAt: -1 }).toArray();
    res.json(allPosts);
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch posts' });
  }
});

// Get a single post
app.get('/api/posts/:slug', async (req, res) => {
  try {
    const posts = await getPostsCollection();
    const post = await posts.findOne({ slug: req.params.slug });
    
    if (!post) {
      return res.status(404).json({ error: 'Post not found' });
    }
    
    res.json(post);
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch post' });
  }
});

// Create a new post
app.post('/api/posts', async (req, res) => {
  try {
    const { slug, title, content, author } = req.body;
    
    if (!slug || !title || !content || !author) {
      return res.status(400).json({ error: 'Missing required fields' });
    }
    
    const posts = await getPostsCollection();
    
    // Check if slug already exists
    const existing = await posts.findOne({ slug });
    if (existing) {
      return res.status(409).json({ error: 'Slug already exists' });
    }
    
    const newPost = {
      slug,
      title,
      content,
      author,
      createdAt: new Date(),
      updatedAt: new Date(),
    };
    
    await posts.insertOne(newPost);
    
    // Trigger ISR revalidation
    await fetch(`http://localhost:3000/api/revalidate?path=/`);
    await fetch(`http://localhost:3000/api/revalidate?path=/posts/${slug}`);
    
    res.status(201).json(newPost);
  } catch (error) {
    res.status(500).json({ error: 'Failed to create post' });
  }
});

// Update a post
app.put('/api/posts/:slug', async (req, res) => {
  try {
    const { title, content, author } = req.body;
    const posts = await getPostsCollection();
    
    const result = await posts.findOneAndUpdate(
      { slug: req.params.slug },
      { 
        $set: { 
          title, 
          content, 
          author, 
          updatedAt: new Date() 
        } 
      },
      { returnDocument: 'after' }
    );
    
    if (!result) {
      return res.status(404).json({ error: 'Post not found' });
    }
    
    // Trigger ISR revalidation
    await fetch(`http://localhost:3000/api/revalidate?path=/`);
    await fetch(`http://localhost:3000/api/revalidate?path=/posts/${req.params.slug}`);
    
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: 'Failed to update post' });
  }
});

// Delete a post
app.delete('/api/posts/:slug', async (req, res) => {
  try {
    const posts = await getPostsCollection();
    const result = await posts.deleteOne({ slug: req.params.slug });
    
    if (result.deletedCount === 0) {
      return res.status(404).json({ error: 'Post not found' });
    }
    
    // Trigger ISR revalidation
    await fetch(`http://localhost:3000/api/revalidate?path=/`);
    
    res.json({ message: 'Post deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: 'Failed to delete post' });
  }
});

app.listen(PORT, () => {
  console.log(`✅ API server running on http://localhost:${PORT}`);
});
```

### Create Next.js revalidation API route

Create `pages/api/revalidate.ts`:

```typescript
import type { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  // Check for secret to confirm this is a valid request
  if (req.query.secret !== process.env.REVALIDATION_SECRET) {
    return res.status(401).json({ message: 'Invalid token' });
  }

  try {
    const path = req.query.path as string;
    
    if (!path) {
      return res.status(400).json({ message: 'Missing path' });
    }

    // Revalidate the specified path
    await res.revalidate(path);
    
    return res.json({ revalidated: true, path });
  } catch (err) {
    return res.status(500).json({ message: 'Error revalidating' });
  }
}
```

### Add environment variable

Add to `.env.local`:

```bash
REVALIDATION_SECRET=my-secret-token-12345
```

### Update server to use the secret

Update the fetch calls in `server/index.ts`:

```typescript
// Replace the revalidation fetch calls with:
const secret = process.env.REVALIDATION_SECRET || 'my-secret-token-12345';
await fetch(`http://localhost:3000/api/revalidate?secret=${secret}&path=/`);
await fetch(`http://localhost:3000/api/revalidate?secret=${secret}&path=/posts/${slug}`);
```

### Add run script

Add to `package.json` scripts:

```json
"api": "ts-node --compiler-options {\\\"module\\\":\\\"commonjs\\\"} server/index.ts"
```

### Testing

Open two terminals:

**Terminal 1** (Next.js):
```bash
npm run build
npm run start
```

**Terminal 2** (API server):
```bash
npm run api
```

### Test the API

Use curl or a tool like Postman:

```bash
# Create a new post
curl -X POST http://localhost:4000/api/posts \
  -H "Content-Type: application/json" \
  -d '{
    "slug": "new-post",
    "title": "A New Post",
    "content": "This post was created via the API!",
    "author": "API User"
  }'

# Update a post
curl -X PUT http://localhost:4000/api/posts/hello-isr \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Hello ISR (Updated!)",
    "content": "This content was updated via the API.",
    "author": "John Doe"
  }'
```

After running these commands:
1. Visit `http://localhost:3000`
2. You should see the new/updated posts **immediately** (ISR was triggered!)
3. Check the "Last built" timestamp - it should be recent

### Troubleshooting

- **"ECONNREFUSED"**: Ensure Next.js is running on port 3000
- **"Invalid token"**: Check `REVALIDATION_SECRET` matches in both places
- **Revalidation doesn't work**: Ensure you're using `npm run start` (production), not `npm run dev`
- **TypeScript errors**: Run `npm install --save-dev @types/node`

**Summary**: ✅ API server created! Can create/update posts and trigger ISR revalidation on-demand.

---

## Step 8: Understanding On-Demand Revalidation

### What we're doing
Understanding how on-demand ISR works vs time-based ISR, and when to use each.

### Two types of ISR

**1. Time-based ISR** (what we've been using):
```typescript
export const getStaticProps: GetStaticProps = async () => {
  return {
    props: { /* ... */ },
    revalidate: 10, // Regenerate after 10 seconds
  };
};
```

**How it works**:
- Page is cached for 10 seconds
- After 10 seconds, next request serves stale page and triggers background regeneration
- Subsequent requests get fresh page
- **Downside**: Can't immediately update when content changes

**2. On-demand ISR** (res.revalidate()):
```typescript
// In an API route
await res.revalidate('/posts/my-post');
```

**How it works**:
- You manually trigger regeneration via API call
- Page regenerates immediately in the background
- Next request gets the fresh page
- **Benefit**: Content updates immediately when you want it to

### Combining both approaches

You can (and should!) use both:

```typescript
export const getStaticProps: GetStaticProps = async () => {
  return {
    props: { /* ... */ },
    revalidate: 60, // Fallback: regenerate every 60 seconds
  };
};
```

Then trigger on-demand when content changes:
```typescript
await res.revalidate('/posts/my-post');
```

This provides:
- **Immediate updates** when content is edited (on-demand)
- **Automatic freshness** even if you forget to revalidate (time-based)

### Best practices

**Use time-based ISR for**:
- Content that changes predictably (e.g., daily news)
- When you want "good enough" freshness without manual triggers
- Fallback for when on-demand revalidation might fail

**Use on-demand ISR for**:
- Content management systems
- E-commerce (product updates, inventory)
- User-generated content
- When you need instant updates




### Revalidate intervals - choosing the right value

```typescript
revalidate: 1     // Every second - very aggressive, use sparingly
revalidate: 10    // Every 10 seconds - good for demos/high-frequency updates
revalidate: 60    // Every minute - frequently changing content
revalidate: 300   // Every 5 minutes - moderate freshness
revalidate: 3600  // Every hour - slow-changing content
revalidate: 86400 // Every day - rare updates
```

**Consider**:
- **Traffic**: High traffic + low revalidate = many regenerations
- **Build time**: Complex pages take longer to regenerate
- **Database load**: Each regeneration queries your database
- **Content change frequency**: Match revalidate to actual update rate

### Testing on-demand revalidation

Let's verify it works:

1. **Ensure both servers are running** (Terminal 1: `npm run start`, Terminal 2: `npm run api`)

2. **Visit a post**: `http://localhost:3000/posts/hello-isr`
   - Note the "Page last built" timestamp

3. **Update the post via API**:
```bash
curl -X PUT http://localhost:4000/api/posts/hello-isr \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Hello ISR - Updated at '$(date +%H:%M:%S)'",
    "content": "This was just updated! The timestamp in the title shows when.",
    "author": "John Doe"
  }'
```

4. **Refresh the browser immediately**
   - The title should show the update!
   - "Page last built" timestamp should be very recent
   - This happened **without waiting** for the time-based revalidate period

### Troubleshooting

- **Page doesn't update**: Check the API response - did revalidation succeed?
- **"Revalidated: true" but page is stale**: Wait a few seconds, then refresh
- **Still seeing old content**: Clear browser cache (hard refresh: Ctrl+Shift+R)

**Summary**: ✅ On-demand ISR provides instant updates! Time-based ISR provides automatic freshness. Use both together.

---

## Step 9: Implement Preview Mode

### What we're doing
Adding Preview Mode - a way to bypass ISR caching and see draft/unpublished content before it goes live.

### Why Preview Mode?

Imagine you're editing a blog post:
- ISR serves cached pages to visitors
- You want to preview changes **before** publishing
- Preview Mode bypasses the cache and shows the latest data

### How it works

1. Enter preview mode via a special API route
2. Next.js sets preview cookies
3. `getStaticProps` receives a `preview` flag
4. You fetch draft data instead of published data
5. Exit preview mode to return to cached pages

### Add draft status to posts

First, update the Post interface in `lib/db.ts`:

```typescript
export interface Post {
  _id?: string;
  slug: string;
  title: string;
  content: string;
  author: string;
  createdAt: Date;
  updatedAt: Date;
  status: 'draft' | 'published'; // Add this
}
```

### Update seed script

Update `scripts/seed.ts` to include status:

```typescript
const samplePosts = [
  {
    slug: 'hello-isr',
    title: 'Hello ISR!',
    content: 'This is our first post demonstrating Incremental Static Regeneration. ISR allows us to update this content without rebuilding the entire site.',
    author: 'John Doe',
    status: 'published' as const,
    createdAt: new Date(),
    updatedAt: new Date(),
  },
  {
    slug: 'understanding-isr',
    title: 'Understanding ISR',
    content: 'ISR combines the best of static and dynamic. Pages are pre-rendered but can be updated in the background based on a revalidation period.',
    author: 'Jane Smith',
    status: 'published' as const,
    createdAt: new Date(),
    updatedAt: new Date(),
  },
  {
    slug: 'isr-benefits',
    title: 'Benefits of ISR',
    content: 'ISR provides fast page loads, SEO benefits, and the ability to update content without redeployment. Perfect for blogs, e-commerce, and content sites.',
    author: 'Bob Johnson',
    status: 'published' as const,
    createdAt: new Date(),
    updatedAt: new Date(),
  },
  {
    slug: 'draft-post',
    title: 'This is a Draft Post',
    content: 'This post is still being written. It should only be visible in preview mode.',
    author: 'Editor',
    status: 'draft' as const,
    createdAt: new Date(),
    updatedAt: new Date(),
  },
];
```

Run the seed again:
```bash
npm run seed
```

### Create preview API routes

Create `pages/api/preview.ts`:

```typescript
import type { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  // Check the secret to confirm this is a valid request
  if (req.query.secret !== process.env.PREVIEW_SECRET) {
    return res.status(401).json({ message: 'Invalid token' });
  }

  // Enable Preview Mode by setting the cookies
  res.setPreviewData({});

  // Redirect to the path from the query parameter
  const redirectPath = req.query.redirect || '/';
  res.redirect(redirectPath as string);
}
```

Create `pages/api/exit-preview.ts`:

```typescript
import type { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  // Clear the preview mode cookies
  res.clearPreviewData();

  // Redirect to the path from the query parameter
  const redirectPath = req.query.redirect || '/';
  res.redirect(redirectPath as string);
}
```

### Add preview secret to environment

Add to `.env.local`:

```bash
PREVIEW_SECRET=preview-secret-12345
```

### Update home page to support preview

Update `pages/index.tsx`:

```typescript
import { GetStaticProps } from 'next';
import Link from 'next/link';
import { useRouter } from 'next/router';
import { getPostsCollection, Post } from '../lib/db';

interface HomeProps {
  posts: Post[];
  lastBuilt: string;
  preview: boolean;
}

export default function Home({ posts, lastBuilt, preview }: HomeProps) {
  const router = useRouter();

  return (
    <div className="min-h-screen bg-gray-50 py-12 px-4">
      <div className="max-w-4xl mx-auto">
        {preview && (
          <div className="bg-yellow-100 border-l-4 border-yellow-500 text-yellow-700 p-4 mb-6">
            <div className="flex justify-between items-center">
              <div>
                <p className="font-bold">Preview Mode Active</p>
                <p className="text-sm">You're seeing draft content</p>
              </div>
              
                href="/api/exit-preview"
                className="bg-yellow-600 text-white px-4 py-2 rounded hover:bg-yellow-700"
              >
                Exit Preview
              </a>
            </div>
          </div>
        )}

        <header className="mb-12">
          <h1 className="text-4xl font-bold text-gray-900 mb-2">
            ISR Blog Demo
          </h1>
          <p className="text-gray-600">
            Demonstrating Incremental Static Regeneration
          </p>
          <p className="text-sm text-gray-500 mt-2">
            Last built: {lastBuilt}
          </p>
        </header>

        <div className="space-y-6">
          {posts.map((post) => (
            <article
              key={post.slug}
              className="bg-white rounded-lg shadow-md p-6 hover:shadow-lg transition-shadow"
            >
              <Link href={`/posts/${post.slug}`}>
                <h2 className="text-2xl font-semibold text-blue-600 hover:text-blue-800 mb-2">
                  {post.title}
                  {post.status === 'draft' && (
                    <span className="ml-2 text-sm bg-yellow-200 text-yellow-800 px-2 py-1 rounded">
                      DRAFT
                    </span>
                  )}
                </h2>
              </Link>
              <p className="text-gray-700 mb-4 line-clamp-3">
                {post.content}
              </p>
              <div className="flex justify-between items-center text-sm text-gray-500">
                <span>By {post.author}</span>
                <span>{new Date(post.updatedAt).toLocaleDateString()}</span>
              </div>
            </article>
          ))}
        </div>
      </div>
    </div>
  );
}

export const getStaticProps: GetStaticProps<HomeProps> = async ({ preview = false }) => {
  const posts = await getPostsCollection();
  
  // In preview mode, show all posts (including drafts)
  // In normal mode, only show published posts
  const query = preview ? {} : { status: 'published' };
  const allPosts = await posts.find(query).sort({ createdAt: -1 }).toArray();

  return {
    props: {
      posts: JSON.parse(JSON.stringify(allPosts)),
      lastBuilt: new Date().toISOString(),
      preview,
    },
    revalidate: 10,
  };
};
```

### Update post page to support preview

Update `pages/posts/[slug].tsx`:

```typescript
import { GetStaticPaths, GetStaticProps } from 'next';
import Link from 'next/link';
import { useRouter } from 'next/router';
import { getPostsCollection, Post } from '../../lib/db';

interface PostPageProps {
  post: Post;
  lastBuilt: string;
  preview: boolean;
}

export default function PostPage({ post, lastBuilt, preview }: PostPageProps) {
  const router = useRouter();

  if (router.isFallback) {
    return (
      <div className="min-h-screen bg-gray-50 flex items-center justify-center">
        <div className="text-xl text-gray-600">Loading...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-50 py-12 px-4">
      {preview && (
        <div className="max-w-3xl mx-auto mb-6">
          <div className="bg-yellow-100 border-l-4 border-yellow-500 text-yellow-700 p-4">
            <div className="flex justify-between items-center">
              <div>
                <p className="font-bold">Preview Mode Active</p>
                <p className="text-sm">You're viewing draft content</p>
              </div>
              
                href="/api/exit-preview"
                className="bg-yellow-600 text-white px-4 py-2 rounded hover:bg-yellow-700"
              >
                Exit Preview
              </a>
            </div>
          </div>
        </div>
      )}

      <article className="max-w-3xl mx-auto bg-white rounded-lg shadow-md p-8">
        <Link href="/" className="text-blue-600 hover:text-blue-800 mb-6 inline-block">
          ← Back to home
        </Link>
        
        <header className="mb-8">
          <h1 className="text-4xl font-bold text-gray-900 mb-4">
            {post.title}
            {post.status === 'draft' && (
              <span className="ml-3 text-lg bg-yellow-200 text-yellow-800 px-3 py-1 rounded">
                DRAFT
              </span>
            )}
          </h1>
          <div className="flex items-center text-gray-600 text-sm space-x-4">
            <span>By {post.author}</span>
            <span>•</span>
            <span>{new Date(post.updatedAt).toLocaleDateString()}</span>
            <span>•</span>
            <span className="capitalize">{post.status}</span>
          </div>
        </header>

        <div className="prose prose-lg max-w-none">
          <p className="text-gray-700 leading-relaxed whitespace-pre-wrap">
            {post.content}
          </p>
        </div>

        <footer className="mt-12 pt-6 border-t border-gray-200">
          <p className="text-xs text-gray-500">
            Page last built: {new Date(lastBuilt).toLocaleString()}
          </p>
        </footer>
      </article>
    </div>
  );
}

export const getStaticPaths: GetStaticPaths = async () => {
  const posts = await getPostsCollection();
  // Only pre-render published posts
  const publishedPosts = await posts.find({ status: 'published' }).toArray();

  const paths = publishedPosts.map((post) => ({
    params: { slug: post.slug },
  }));

  return {
    paths,
    fallback: 'blocking',
  };
};

export const getStaticProps: GetStaticProps<PostPageProps> = async ({ 
  params, 
  preview = false 
}) => {
  const posts = await getPostsCollection();
  
  // In preview mode, fetch any post (including drafts)
  // In normal mode, only fetch published posts
  const query = preview 
    ? { slug: params?.slug as string }
    : { slug: params?.slug as string, status: 'published' };
  
  const post = await posts.findOne(query);

  if (!post) {
    return {
      notFound: true,
    };
  }

  return {
    props: {
      post: JSON.parse(JSON.stringify(post)),
      lastBuilt: new Date().toISOString(),
      preview,
    },
    revalidate: 10,
  };
};
```

### Testing Preview Mode

1. **Rebuild and start** (to see published posts only):
```bash
npm run build
npm run start
```

2. **Visit home page**: `http://localhost:3000`
   - You should see 3 published posts (no draft)

3. **Try to visit draft post**: `http://localhost:3000/posts/draft-post`
   - Should show 404 (draft posts aren't accessible normally)

4. **Enter preview mode**: `http://localhost:3000/api/preview?secret=preview-secret-12345&redirect=/`
   - You're redirected to home page
   - Yellow banner appears: "Preview Mode Active"
   - You now see **4 posts** including the draft!

5. **Visit draft post**: `http://localhost:3000/posts/draft-post`
   - Now it's accessible!
   - Shows "DRAFT" badge
   - Yellow preview banner visible

6. **Exit preview mode**: Click "Exit Preview" button
   - Draft post disappears from listing
   - Visiting `/posts/draft-post` shows 404 again

### Common use cases

**CMS Preview Button**:
```html
<a href="/api/preview?secret=YOUR_SECRET&redirect=/posts/my-draft">
  Preview Draft
</a>
```

**Preview specific content**:
```
/api/preview?secret=YOUR_SECRET&redirect=/posts/draft-post
```

**Preview entire site**:
```
/api/preview?secret=YOUR_SECRET&redirect=/
```

### Troubleshooting

- **"Invalid token"**: Check `PREVIEW_SECRET` in `.env.local`
- **Draft posts still show after exit**: Clear browser cookies or use incognito
- **Preview mode doesn't activate**: Ensure you're using `npm run start` (not dev mode)
- **404 for draft in preview mode**: Check the status field in MongoDB is exactly 'draft'

**Summary**: ✅ Preview Mode implemented! Editors can preview drafts without publishing them.

---

## Step 10: Deploy to Vercel

### What we're doing
Deploying our ISR blog to Vercel (the creators of Next.js) where ISR works out-of-the-box with edge caching.

### Prerequisites

1. **GitHub account** (to push your code)
2. **Vercel account** (free tier works great)

### Prepare the code

1. **Create `.gitignore`** (if not already created):
```bash
# Dependencies
node_modules/
.pnp/

# Next.js
.next/
out/
build/

# Environment
.env
.env.local
.env*.local

# Debug logs
npm-debug.log*
yarn-debug.log*

# OS
.DS_Store
```

2. **Initialize git** (if not already):
```bash
git init
git add .
git commit -m "Initial commit: ISR blog with preview mode"
```

3. **Push to GitHub**:
```bash
# Create a new repo on GitHub (don't initialize with README)
# Then run:
git remote add origin https://github.com/YOUR_USERNAME/next-isr-blog.git
git branch -M main
git push -u origin main
```

### Deploy to Vercel

1. **Visit** [vercel.com](https://vercel.com) and sign in with GitHub

2. **Import your repository**:
   - Click "Add New..." → "Project"
   - Select your `next-isr-blog` repository
   - Click "Import"

3. **Configure environment variables**:
   - In the "Environment Variables" section, add:
   ```
   MONGODB_URI=your_mongodb_connection_string
   MONGODB_DB=next_isr_blog
   REVALIDATION_SECRET=my-secret-token-12345
   PREVIEW_SECRET=preview-secret-12345
   ```
   - Use the same values from your `.env.local`

4. **Deploy**:
   - Click "Deploy"
   - Wait 1-2 minutes for the build to complete

5. **Visit your site**:
   - Vercel will show you a URL like `https://next-isr-blog-xxx.vercel.app`
   - Click it to see your live site!

### Testing ISR on Vercel

1. **Visit your deployed site**
   - Note the "Last built" timestamp

2. **Test time-based ISR**:
   - Wait 10 seconds
   - Refresh the page
   - Timestamp should update (ISR worked!)

3. **Test on-demand revalidation**:
   - Your Express server isn't deployed, so we'll use Vercel's preview

### Update API server for production

The Express server won't work on Vercel (it's serverless), but we can convert it to Next.js API routes.

Create `pages/api/posts/index.ts`:

```typescript
import type { NextApiRequest, NextApiResponse } from 'next';
import { getPostsCollection } from '../../../lib/db';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method === 'GET') {
    try {
      const posts = await getPostsCollection();
      const allPosts = await posts.find({}).sort({ createdAt: -1 }).toArray();
      res.status(200).json(allPosts);
    } catch (error) {
      res.status(500).json({ error: 'Failed to fetch posts' });
    }
  } else if (req.method === 'POST') {
    try {
      const { slug, title, content, author, status = 'published' } = req.body;
      
      if (!slug || !title || !content || !author) {
        return res.status(400).json({ error: 'Missing required fields' });
      }
      
      const posts = await getPostsCollection();
      
      const existing = await posts.findOne({ slug });
      if (existing) {
        return res.status(409).json({ error: 'Slug already exists' });
      }
      
      const newPost = {
        slug,
        title,
        content,
        author,
        status,
        createdAt: new Date(),
        updatedAt: new Date(),
      };
      
      await posts.insertOne(newPost);
      
      // Trigger ISR revalidation
      const baseUrl = process.env.VERCEL_URL 
        ? `https://${process.env.VERCEL_URL}`
        : 'http://localhost:3000';
      const secret = process.env.REVALIDATION_SECRET;
      
      await fetch(`${baseUrl}/api/revalidate?secret=${secret}&path=/`);
      await fetch(`${baseUrl}/api/revalidate?secret=${secret}&path=/posts/${slug}`);
      
      res.status(201).json(newPost);
    } catch (error) {
      res.status(500).json({ error: 'Failed to create post' });
    }
  } else {
    res.status(405).json({ error: 'Method not allowed' });
  }
}
```

Create `pages/api/posts/[slug].ts`:

```typescript
import type { NextApiRequest, NextApiResponse } from 'next';
import { getPostsCollection } from '../../../lib/db';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  const { slug } = req.query;

  if (req.method === 'GET') {
    try {
      const posts = await getPostsCollection();
      const post = await posts.findOne({ slug: slug as string });
      
      if (!post) {
        return res.status(404).json({ error: 'Post not found' });
      }
      
      res.status(200).json(post);
    } catch (error) {
      res.status(500).json({ error: 'Failed to fetch post' });
    }
  } else if (req.method === 'PUT') {
    try {
      const { title, content, author, status } = req.body;
      const posts = await getPostsCollection();
      
      const result = await posts.findOneAndUpdate(
        { slug: slug as string },
        { 
          $set: { 
            title, 
            content, 
            author,
            status,
            updatedAt: new Date() 
          } 
        },
        { returnDocument: 'after' }
      );
      
      if (!result) {
        return res.status(404).json({ error: 'Post not found' });
      }
      
      // Trigger ISR revalidation
      const baseUrl = process.env.VERCEL_URL 
        ? `https://${process.env.VERCEL_URL}`
        : 'http://localhost:3000';
      const secret = process.env.REVALIDATION_SECRET;
      
      await fetch(`${baseUrl}/api/revalidate?secret=${secret}&path=/`);
      await fetch(`${baseUrl}/api/revalidate?secret=${secret}&path=/posts/${slug}`);
      
      res.status(200).json(result);
    } catch (error) {
      res.status(500).json({ error: 'Failed to update post' });
    }
  } else if (req.method === 'DELETE') {
    try {
      const posts = await getPostsCollection();
      const result = await posts.deleteOne({ slug: slug as string });
      
      if (result.deletedCount === 0) {
        return res.status(404).json({ error: 'Post not found' });
      }
      
      // Trigger ISR revalidation
      const baseUrl = process.env.VERCEL_URL 
        ? `https://${process.env.VERCEL_URL}`
        : 'http://localhost:3000';
      const secret = process.env.REVALIDATION_SECRET;
      
      await fetch(`${baseUrl}/api/revalidate?secret=${secret}&path=/`);
      
      res.status(200).json({ message: 'Post deleted successfully' });
    } catch (error) {
      res.status(500).json({ error: 'Failed to delete post' });
    }
  } else {
    res.status(405).json({ error: 'Method not allowed' });
  }
}
```

### Deploy the updates

```bash
git add .
git commit -m "Add Next.js API routes for production"
git push
```

Vercel will automatically redeploy!

### Test on-demand revalidation in production

```bash
# Replace YOUR_SITE with your actual Vercel URL
curl -X PUT https://YOUR_SITE.vercel.app/api/posts/hello-isr \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Hello ISR - Updated on Vercel!",
    "content": "This was updated in production!",
    "author": "John Doe",
    "status": "published"
  }'
```

Visit your site - the post should be updated immediately!

### Test Preview Mode in production

Visit:
```
https://YOUR_SITE.vercel.app/api/preview?secret=preview-secret-12345&redirect=/
```

You should see draft posts!

### Troubleshooting deployment

- **Build fails**: Check the Vercel build logs for errors
- **MongoDB connection fails**: Verify environment variables are set correctly
- **API routes return 500**: Check function logs in Vercel dashboard
- **ISR not working**: Ensure `revalidate` is set in `getStaticProps`

**Summary**: ✅ Deployed to Vercel! ISR, on-demand revalidation, and preview mode all work in production.

---

## Step 11: Performance Monitoring & Best Practices

### What we're doing
Understanding how to monitor ISR performance and optimize for production.

### Key ISR metrics to monitor

**1. Cache Hit Rate**:
- How often ISR serves cached pages vs regenerating
- Higher is better (means less compute)

**2. Regeneration Time**:
- How long it takes to regenerate a page
- Should be under 1-2 seconds for good UX

**3. Stale-While-Revalidate Duration**:
- How long users see stale content
- Depends on your `revalidate` value

### Vercel Analytics (built-in)

1. Go to your project on Vercel
2. Click "Analytics" tab
3. You'll see:
   - Page views
   - Load times
   - Cache hit rates
   - ISR regenerations

### Best practices

**1. Choose appropriate revalidate intervals**:
```typescript
// TOO AGGRESSIVE - expensive
revalidate: 1

// GOOD - balance freshness and cost
revalidate: 60

// TOO LONG - content feels stale
revalidate: 86400
```

**2. Prefer on-demand revalidation**:
```typescript
// Instead of aggressive time-based:
revalidate: 5

// Use moderate time-based + on-demand:
revalidate: 300
// Then call res.revalidate() when content changes
```

**3. Optimize database queries**:
```typescript
// BAD - fetches all fields
const posts = await posts.find({}).toArray();

// GOOD - only fetch needed fields
const posts = await posts.find({}, {
  projection: { title: 1, slug: 1, excerpt: 1 }
}).toArray();
```

**4. Use proper caching headers**:
```typescript
// In getStaticProps, you can also set cache headers
export const getStaticProps: GetStaticProps = async () => {
  return {
    props: { /* ... */ },
    revalidate: 60,
    // These are set automatically by Next.js
  };
};
```

**5. Handle errors gracefully**:
```typescript
export const getStaticProps: GetStaticProps = async () => {
  try {
    const posts = await getPostsCollection();
    const allPosts = await posts.find({}).toArray();
    
    return {
      props: { posts: JSON.parse(JSON.stringify(allPosts)) },
      revalidate: 60,
    };
  } catch (error) {
    console.error('Failed to fetch posts:', error);
    
    // Return empty array instead of failing
    return {
      props: { posts: [] },
      revalidate: 10, // Retry sooner
    };
  }
};
```

**6. Use fallback strategically**:
```typescript
// For large sites with thousands of pages
export const getStaticPaths: GetStaticPaths = async () => {
  const posts = await getPostsCollection();
  
  // Only pre-render top 10 most popular
  const topPosts = await posts
    .find({})
    .sort({ views: -1 })
    .limit(10)
    .toArray();

  return {
    paths: topPosts.map(p => ({ params: { slug: p.slug } })),
    fallback: 'blocking', // Generate others on-demand
  };
};
```

### Common pitfalls to avoid

**❌ Don't**:
- Use ISR for real-time data (use SSR instead)
- Set revalidate too low on high-traffic pages
- Forget to handle database errors
- Return large JSON props (affects performance)
- Use ISR for personalized content (use client-side fetching)

**✅ Do**:
- Use ISR for content that changes occasionally
- Combine time-based and on-demand revalidation
- Monitor regeneration performance
- Cache database connections properly
- Use proper TypeScript types

### Testing checklist

Before deploying to production, verify:

- [ ] ISR regenerates pages after `revalidate` period
- [ ] On-demand revalidation works via API
- [ ] Preview mode shows draft content
- [ ] 404 pages work for non-existent slugs
- [ ] Database connections are pooled properly
- [ ] Error states are handled gracefully
- [ ] Environment variables are set
- [ ] API routes are protected (if needed)

**Summary**: ✅ ISR monitoring and best practices covered. You're ready for production!

---

## Final Summary: What You've Learned

### Core ISR Concepts

1. **Time-based ISR**: Pages regenerate automatically after a time period
2. **On-demand ISR**: Trigger regeneration via API when content changes
3. **Preview Mode**: Bypass cache to preview unpublished content
4. **Fallback Modes**: Control how new pages are generated on-demand

### Project Architecture

```
next-isr-blog/
├── pages/
│   ├── index.tsx              # Home page with ISR
│   ├── posts/[slug].tsx       # Dynamic post pages with ISR
│   └── api/
│       ├── revalidate.ts      # On-demand revalidation endpoint
│       ├── preview.ts         # Enter preview mode
│       ├── exit-preview.ts    # Exit preview mode
│       └── posts/             # CRUD API for posts
├── lib/
│   ├── mongodb.ts             # MongoDB connection
│   └── db.ts                  # Database helpers
├── scripts/
│   └── seed.ts                # Seed sample data
└── server/                    # (Optional) Express server
```

### When to Use ISR

**✅ Great for**:
- Blogs and content sites
- E-commerce product pages
- Documentation sites
- Marketing pages
- Content that updates occasionally

**❌ Not suitable for**:
- Real-time data (stock prices, live scores)
- Personalized content (user dashboards)
- Frequently changing data (every second)
- Content requiring authentication

### Next Steps

1. **Add authentication**: Protect API routes with API keys or JWT
2. **Add an admin panel**: UI for creating/editing posts
3. **Implement search**: Add full-text search to posts
4. **Add images**: Handle image uploads to cloud storage
5. **Add categories/tags**: Organize posts with taxonomy
6. **Improve SEO**: Add metadata, sitemaps, structured data
7. **Add comments**: Enable user interaction

### Resources (continued)

- [Next.js ISR Documentation](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)
- [Vercel ISR Guide](https://vercel.com/docs/concepts/incremental-static-regeneration)
- [MongoDB Node.js Driver Docs](https://www.mongodb.com/docs/drivers/node/current/)
- [Next.js API Routes](https://nextjs.org/docs/pages/building-your-application/routing/api-routes)

---

## Bonus Step: Create an Admin Panel

### What we're doing
Building a simple admin interface to manage posts without using curl commands.

### Create the admin page

Create `pages/admin.tsx`:

```typescript
import { useState, useEffect } from 'react';
import { Post } from '../lib/db';

export default function AdminPanel() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [loading, setLoading] = useState(true);
  const [editing, setEditing] = useState<string | null>(null);
  const [showCreateForm, setShowCreateForm] = useState(false);
  
  // Form state
  const [formData, setFormData] = useState({
    slug: '',
    title: '',
    content: '',
    author: '',
    status: 'draft' as 'draft' | 'published',
  });

  // Fetch all posts
  useEffect(() => {
    fetchPosts();
  }, []);

  const fetchPosts = async () => {
    try {
      const res = await fetch('/api/posts');
      const data = await res.json();
      setPosts(data);
      setLoading(false);
    } catch (error) {
      console.error('Failed to fetch posts:', error);
      setLoading(false);
    }
  };

  // Handle form input changes
  const handleChange = (e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement>) => {
    setFormData({
      ...formData,
      [e.target.name]: e.target.value,
    });
  };

  // Create a new post
  const handleCreate = async (e: React.FormEvent) => {
    e.preventDefault();
    
    try {
      const res = await fetch('/api/posts', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(formData),
      });

      if (res.ok) {
        alert('Post created successfully!');
        setShowCreateForm(false);
        setFormData({ slug: '', title: '', content: '', author: '', status: 'draft' });
        fetchPosts();
      } else {
        const error = await res.json();
        alert(`Error: ${error.error}`);
      }
    } catch (error) {
      alert('Failed to create post');
    }
  };

  // Update an existing post
  const handleUpdate = async (slug: string) => {
    const post = posts.find(p => p.slug === slug);
    if (!post) return;

    try {
      const res = await fetch(`/api/posts/${slug}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          title: post.title,
          content: post.content,
          author: post.author,
          status: post.status,
        }),
      });

      if (res.ok) {
        alert('Post updated successfully!');
        setEditing(null);
        fetchPosts();
      } else {
        alert('Failed to update post');
      }
    } catch (error) {
      alert('Failed to update post');
    }
  };

  // Delete a post
  const handleDelete = async (slug: string) => {
    if (!confirm('Are you sure you want to delete this post?')) return;

    try {
      const res = await fetch(`/api/posts/${slug}`, {
        method: 'DELETE',
      });

      if (res.ok) {
        alert('Post deleted successfully!');
        fetchPosts();
      } else {
        alert('Failed to delete post');
      }
    } catch (error) {
      alert('Failed to delete post');
    }
  };

  // Update local post state while editing
  const updatePostField = (slug: string, field: keyof Post, value: string) => {
    setPosts(posts.map(post => 
      post.slug === slug ? { ...post, [field]: value } : post
    ));
  };

  if (loading) {
    return (
      <div className="min-h-screen bg-gray-50 flex items-center justify-center">
        <div className="text-xl text-gray-600">Loading...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-50 py-12 px-4">
      <div className="max-w-6xl mx-auto">
        <div className="flex justify-between items-center mb-8">
          <div>
            <h1 className="text-4xl font-bold text-gray-900">Admin Panel</h1>
            <p className="text-gray-600 mt-2">Manage your blog posts</p>
          </div>
          <div className="space-x-4">
            
              href="/"
              className="inline-block bg-gray-600 text-white px-4 py-2 rounded hover:bg-gray-700"
            >
              View Site
            </a>
            <button
              onClick={() => setShowCreateForm(!showCreateForm)}
              className="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700"
            >
              {showCreateForm ? 'Cancel' : 'Create New Post'}
            </button>
          </div>
        </div>

        {/* Create Form */}
        {showCreateForm && (
          <div className="bg-white rounded-lg shadow-md p-6 mb-8">
            <h2 className="text-2xl font-bold mb-4">Create New Post</h2>
            <form onSubmit={handleCreate} className="space-y-4">
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-1">
                  Slug (URL-friendly, e.g., "my-awesome-post")
                </label>
                <input
                  type="text"
                  name="slug"
                  value={formData.slug}
                  onChange={handleChange}
                  required
                  pattern="[a-z0-9-]+"
                  className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
                  placeholder="my-awesome-post"
                />
              </div>
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-1">
                  Title
                </label>
                <input
                  type="text"
                  name="title"
                  value={formData.title}
                  onChange={handleChange}
                  required
                  className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
                />
              </div>
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-1">
                  Content
                </label>
                <textarea
                  name="content"
                  value={formData.content}
                  onChange={handleChange}
                  required
                  rows={6}
                  className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
                />
              </div>
              <div className="grid grid-cols-2 gap-4">
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">
                    Author
                  </label>
                  <input
                    type="text"
                    name="author"
                    value={formData.author}
                    onChange={handleChange}
                    required
                    className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
                  />
                </div>
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">
                    Status
                  </label>
                  <select
                    name="status"
                    value={formData.status}
                    onChange={handleChange}
                    className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
                  >
                    <option value="draft">Draft</option>
                    <option value="published">Published</option>
                  </select>
                </div>
              </div>
              <button
                type="submit"
                className="w-full bg-blue-600 text-white py-2 rounded hover:bg-blue-700 font-medium"
              >
                Create Post
              </button>
            </form>
          </div>
        )}

        {/* Posts List */}
        <div className="space-y-4">
          {posts.map((post) => (
            <div key={post.slug} className="bg-white rounded-lg shadow-md p-6">
              {editing === post.slug ? (
                // Edit Mode
                <div className="space-y-4">
                  <input
                    type="text"
                    value={post.title}
                    onChange={(e) => updatePostField(post.slug, 'title', e.target.value)}
                    className="w-full text-2xl font-bold border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
                  />
                  <textarea
                    value={post.content}
                    onChange={(e) => updatePostField(post.slug, 'content', e.target.value)}
                    rows={6}
                    className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
                  />
                  <div className="grid grid-cols-2 gap-4">
                    <input
                      type="text"
                      value={post.author}
                      onChange={(e) => updatePostField(post.slug, 'author', e.target.value)}
                      className="border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
                    />
                    <select
                      value={post.status}
                      onChange={(e) => updatePostField(post.slug, 'status', e.target.value as 'draft' | 'published')}
                      className="border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
                    >
                      <option value="draft">Draft</option>
                      <option value="published">Published</option>
                    </select>
                  </div>
                  <div className="flex space-x-2">
                    <button
                      onClick={() => handleUpdate(post.slug)}
                      className="bg-green-600 text-white px-4 py-2 rounded hover:bg-green-700"
                    >
                      Save Changes
                    </button>
                    <button
                      onClick={() => setEditing(null)}
                      className="bg-gray-600 text-white px-4 py-2 rounded hover:bg-gray-700"
                    >
                      Cancel
                    </button>
                  </div>
                </div>
              ) : (
                // View Mode
                <>
                  <div className="flex justify-between items-start mb-4">
                    <div>
                      <h2 className="text-2xl font-bold text-gray-900">
                        {post.title}
                        <span className={`ml-3 text-sm px-2 py-1 rounded ${
                          post.status === 'published' 
                            ? 'bg-green-100 text-green-800' 
                            : 'bg-yellow-100 text-yellow-800'
                        }`}>
                          {post.status.toUpperCase()}
                        </span>
                      </h2>
                      <p className="text-sm text-gray-600 mt-1">
                        /{post.slug} • By {post.author}
                      </p>
                    </div>
                    <div className="flex space-x-2">
                      
                        href={`/posts/${post.slug}`}
                        target="_blank"
                        rel="noopener noreferrer"
                        className="text-blue-600 hover:text-blue-800 text-sm"
                      >
                        View
                      </a>
                      
                        href={`/api/preview?secret=${process.env.NEXT_PUBLIC_PREVIEW_SECRET || 'preview-secret-12345'}&redirect=/posts/${post.slug}`}
                        target="_blank"
                        rel="noopener noreferrer"
                        className="text-purple-600 hover:text-purple-800 text-sm"
                      >
                        Preview
                      </a>
                      <button
                        onClick={() => setEditing(post.slug)}
                        className="text-blue-600 hover:text-blue-800 text-sm"
                      >
                        Edit
                      </button>
                      <button
                        onClick={() => handleDelete(post.slug)}
                        className="text-red-600 hover:text-red-800 text-sm"
                      >
                        Delete
                      </button>
                    </div>
                  </div>
                  <p className="text-gray-700 line-clamp-2">{post.content}</p>
                  <p className="text-xs text-gray-500 mt-4">
                    Updated: {new Date(post.updatedAt).toLocaleString()}
                  </p>
                </>
              )}
            </div>
          ))}
        </div>

        {posts.length === 0 && (
          <div className="text-center py-12 text-gray-600">
            <p className="text-xl">No posts yet. Create your first post!</p>
          </div>
        )}
      </div>
    </div>
  );
}
```

### Add public environment variable (optional)

For the preview button to work, add to `.env.local`:

```bash
NEXT_PUBLIC_PREVIEW_SECRET=preview-secret-12345
```

Note: `NEXT_PUBLIC_` makes it available in the browser. This is okay for preview secrets in non-production environments.

### Testing the admin panel

1. **Start your app**:
```bash
npm run build
npm run start
```

2. **Visit the admin panel**: `http://localhost:3000/admin`

3. **Try creating a post**:
   - Click "Create New Post"
   - Fill in the form:
     - Slug: `admin-test-post`
     - Title: `Created via Admin Panel`
     - Content: `This post was created through the UI!`
     - Author: `Admin User`
     - Status: `published`
   - Click "Create Post"
   - Visit the home page - your new post should appear!

4. **Try editing a post**:
   - Click "Edit" on any post
   - Modify the title or content
   - Click "Save Changes"
   - Refresh the home page - changes are live!

5. **Test preview mode**:
   - Create a draft post
   - Click "Preview" next to it
   - You'll see the draft in preview mode
   - Exit preview and try viewing it normally (404)

6. **Test delete**:
   - Click "Delete" on a post
   - Confirm the deletion
   - The post is removed from MongoDB and from the site

### Deploy the admin panel

```bash
git add .
git commit -m "Add admin panel"
git push
```

Vercel will automatically deploy!

**Important**: In production, you should add authentication to protect the admin panel. Anyone with the URL can currently access it.

### Quick authentication (optional)

Add basic password protection. Create `middleware.ts` in the project root:

```typescript
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Only protect the admin route
  if (request.nextUrl.pathname.startsWith('/admin')) {
    const basicAuth = request.headers.get('authorization');
    
    if (basicAuth) {
      const auth = basicAuth.split(' ')[1];
      const [user, pwd] = Buffer.from(auth, 'base64').toString().split(':');
      
      // Check credentials (use environment variables in production!)
      if (user === 'admin' && pwd === process.env.ADMIN_PASSWORD) {
        return NextResponse.next();
      }
    }
    
    // Request authentication
    return new NextResponse('Authentication required', {
      status: 401,
      headers: {
        'WWW-Authenticate': 'Basic realm="Admin Area"',
      },
    });
  }
}
```

Add to `.env.local`:
```bash
ADMIN_PASSWORD=your-secure-password-here
```

Now when you visit `/admin`, you'll be prompted for username (`admin`) and password.

**Summary**: ✅ Admin panel created! You can now manage posts through a UI instead of API calls.

---

## Complete Code Reference

### Final project structure

```
next-isr-blog/
├── lib/
│   ├── mongodb.ts              # MongoDB connection
│   └── db.ts                   # Database utilities & types
├── pages/
│   ├── _app.tsx                # Next.js app wrapper
│   ├── index.tsx               # Home page (ISR)
│   ├── admin.tsx               # Admin panel
│   ├── posts/
│   │   └── [slug].tsx          # Dynamic post pages (ISR)
│   └── api/
│       ├── revalidate.ts       # On-demand revalidation
│       ├── preview.ts          # Enter preview mode
│       ├── exit-preview.ts     # Exit preview mode
│       └── posts/
│           ├── index.ts        # GET/POST posts
│           └── [slug].ts       # GET/PUT/DELETE single post
├── scripts/
│   └── seed.ts                 # Seed database
├── middleware.ts               # (Optional) Auth protection
├── .env.local                  # Environment variables
├── package.json
├── tsconfig.json
└── tailwind.config.ts
```

### Key files summary

**Environment variables** (`.env.local`):
```bash
MONGODB_URI=mongodb+srv://...
MONGODB_DB=next_isr_blog
REVALIDATION_SECRET=my-secret-token-12345
PREVIEW_SECRET=preview-secret-12345
ADMIN_PASSWORD=your-password
NEXT_PUBLIC_PREVIEW_SECRET=preview-secret-12345
```

**ISR Configuration** (`getStaticProps`):
```typescript
export const getStaticProps: GetStaticProps = async ({ preview = false }) => {
  // Fetch data from database
  const data = await fetchData();
  
  return {
    props: { data, preview },
    revalidate: 60, // Time-based ISR: 60 seconds
  };
};
```

**On-demand Revalidation** (`/api/revalidate`):
```typescript
await res.revalidate('/'); // Revalidate home page
await res.revalidate('/posts/my-slug'); // Revalidate specific post
```

**Preview Mode** (enter/exit):
```typescript
// Enter: /api/preview?secret=XXX&redirect=/posts/draft
res.setPreviewData({});

// Exit: /api/exit-preview?redirect=/
res.clearPreviewData();
```

---

## Troubleshooting Guide

### Common issues and solutions

**1. "Cannot connect to MongoDB"**
- ✅ Check `.env.local` exists and has correct URI
- ✅ Verify MongoDB Atlas IP whitelist includes your IP
- ✅ Confirm username/password in connection string
- ✅ Test with `node test-db.js`

**2. "ISR not working in development"**
- ✅ ISR is intentionally disabled in `npm run dev`
- ✅ Use `npm run build && npm run start` to test ISR
- ✅ Remember: dev mode regenerates every request

**3. "Page shows old content after update"**
- ✅ Check revalidation was triggered (check API response)
- ✅ Hard refresh browser (Ctrl+Shift+R)
- ✅ Wait for the revalidate period to pass
- ✅ Check "Last built" timestamp on page

**4. "Preview mode not working"**
- ✅ Verify `PREVIEW_SECRET` matches in URL and `.env.local`
- ✅ Check preview cookies are set (browser DevTools → Application → Cookies)
- ✅ Ensure you're testing in production mode (`npm run start`)
- ✅ Clear cookies and try again

**5. "Build fails on Vercel"**
- ✅ Check environment variables are set in Vercel dashboard
- ✅ Review build logs for specific errors
- ✅ Ensure MongoDB is accessible from Vercel (check firewall)
- ✅ Verify all dependencies are in `package.json`

**6. "API routes return 500"**
- ✅ Check Vercel function logs
- ✅ Verify database connection
- ✅ Ensure request body is valid JSON
- ✅ Add error logging to identify the issue

**7. "TypeScript errors"**
- ✅ Run `npm install` to ensure all types are installed
- ✅ Check `tsconfig.json` is configured correctly
- ✅ Verify interface definitions match actual data
- ✅ Use `JSON.parse(JSON.stringify())` for date serialization

**8. "Fallback page shows forever"**
- ✅ Check MongoDB connection during page generation
- ✅ Verify the slug exists in the database
- ✅ Check server logs for errors
- ✅ Ensure `getStaticProps` returns valid data

---

## Performance Optimization Tips

### 1. Optimize database queries

```typescript
// ❌ BAD: Fetches all data
const posts = await posts.find({}).toArray();

// ✅ GOOD: Limit fields and results
const posts = await posts
  .find({}, { projection: { _id: 0, title: 1, slug: 1, excerpt: 1 } })
  .limit(10)
  .toArray();
```

### 2. Implement pagination

```typescript
export const getStaticProps: GetStaticProps = async ({ params }) => {
  const page = parseInt(params?.page as string) || 1;
  const limit = 10;
  const skip = (page - 1) * limit;
  
  const posts = await getPostsCollection();
  const results = await posts
    .find({ status: 'published' })
    .sort({ createdAt: -1 })
    .skip(skip)
    .limit(limit)
    .toArray();
  
  return {
    props: { posts: JSON.parse(JSON.stringify(results)) },
    revalidate: 60,
  };
};
```

### 3. Add caching headers

```typescript
// In API routes
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  // Cache for 60 seconds, stale-while-revalidate for 120
  res.setHeader('Cache-Control', 's-maxage=60, stale-while-revalidate=120');
  
  const data = await fetchData();
  res.json(data);
}
```

### 4. Optimize images

Use Next.js Image component:

```typescript
import Image from 'next/image';

<Image 
  src="/post-image.jpg" 
  alt="Post" 
  width={800} 
  height={400}
  priority // For above-the-fold images
/>
```

### 5. Pre-render popular pages only

```typescript
export const getStaticPaths: GetStaticPaths = async () => {
  const posts = await getPostsCollection();
  
  // Only pre-render top 20 most viewed posts
  const popularPosts = await posts
    .find({ status: 'published' })
    .sort({ views: -1 })
    .limit(20)
    .toArray();

  return {
    paths: popularPosts.map(post => ({ params: { slug: post.slug } })),
    fallback: 'blocking', // Others generated on-demand
  };
};
```

### 6. Monitor performance

Add logging to track regeneration times:

```typescript
export const getStaticProps: GetStaticProps = async () => {
  const start = Date.now();
  
  const data = await fetchData();
  
  const duration = Date.now() - start;
  console.log(`Page regenerated in ${duration}ms`);
  
  return {
    props: { data },
    revalidate: 60,
  };
};
```

---

## Real-World Use Cases

### Use Case 1: E-commerce Product Pages

```typescript
// Product page with ISR
export const getStaticProps: GetStaticProps = async ({ params }) => {
  const product = await getProduct(params.id);
  
  return {
    props: { product },
    revalidate: 300, // Update every 5 minutes
  };
};

// Trigger revalidation when inventory changes
await res.revalidate(`/products/${productId}`);
```

### Use Case 2: News Site

```typescript
// News article with aggressive ISR
export const getStaticProps: GetStaticProps = async ({ params }) => {
  const article = await getArticle(params.slug);
  
  return {
    props: { article },
    revalidate: 60, // Fresh content every minute
  };
};
```

### Use Case 3: Documentation Site

```typescript
// Docs with long revalidation
export const getStaticProps: GetStaticProps = async ({ params }) => {
  const doc = await getDocumentation(params.slug);
  
  return {
    props: { doc },
    revalidate: 3600, // Update hourly
  };
};

// Manual revalidation when docs are edited
await res.revalidate(`/docs/${docSlug}`);
```

---

## Congratulations! 🎉

You've successfully learned:

- ✅ Core ISR concepts (time-based and on-demand)
- ✅ How to build an ISR-powered Next.js app
- ✅ MongoDB integration for dynamic content
- ✅ Preview Mode for draft content
- ✅ API routes for content management
- ✅ Admin panel for easier content editing
- ✅ Deployment to Vercel
- ✅ Performance optimization techniques
- ✅ Best practices and troubleshooting

### What's next?

**Enhance your project:**
1. Add authentication (NextAuth.js, Auth0, or Clerk)
2. Implement image uploads (Cloudinary, AWS S3)
3. Add search functionality (Algolia, Meilisearch)
4. Create categories and tags
5. Add comments system
6. Implement analytics (Vercel Analytics, Google Analytics)
7. Add SEO metadata and sitemaps
8. Create RSS feeds

**Learn more:**
- Explore App Router (Next.js 13+) with ISR
- Learn about Edge Functions
- Study incremental adoption strategies
- Experiment with different caching strategies

You now have a solid foundation in ISR and can build production-ready applications with Next.js!
