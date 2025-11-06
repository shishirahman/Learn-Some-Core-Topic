# TypeScript Concepts Used in This Project

I'll explain only the TypeScript features we actually used, step by step, with simple examples.

---

## 1. Basic Types

### What are types?
Types tell TypeScript what kind of data a variable can hold. This helps catch errors before running the code.

### String, Number, Boolean

```typescript
// Basic types
let title: string = "Hello ISR";  // Can only be text
let views: number = 100;          // Can only be numbers
let isPublished: boolean = true;  // Can only be true/false

// TypeScript will catch errors:
title = 123;  // ❌ Error: Type 'number' is not assignable to type 'string'
```

**Why we need this:**
```typescript
// Without types (JavaScript):
let age = "25";
let nextAge = age + 1;  // "251" - Oops! String concatenation

// With types (TypeScript):
let age: number = 25;
let nextAge = age + 1;  // 26 - Correct!
```

### Arrays

```typescript
// Array of strings
let tags: string[] = ["javascript", "typescript", "nextjs"];

// Array of numbers
let ids: number[] = [1, 2, 3, 4];

// TypeScript prevents mistakes:
tags.push(123);  // ❌ Error: Argument of type 'number' is not assignable
tags.push("react");  // ✅ OK
```

**In our project:**
```typescript
// We used arrays for posts
const posts: Post[] = []; // Array of Post objects
```

---

## 2. Interfaces

### What is an interface?
An interface defines the **shape** of an object - what properties it should have and what type each property is.

### Simple example

```typescript
// Define the shape of a Person
interface Person {
  name: string;
  age: number;
}

// Create objects that match this shape
const john: Person = {
  name: "John",
  age: 30
};

const jane: Person = {
  name: "Jane",
  age: 25
};

// TypeScript catches mistakes:
const invalid: Person = {
  name: "Bob"
  // ❌ Error: Property 'age' is missing
};
```

### Optional properties

```typescript
interface Person {
  name: string;
  age: number;
  email?: string;  // ❓ The ? means optional
}

// Both are valid:
const person1: Person = {
  name: "Alice",
  age: 28,
  email: "alice@example.com"  // email included
};

const person2: Person = {
  name: "Bob",
  age: 35
  // email omitted - that's OK because it's optional
};
```

**In our project:**
```typescript
// Our Post interface
export interface Post {
  _id?: string;        // Optional - MongoDB adds this
  slug: string;        // Required
  title: string;       // Required
  content: string;     // Required
  author: string;      // Required
  createdAt: Date;     // Required
  updatedAt: Date;     // Required
  status: 'draft' | 'published';  // Must be one of these two
}
```

### Why interfaces matter

```typescript
// Without interface:
function displayPost(post) {
  console.log(post.title);  // What if post doesn't have title?
  console.log(post.autor);  // Typo! But no error until runtime
}

// With interface:
interface Post {
  title: string;
  author: string;
}

function displayPost(post: Post) {
  console.log(post.title);  // ✅ TypeScript knows this exists
  console.log(post.autor);  // ❌ Error: Property 'autor' does not exist
}
```

---

## 3. Type Annotations for Functions

### Basic function types

```typescript
// Annotate parameters and return type
function add(a: number, b: number): number {
  return a + b;
}

// TypeScript catches errors:
add(5, 10);      // ✅ Returns 15
add("5", "10");  // ❌ Error: Argument of type 'string' is not assignable
```

### Void return type

```typescript
// When a function doesn't return anything
function logMessage(message: string): void {
  console.log(message);
  // No return statement
}
```

### Async functions

```typescript
// Async functions return a Promise
async function fetchData(): Promise<string> {
  const response = await fetch('/api/data');
  const text = await response.text();
  return text;  // Returns Promise<string>
}
```

**In our project:**
```typescript
// From our seed script
async function seed(): Promise<void> {
  const posts = await getPostsCollection();
  await posts.insertMany(samplePosts);
  // Returns Promise<void> (nothing)
}
```

