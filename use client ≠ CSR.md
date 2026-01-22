# “use client” ≠ CSR

---

### 1️⃣ First, “use client” ≠ CSR

When you mark a component as `"use client"`:

* It **becomes a Client Component**
* It **can run hooks like `useState`, `useEffect`**
* It **can use browser APIs** (`window`, `localStorage`, etc.)

**But:**

* It **still renders HTML on the server first** if it’s inside a Server Component
* Then it **hydrates on the client** → attaches interactivity

So **just `"use client"` alone does not make it CSR**. It’s still **SSR + hydration** by default.

---

### 2️⃣ When it actually behaves like CSR

CSR happens only if:

1. You **don’t render anything meaningful on the server** (or defer rendering)
2. You **fetch or compute data only in `useEffect`**

Example mental model:

* **Client Component without `useEffect`**

  * Server renders: `<div>0</div>`
  * Browser hydrates: `<div>0</div>` → SSR + hydration

* **Client Component with `useEffect` data fetch**

  * Server renders: `<div></div>` (placeholder)
  * Browser fetches data → `<div>42</div>`
  * Looks like CSR because **HTML initially empty, data comes later**

So **`useEffect` is the main tool that triggers client-only behavior** after the first render.



