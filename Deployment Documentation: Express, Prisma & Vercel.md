# 🚀 Deployment Documentation: Express, Prisma & Vercel

This guide covers how to deploy a TypeScript Express server using `tsup` to bundle your code into a single ES Module for Vercel’s serverless environment.

### 1. Prerequisites
Ensure your project has the following structure and `package.json` settings:
* **Main Entry:** `src/index.ts`
* **Type:** `"type": "module"` in `package.json`
* **Dependencies:** `express`, `prisma`, `better-auth`, `tsup`, `tsx`.

---

### 2. Configuration Files

#### `tsconfig.json`
Use the **Bundler** resolution to remove VS Code errors while letting `tsup` handle the build.

```json
{
  "compilerOptions": {
    "target": "es2023",
    "module": "esnext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "./",

    // "lib": ["es2023"], (optional)
    // "types": ["node"], (optional)
    // noEmit: true: (optional) // Since tsup is your "heavy lifter" for the production build, you don't want tsc trying to generate files at the same time. This turns TypeScript into a pure "type-checker" for your editor.
  },
  "include": ["src", "prisma.config.ts", "generated/prisma"],
  "exclude": ["node_modules", "dist"]
}
```

#### `vercel.json`
Instruct Vercel to use your compiled JavaScript file.

```json
{
  "version": 2,
  "builds": [
    {
      "src": "dist/index.js",
      "use": "@vercel/node"
    }
  ],
  "routes": [
    {
      "src": "/(.*)",
      "dest": "dist/index.js"
    }
  ]
}
```

---

### 3. Update `package.json` Scripts
Vercel needs to generate the Prisma client and bundle your code before deploying.

```json
"scripts": {
  "dev": "tsx watch src/index.ts",
  "build": "prisma generate && tsup src/index.ts --format esm --clean",
  "vercel-build": "prisma generate && tsup src/index.ts --format esm --clean",
  "start": "node dist/index.js",
  "postinstall": "prisma generate"
}
```

---

### 4. Code Adjustments (`src/index.ts`)
Vercel requires a **default export** of your Express app and manages its own ports.

```typescript
import express from "express";
const app = express();

// Only listen locally, Vercel handles the port in production
if (process.env.NODE_ENV !== "production") {
  const PORT = process.env.PORT || 5000;
  app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
}

export default app; 
```

**(Optional) Why do we check `process.env.NODE_ENV !== "production"` before calling `app.listen`?**
* **Locally:** You need `app.listen` to open a port so you can visit `localhost:5000`.
* **On Vercel:** Vercel "listens" for you. If your code tries to start its own listener on a fixed port, the serverless function might crash or hang because it doesn't have permission to manage the hardware ports directly.

---

### (Not sure why?) One last check: `package.json`
Since you're using `ESNext`/`NodeNext`, make sure your `package.json` includes:
```json
"type": "module"
```
If you don't have `"type": "module"`, you should change the `module` and `moduleResolution` in your `tsconfig.json` to `CommonJS` and `node` respectively.

---

### 5. Deployment Steps

#### Step 1: Login and Link
```bash
vercel login
vercel link
```

#### Step 2: Set Environment Variables
Add your database connection and any authentication secrets to the Vercel dashboard or via CLI:
```bash
vercel env add DATABASE_URL
vercel env add BETTER_AUTH_SECRET
```

#### Step 3: Production Push
Run the production deployment. Vercel will trigger your `vercel-build` script automatically.
```bash
vercel --prod
```

---

### 6. Post-Deployment Checklist
* **Database Migrations:** If your database schema changed, run `npx prisma migrate deploy` locally (pointing to your production DB) or add it to your `vercel-build` script.
* **CORS:** Ensure your Vercel URL is added to your CORS allow-list in `src/index.ts`.
* **Better-Auth URL:** Update your `BETTER_AUTH_URL` environment variable to match your new `.vercel.app` domain.

---

That is a great call. Understanding the "why" behind these configuration choices makes it much easier to troubleshoot later if something goes wrong.

Here is the breakdown of your deployment steps, now including the **"What"** (the action) and the **"Why"** (the reason it’s necessary).



### (Optional) Prisma Client Singleton
In a serverless environment, database connections can multiply quickly as functions wake up. You should use a singleton pattern for your Prisma client to prevent "too many clients" errors.

**`src/lib/prisma.ts`:**
```typescript
import { PrismaClient } from '@prisma/client';

const globalForPrisma = global as unknown as { prisma: PrismaClient };

export const prisma = globalForPrisma.prisma || new PrismaClient();

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
```

<br>
<br>
<br>
<br>
<br>


# 🛠️ Vercel Deployment Guide (with Rationale)

### Phase 1: The TypeScript Environment
These settings align your code with how Vercel and Node.js actually execute files.

| **What** (Action) | **Why** (Rationale) |
| :--- | :--- |
| **Set `"type": "module"`** in `package.json` | Tells Node.js to use modern **ES Modules** (`import/export`). Without this, Node expects older `require()` syntax and will throw errors on your imports. |
| **Set `moduleResolution: "bundler"`** in `tsconfig.json` | Tells VS Code that an external tool (`tsup`) is handling the file paths. This **silences the red error lines** that demand `.js` extensions in your source code. |

---

### Phase 2: The Build Process (`tsup` & Prisma)
Because Vercel is a serverless environment, your code needs to be prepared specifically for it.

| **What** (Action) | **Why** (Rationale) |
| :--- | :--- |
| **Use `tsup` instead of `tsc`** | **The Issue:** `tsc` is a transpiler; it just turns `.ts` into `.js` but leaves your imports exactly as they are. In ESM mode, Node.js crashes if those imports don't end in `.js`. <br><br> **The Fix:** `tsup` is a **bundler**. It actually "follows" your imports, grabs the code, and resolves all those path issues and missing extensions automatically. It turns a broken, unrunnable mess into a single, clean, optimized `dist/index.js` file. |
| **Run `prisma generate`** in `build` script | The Prisma Client is generated specifically for the environment it's in. Running this on Vercel ensures the **Query Engine** matches Vercel’s Linux operating system. |
| **Export `default app`** in `index.ts` | Vercel doesn't "run" your file like a normal server; it **imports** it. It looks specifically for a default export to know which Express instance should handle the incoming web requests. |




---

### Phase 3: The Vercel Configuration
This file acts as the "instruction manual" for Vercel's infrastructure.

| **What** (Action) | **Why** (Rationale) |
| :--- | :--- |
| **`"use": "@vercel/node"`** | This tells Vercel to use its optimized Node.js runtime. It includes a "bridge" that allows traditional Express apps to work inside a Serverless Function. |
| **Routing `/(.*)` to `dist/index.js`** | By default, Vercel looks for files in an `api/` folder. This wildcard route tells Vercel: "Send **every single request** to my Express app and let Express handle the routing." |

---

### Phase 4: Execution (The CLI)
The final steps to link your local code to the cloud.

| **What** (Action) | **Why** (Rationale) |
| :--- | :--- |
| **`vercel link`** | Connects your local folder to a specific project ID on Vercel so the CLI knows where to send the code. |
| **`vercel env add`** | Adds secrets (like your DB URL) directly to Vercel's encrypted vault. This keeps your credentials out of your code (and off GitHub) for **security**. |
| **`vercel --prod`** | Pushes your code and triggers the **Remote Build**. Vercel builds the project on its own servers to ensure the environment is identical to the final hosting spot. |

---