### Function parameter types

```typescript
interface User {
  name: string;
  email: string;
}

// Function takes a User object
function greetUser(user: User): string {
  return `Hello, ${user.name}!`;
}

// Usage:
const user = { name: "Alice", email: "alice@example.com" };
greetUser(user);  // ✅ "Hello, Alice!"

const invalid = { name: "Bob" };
greetUser(invalid);  // ❌ Error: Property 'email' is missing
```

---

## 4. Union Types

### What are union types?
A variable can be **one of several types**.

### Simple example

```typescript
// Can be string OR number
let id: string | number;

id = "abc123";  // ✅ OK
id = 456;       // ✅ OK
id = true;      // ❌ Error: Type 'boolean' is not assignable
```

### Literal union types

```typescript
// Can ONLY be one of these exact values
type Status = 'draft' | 'published' | 'archived';

let postStatus: Status;

postStatus = 'draft';      // ✅ OK
postStatus = 'published';  // ✅ OK
postStatus = 'pending';    // ❌ Error: not one of the allowed values
```

**In our project:**
```typescript
interface Post {
  // ... other properties
  status: 'draft' | 'published';  // ONLY these two values allowed
}

// This works:
const post: Post = {
  slug: "test",
  title: "Test",
  content: "Content",
  author: "Me",
  status: 'draft',  // ✅ OK
  createdAt: new Date(),
  updatedAt: new Date()
};

// This fails:
const invalid: Post = {
  // ... other properties
  status: 'pending'  // ❌ Error: '"pending"' is not assignable
};
```

---

## 5. Type Inference

### What is type inference?
TypeScript can **automatically figure out** types without you specifying them.

### Examples

```typescript
// TypeScript infers this is a string
let message = "Hello";
message = 123;  // ❌ Error: TypeScript knows it's a string

// TypeScript infers this is a number
let count = 0;
count = "zero";  // ❌ Error: TypeScript knows it's a number

// TypeScript infers the return type
function double(n: number) {  // Returns number (inferred)
  return n * 2;
}
```

### When to use inference

```typescript
// ❌ Too verbose (unnecessary type annotation):
const name: string = "Alice";

// ✅ Better (let TypeScript infer):
const name = "Alice";  // TypeScript knows it's a string

// ✅ But do annotate function parameters:
function greet(name: string) {  // Parameter needs type
  const message = `Hello, ${name}`;  // Return type inferred
  return message;
}
```

**In our project:**
```typescript
// TypeScript infers these types:
const samplePosts = [
  {
    slug: 'hello-isr',
    title: 'Hello ISR!',
    // TypeScript knows this entire object structure
  }
];
```

---

## 6. Generics (Basic)

### What are generics?
Generics let you write **reusable code** that works with multiple types.

### Simple example

```typescript
// Without generics - need separate functions:
function getFirstString(arr: string[]): string {
  return arr[0];
}

function getFirstNumber(arr: number[]): number {
  return arr[0];
}

// With generics - one function for all:
function getFirst<T>(arr: T[]): T {
  return arr[0];
}

// Usage:
const names = ["Alice", "Bob"];
const first = getFirst<string>(names);  // Returns string

const numbers = [1, 2, 3];
const firstNum = getFirst<number>(numbers);  // Returns number
```

### Generic interfaces

```typescript
// Generic "Box" that can hold any type
interface Box<T> {
  value: T;
}

// Box holding a string
const stringBox: Box<string> = {
  value: "Hello"
};

// Box holding a number
const numberBox: Box<number> = {
  value: 42
};
```

**In our project - Next.js types:**
```typescript
// GetStaticProps is generic - we specify what props we return
export const getStaticProps: GetStaticProps<HomeProps> = async () => {
  //                                        ^^^^^^^^^ 
  //                                        Our specific type
  
  return {
    props: {
      posts: [],        // TypeScript knows this should match HomeProps
      lastBuilt: "",
      preview: false
    }
  };
};
```

