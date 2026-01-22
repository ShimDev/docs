# Shim.so Production Deployment Manual

## Phase 1: Cloudflare Backend (The Engine)

We need to provision the actual infrastructure on Cloudflare.

### 1. Authenticate
Run this in your terminal:
```bash
npx wrangler login
```
*This will open a browser window. Click "Allow".*

### 2. Provision Database (D1)
Create the production database:
```bash
npx wrangler d1 create shim-db-prod
```
**ACTION:** Copy the `database_id` output from this command. We will need it.

### 3. Provision Cache (KV)
Create the production KV namespace:
```bash
npx wrangler kv:namespace create shim-kv-prod
```
**ACTION:** Copy the `id` output from this command.

### 4. Update Configuration
Edit `wrangler.toml` with the IDs you just generated:
```toml
[[kv_namespaces]]
binding = "KV"
id = "<PASTE_KV_ID_HERE>"

[[d1_databases]]
binding = "DB"
database_name = "shim-db-prod"
database_id = "<PASTE_D1_ID_HERE>"
```

### 5. Deploy API
Push the code to the edge:
```bash
npx wrangler deploy
```
*This will output your API URL (e.g., `https://shim-api.yourname.workers.dev`). Save this.*

### 6. Initialize Database Schema
Apply the tables to your new production DB:
```bash
npx wrangler d1 execute shim-db-prod --file=./schema.sql
```

---

## Phase 2: Vercel Dashboard (The Console)

Now we deploy the UI.

### 1. Connect Vercel
1. Go to [Vercel.com](https://vercel.com) -> "Add New..." -> "Project"
2. Import `ShimDev/shim-core`
3. **IMPORTANT:** In "Framework Preset", verify it detects **Next.js**.
4. **IMPORTANT:** In "Root Directory", click "Edit" and select `console`.

### 2. Environment Variables
Add these environment variables in the Vercel project settings:

| Variable | Value | Description |
|----------|-------|-------------|
| `NEXTAUTH_URL` | `https://your-domain.com` | Your production dashboard URL |
| `NEXTAUTH_SECRET` | *(Generate a random string)* | `openssl rand -base64 32` |
| `POLAR_CLIENT_ID` | ... | From Polar Dashboard |
| `POLAR_CLIENT_SECRET` | ... | From Polar Dashboard |
| `POLAR_ACCESS_TOKEN` | ... | Personal Access Token from Polar |
| `CLOUDFLARE_ACCOUNT_ID` | ... | From Cloudflare Dashboard (URL) |
| `CLOUDFLARE_D1_DATABASE_ID` | ... | The ID from Phase 1 Step 2 |
| `CLOUDFLARE_API_TOKEN` | ... | Create in CF with "D1 Edit" permissions |

### 3. Deploy
Click "Deploy". Vercel will build and launch.

---

## Phase 3: Domains (DNS)

We need your input here.

**Recommended Setup:**
*   **Console:** `console.shim.so` (or `app.shim.so`) -> Points to **Vercel**
*   **Engine:** `api.shim.so` -> Points to **Cloudflare Worker**

**Action Required:**
1.  Add `console.shim.so` in Vercel Domains settings.
2.  Add `api.shim.so` in Cloudflare Worker Triggers settings.
