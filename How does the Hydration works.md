# How does the Hydration works in Next.js

---

## 1️⃣ What “hydration” actually means (plain English)

> **Hydration = attaching JavaScript behavior to already-rendered HTML**

Think of it like this:

* Server sends **dead HTML** 🧱
* Browser shows it instantly (fast, SEO-friendly)
* React then **“wires up”**:

  * event handlers
  * state
  * effects

➡️ Now it becomes **interactive**

---

## 2️⃣ Step-by-step: what happens in Next.js

### 🔹 Step 1 — Server render (RSC phase)

On the **server**:

* Server Components run
* Data is fetched
* JSX → HTML

```tsx
// Server Component
export default async function Page() {
  const posts = await getPosts();
  return <PostList posts={posts} />;
}
```

At this point:

* No JS
* No hooks
* Just HTML

---

### 🔹 Step 2 — Client Component boundaries are marked

When Next.js sees:

```tsx
"use client";
```

It:

* **Does NOT render it on the server**
* Inserts a **placeholder marker** into HTML

Something like:

```html
<!-- client boundary -->
<div data-rsc-id="123"></div>
```

This tells the browser:

> “JS will hydrate this part later”

---

### 🔹 Step 3 — Browser receives HTML (first paint)

User sees:

* layout
* text
* content

Even client components appear as **static markup** initially.

✔ Fast
✔ SEO-friendly

---

### 🔹 Step 4 — JS bundle loads

Browser downloads:

* React
* Client Component JS
* Providers
* Event logic

---

### 🔹 Step 5 — Hydration starts

React walks the HTML tree:

* Finds client boundaries
* Matches JSX with existing DOM
* Attaches:

  * `onClick`
  * `useState`
  * `useEffect`

```tsx
<button onClick={...}>Click</button>
```

Now clicking works.

---

## 6️⃣ Hydration ≠ Rendering again

This is crucial:

❌ Hydration is NOT:

```txt
rerender everything in browser
```

✅ It is:

```txt
match existing HTML → attach listeners
```

That’s why mismatches cause errors.