### How GetStaticProps works

```typescript
// Simplified version of Next.js type:
type GetStaticProps<Props> = () => Promise<{
  props: Props;
  revalidate?: number;
}>;

// When we use it:
interface HomeProps {
  posts: Post[];
  lastBuilt: string;
}

const getStaticProps: GetStaticProps<HomeProps> = async () => {
  return {
    props: {
      posts: [],       // Must be Post[]
      lastBuilt: ""    // Must be string
    }
  };
};
```

---

## 7. Type Assertions

### What are type assertions?
Telling TypeScript "trust me, I know what type this is."

### The `as` keyword

```typescript
// TypeScript doesn't know the specific type
const input = document.getElementById('username');  // HTMLElement | null

// We tell TypeScript it's specifically an input
const inputElement = input as HTMLInputElement;
inputElement.value = "Alice";  // Now we can use input-specific properties
```

### Using `as const`

```typescript
// Without 'as const':
const post = {
  status: 'draft'  // Type: string (any string)
};

// With 'as const':
const post = {
  status: 'draft' as const  // Type: 'draft' (exactly this value)
};
```

**In our project:**
```typescript
const samplePosts = [
  {
    slug: 'hello-isr',
    title: 'Hello ISR!',
    status: 'published' as const,  // Exactly 'published', not any string
    // ...
  }
];
```

---

## 8. Null and Undefined

### Handling missing values

```typescript
// A variable that might not exist
let username: string | null = null;

username = "Alice";  // ✅ OK
username = null;     // ✅ OK
username = 123;      // ❌ Error
```

### Optional chaining

```typescript
interface User {
  name: string;
  address?: {
    city: string;
  };
}

const user: User = { name: "Alice" };

// ❌ Unsafe - might crash if address is undefined:
console.log(user.address.city);

// ✅ Safe - returns undefined if address doesn't exist:
console.log(user.address?.city);
```

**In our project:**
```typescript
// From our post page:
export const getStaticProps: GetStaticProps = async ({ params }) => {
  const post = await posts.findOne({ 
    slug: params?.slug as string  // ? because params might be undefined
  });
};
```

### Nullish coalescing

```typescript
// Provide a default value if something is null/undefined
const username = user.name ?? "Guest";

// Examples:
null ?? "default"       // "default"
undefined ?? "default"  // "default"
"Alice" ?? "default"    // "Alice"
0 ?? "default"          // 0 (doesn't use default for 0)
```

---

## 9. React Component Props

### Typing React components

```typescript
// Define what props the component expects
interface ButtonProps {
  text: string;
  onClick: () => void;
  disabled?: boolean;
}

// Use the props type in the component
function Button({ text, onClick, disabled }: ButtonProps) {
  return (
    <button onClick={onClick} disabled={disabled}>
      {text}
    </button>
  );
}

// Usage:
<Button text="Click me" onClick={() => console.log('Clicked')} />
```

### With state

```typescript
import { useState } from 'react';

interface Post {
  id: string;
  title: string;
}

function PostList() {
  // Specify the type of state
  const [posts, setPosts] = useState<Post[]>([]);
  //                                 ^^^^^^ Array of Posts
  
  // TypeScript knows posts is Post[]
  posts.map(post => post.title);  // ✅ Knows about .title
}
```

**In our project:**
```typescript
interface HomeProps {
  posts: Post[];
  lastBuilt: string;
  preview: boolean;
}

export default function Home({ posts, lastBuilt, preview }: HomeProps) {
  // TypeScript knows the types of these props
  return (
    <div>
      {posts.map(post => (  // TypeScript knows post is a Post
        <h2>{post.title}</h2>
      ))}
    </div>
  );
}
```

---

## 10. Next.js Specific Types

### GetStaticProps

