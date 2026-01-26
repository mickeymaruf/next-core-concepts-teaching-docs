# Should we put `try/catch` *inside* services?
---

Short answer: **usually no** — don’t put `try/catch` *inside* services by default.

---

## The rule of thumb

- **Services** should throw errors.
- **Boundaries** should catch them.

**Services** = business/data logic;
**Boundaries** = API routes, Server Actions, UI

---

## Why you generally *don’t* catch inside services

If you do this:

```ts
export async function getUser(id: string) {
  try {
    const res = await fetch(...)
    return res.json()
  } catch {
    return null // ❌
  }
}
```

You create problems:

* ❌ You hide the real error
* ❌ Callers don’t know *what* failed
* ❌ You lose stack traces
* ❌ Every caller must guess what `null` means

Services should be **honest**:

> “Something went wrong — deal with it.”

---

## The preferred pattern ✅

### Service: validate & throw

```ts
// services/user.service.ts
export async function getUser(id: string) {
  const res = await fetch(`https://api.example.com/users/${id}`)

  // fetch doesn't throw on 4xx/5xx — prevent parsing HTML error responses as JSON
  if (!res.ok) {
    throw new Error(`Failed to fetch user (${res.status})`)
  }

  return res.json()
}
```

✔ No `try/catch`
✔ Clear failure signal
✔ Easy to test

---

### Boundary: catch & translate

#### API route

```ts
export async function GET() {
  try {
    const user = await getUser("123")
    return NextResponse.json(user)
  } catch (err) {
    return NextResponse.json(
      { error: "User not found" },
      { status: 404 }
    )
  }
}
```

#### Server Component

```ts
try {
  const user = await getUser("123")
  return <User user={user} />
} catch {
  return <ErrorState />
}
```

#### Server Action

```ts
export async function createUser(data: FormData) {
  try {
    await createUserService(data)
  } catch (err) {
    return { error: "Could not create user" }
  }
}
```

---

## When *should* you use try/catch inside a service?

There **are** legit cases 👇

### 1️⃣ To add context, then rethrow

```ts
export async function getUser(id: string) {
  try {
    return await fetchUser(id)
  } catch (err) {
    throw new Error(`getUser failed for id=${id}`, { cause: err })
  }
}
```

✔ Adds debugging value
✔ Still throws

---

### 2️⃣ To normalize third-party weirdness

```ts
try {
  const res = await fetch(...)
  if (res.status === 404) return null
  if (!res.ok) throw new Error()
} catch (err) {
  throw new ExternalApiError(err)
}
```

Here the service is **intentionally defining behavior**.

---

### 3️⃣ Retry / fallback logic

```ts
try {
  return await primaryApi()
} catch {
  return await backupApi()
}
```

Still fine — but **don’t swallow errors silently**.

---

## Mental model (easy to remember)

* ❌ Services: “I’ll handle it”
* ✅ Services: “Here’s what happened”
* ✅ Boundaries: “Here’s how we respond”

---

## One-line recommendation

> **Don’t catch in services unless you’re adding meaning or transforming the error — otherwise, let it throw and handle it at the boundary.**
