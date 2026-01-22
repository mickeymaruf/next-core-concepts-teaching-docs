# How Hydration Works in Next.js
---

## 1️⃣ Server first

When you have a **Client Component**, e.g.:

```tsx
"use client";
function Button() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

* **Server**:

  * It can render only **the static HTML** of this component if it’s used inside a Server Component.
  * But the **logic inside hooks (`useState`, `useEffect`) is never executed** on the server.
  * Next.js will output a **placeholder HTML** like:

```html
<button>0</button>
```

---

## 2️⃣ Browser receives HTML

* The browser **renders the HTML immediately**
* User sees `<button>0</button>`
* ✅ Fast first paint, even before JS loads

This is why SSG or SSR feels instant.

---

## 3️⃣ Hydration step

Now React JS loads on the client. Hydration is exactly this:

1. **React reads the HTML DOM** that the server sent
2. **Mounts the component logic** on top of it

   * Hooks (`useState`, `useEffect`) are initialized
   * Event listeners (`onClick`) are attached
3. **React checks**: does the HTML from the server match what my component would render on first render?

   * If yes → everything is fine
   * If no → **hydration error**

---

### Visual concept:

```
Server (SSR/SSG)
  └── HTML: <button>0</button>
  
Browser receives HTML
  └── Paints immediately: <button>0</button>

Hydration (client)
  └── React mounts Button
      ├── useState initialized (0)
      ├── onClick attached
      ├── HTML checked → matches
```

✅ User sees HTML instantly + interactivity works

---

### 4️⃣ Where mismatch can happen

* If `Button` instead renders `count` = `Math.random()` initially:

```
Server HTML: <button>0.45</button>
Client first render: <button>0.87</button>
```

➡️ Hydration fails because React expects **0.45**, got **0.87**.

---

### 5️⃣ Key takeaway

* Hydration in **client components** = **attaching the React state + interactivity to the already-rendered HTML**
* Client components **always hydrate**, even if inside Server Components
* Server Components **never hydrate**; they’re purely HTML