```typescript
import { GetStaticProps } from 'next';

interface PageProps {
  data: string;
}

// Type provided by Next.js
export const getStaticProps: GetStaticProps<PageProps> = async (context) => {
  // context is typed automatically
  // context.params, context.preview, etc.
  
  return {
    props: {
      data: "Hello"  // Must match PageProps
    },
    revalidate: 60
  };
};
```

### GetStaticPaths

```typescript
import { GetStaticPaths } from 'next';

export const getStaticPaths: GetStaticPaths = async () => {
  return {
    paths: [
      { params: { slug: 'post-1' } },
      { params: { slug: 'post-2' } }
    ],
    fallback: 'blocking'
  };
};
```

### API Routes

```typescript
import type { NextApiRequest, NextApiResponse } from 'next';

// Response data type
interface ResponseData {
  message: string;
}

export default async function handler(
  req: NextApiRequest,      // Request type
  res: NextApiResponse<ResponseData>  // Response type
) {
  res.status(200).json({ 
    message: 'Hello' 
  });
}
```

**In our project:**
```typescript
export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  // TypeScript knows about req.method, req.query, etc.
  if (req.method === 'POST') {
    const { title, content } = req.body;
    // ...
  }
}
```

---

## 11. Type vs Interface

### When to use which?

Both are similar, but here's a simple guide:

```typescript
// ✅ Use INTERFACE for object shapes:
interface User {
  name: string;
  email: string;
}

// ✅ Use TYPE for unions, primitives, tuples:
type Status = 'active' | 'inactive';
type ID = string | number;
type Coordinates = [number, number];
```

### Extending

```typescript
// Interfaces can extend:
interface Animal {
  name: string;
}

interface Dog extends Animal {
  breed: string;
}

// Types can intersect:
type Animal = {
  name: string;
};

type Dog = Animal & {
  breed: string;
};
```

**In our project, we used interfaces** because we're defining object shapes (Post, Props, etc.).

---

## 12. JSON.parse(JSON.stringify()) Pattern

### Why do we do this?

```typescript
// MongoDB returns a Post with Date objects
const post = await posts.findOne({ slug: 'hello' });
console.log(post.createdAt);  // Date object

// Next.js props must be JSON-serializable
return {
  props: {
    post: post  // ❌ Error: Date objects can't be serialized
  }
};

// Solution: Convert to JSON and back
return {
  props: {
    post: JSON.parse(JSON.stringify(post))  // ✅ Dates become strings
  }
};
```

### What it does

```typescript
const original = {
  title: "Hello",
  date: new Date("2024-01-01")
};

// Step 1: JSON.stringify converts to string
const jsonString = JSON.stringify(original);
// '{"title":"Hello","date":"2024-01-01T00:00:00.000Z"}'

// Step 2: JSON.parse converts back to object
const serialized = JSON.parse(jsonString);
// {
//   title: "Hello",
//   date: "2024-01-01T00:00:00.000Z"  // Now a STRING, not Date
// }
```

---

## 13. Practical Examples from Our Project

### Example 1: Post interface usage

```typescript
// Define the shape
interface Post {
  slug: string;
  title: string;
  content: string;
  author: string;
  status: 'draft' | 'published';
  createdAt: Date;
  updatedAt: Date;
}

// Use it in a function
async function getPost(slug: string): Promise<Post | null> {
  const posts = await getPostsCollection();
  const post = await posts.findOne({ slug });
  return post;
}

// Use it in a component
interface PostPageProps {
  post: Post;
}

function PostPage({ post }: PostPageProps) {
  return <h1>{post.title}</h1>;
}
```

### Example 2: Form handling

```typescript
// State type
const [formData, setFormData] = useState<{
  title: string;
  content: string;
  author: string;
}>({
  title: '',
  content: '',
  author: ''
});

// Event handler type
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  setFormData({
    ...formData,
    [e.target.name]: e.target.value
  });
};
```

### Example 3: API response handling

```typescript
// Define response type
interface ApiResponse {
  success: boolean;
  data?: Post;
  error?: string;
}

// Fetch with type
async function createPost(postData: Partial<Post>): Promise<ApiResponse> {
  const response = await fetch('/api/posts', {
    method: 'POST',
    body: JSON.stringify(postData)
  });
  
  const result: ApiResponse = await response.json();
  return result;
}
```

---

## 14. Common TypeScript Errors & Solutions

### Error 1: "Property does not exist"

```typescript
// ❌ Error: Property 'title' does not exist on type '{}'
const post = {};
console.log(post.title);

// ✅ Solution: Define the type
interface Post {
  title: string;
}

const post: Post = {
  title: "Hello"
};
console.log(post.title);
```

### Error 2: "Type is not assignable"

```typescript
// ❌ Error: Type 'string' is not assignable to type 'number'
let age: number = "25";

// ✅ Solution: Use the correct type
let age: number = 25;
```

### Error 3: "Possibly undefined"

```typescript
// ❌ Error: Object is possibly 'undefined'
const post = await posts.findOne({ slug: 'hello' });
console.log(post.title);

// ✅ Solution 1: Check if it exists
if (post) {
  console.log(post.title);
}

// ✅ Solution 2: Use optional chaining
console.log(post?.title);

// ✅ Solution 3: Provide default
const title = post?.title ?? 'Untitled';
```

### Error 4: "Missing properties"

```typescript
interface Post {
  title: string;
  content: string;
}

// ❌ Error: Property 'content' is missing
const post: Post = {
  title: "Hello"
};

// ✅ Solution: Add missing property
const post: Post = {
  title: "Hello",
  content: "World"
};
```

---

## 15. Quick Reference Cheat Sheet

```typescript
// BASIC TYPES
let name: string = "Alice";
let age: number = 25;
let isActive: boolean = true;
let numbers: number[] = [1, 2, 3];

// INTERFACE
interface User {
  name: string;
  age: number;
  email?: string;  // Optional
}

// FUNCTION
function greet(name: string): string {
  return `Hello, ${name}`;
}

// ASYNC FUNCTION
async function fetchData(): Promise<string> {
  return "data";
}

// UNION TYPES
let id: string | number;
type Status = 'active' | 'inactive';

// GENERICS
function identity<T>(value: T): T {
  return value;
}

// NULL/UNDEFINED
let value: string | null = null;
let optionalValue = user?.name;
let withDefault = value ?? "default";

// TYPE ASSERTION
const input = element as HTMLInputElement;

// REACT COMPONENT
interface Props {
  title: string;
}

function Component({ title }: Props) {
  return <h1>{title}</h1>;
}

// NEXT.JS
import { GetStaticProps } from 'next';

export const getStaticProps: GetStaticProps<Props> = async () => {
  return {
    props: { title: "Hello" }
  };
};
```

---

## Summary

### The TypeScript we actually used:

1. **Basic types** (string, number, boolean) - for variables
2. **Interfaces** - to define object shapes like `Post`
3. **Function types** - to specify parameter and return types
4. **Union types** - for `'draft' | 'published'` status
5. **Optional properties** (`?`) - for fields that might not exist
6. **Generics** - for Next.js types like `GetStaticProps<Props>`
7. **Array types** - for `Post[]` arrays
8. **Async/Promise** - for database operations
9. **React types** - for component props
10. **Next.js types** - for `GetStaticProps`, `NextApiRequest`, etc.

### Why TypeScript helps:

- ✅ **Catches errors early** - before you run the code
- ✅ **Better autocomplete** - your editor knows what properties exist
- ✅ **Self-documenting** - interfaces show what data looks like
- ✅ **Refactoring confidence** - rename something, TypeScript finds all uses
- ✅ **Team collaboration** - everyone knows the expected data shape

### Remember:

- Start with basic types, add complexity only when needed
- Let TypeScript infer when obvious
- Use interfaces for object shapes
- Use type annotations for function parameters
- Read error messages carefully - they usually tell you exactly what's wrong

You don't need to know every TypeScript feature - just these core concepts are enough for most Next.js projects!
