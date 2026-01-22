# Dashboard Backend Integration - Setup Guide

## Status: Ready for Configuration

All code is complete. Follow these steps to connect the dashboard to production data.

---

## Phase 1: Environment Configuration

### 1.1 Create `.env.local` file

```bash
cd dashboard
cp .env.local.example .env.local
```

### 1.2 Configure Polar OAuth

1. Go to https://polar.sh/dashboard → Settings → OAuth Applications
2. Click "Create OAuth Application"
3. Fill in:
   - **Name:** Shim Dashboard
   - **Redirect URI:** `http://localhost:3000/api/auth/callback/polar`
   - **Scopes:** `openid profile email`
4. Copy the **Client ID** and **Client Secret**
5. Update `.env.local`:
   ```
   POLAR_CLIENT_ID=your_client_id_here
   POLAR_CLIENT_SECRET=your_client_secret_here
   ```

### 1.3 Generate NextAuth Secret

```bash
openssl rand -base64 32
```

Add to `.env.local`:
```
NEXTAUTH_SECRET=generated_secret_here
```

### 1.4 Configure Cloudflare Access

#### Get Account ID:
```bash
wrangler whoami
```

#### Get D1 Database ID:
```bash
wrangler d1 list
```

#### Get KV Namespace ID:
```bash
wrangler kv:namespace list
```

#### Create API Token:
1. Go to https://dash.cloudflare.com/profile/api-tokens
2. Click "Create Token"
3. Use "Edit Cloudflare Workers" template
4. Add permissions:
   - **D1:** Read
   - **Workers KV Storage:** Read
5. Copy the token

#### Update `.env.local`:
```
CLOUDFLARE_ACCOUNT_ID=your_account_id
CLOUDFLARE_D1_DATABASE_ID=your_d1_database_id
CLOUDFLARE_KV_NAMESPACE_ID=your_kv_namespace_id
CLOUDFLARE_API_TOKEN=your_api_token
```

---

## Phase 2: Database Setup

### 2.1 Ensure D1 Schema is Applied

```bash
# For development
wrangler d1 execute shim-db-dev --local --file=../db/schema.sql

# For production
wrangler d1 execute shim-db-production --remote --file=../db/schema.sql
```

### 2.2 Verify Tables Exist

```bash
wrangler d1 execute shim-db-dev --local --command="SELECT name FROM sqlite_master WHERE type='table';"
```

Expected output:
- `api_keys`
- `repair_logs`
- `daily_stats`
- `usage_events`

---

## Phase 3: Test the Integration

### 3.1 Start the Dashboard

```bash
cd dashboard
npm run dev
```

### 3.2 Test Authentication Flow

1. Navigate to http://localhost:3000
2. You should be redirected to `/auth/signin`
3. Click "Sign in with Polar"
4. Authorize the application
5. You should be redirected back to the dashboard

### 3.3 Verify Data Flow

**If you have test data in D1:**
- Dashboard should show real stats
- Usage meter should reflect KV counters
- Logs page should show recent repairs

**If no data exists yet:**
- All values will show `0` or `—`
- This is expected for a fresh installation

---

## Phase 4: Create Test Data (Optional)

### 4.1 Insert Test API Key

```bash
wrangler d1 execute shim-db-dev --local --command="
INSERT INTO api_keys (id, key_hash, polar_customer_id, tier, monthly_limit, created_at, is_active)
VALUES ('test_key_1', 'test_hash_123', 'cust_test', 'pro', 100000, $(date +%s)000, 1);
"
```

### 4.2 Insert Test Repair Logs

```bash
wrangler d1 execute shim-db-dev --local --command="
INSERT INTO repair_logs (
  id, api_key_id, timestamp, input_size_bytes, had_schema, mode,
  success, confidence, was_repaired, output_valid_json,
  syntax_repair_types, schema_repair_types,
  warning_codes, warning_severities, error_codes, error_severities,
  error_recoverable_flags, affected_fields, latency_ms
) VALUES (
  'log_1', 'test_key_1', $(date +%s)000, 512, 1, 'strict',
  1, 'high', 1, 1,
  '[\"TRAILING_COMMA\"]', '[\"TYPE_COERCION\"]',
  '[]', '[]', '[]', '[]', '[]', '[\"user.age\"]', 45
);
"
```

### 4.3 Set KV Usage Counter

```bash
wrangler kv:key put --binding=KV "usage:test_key_1:$(date +%Y-%m)" "1247"
```

---

## Phase 5: Production Deployment

### 5.1 Update Environment Variables for Production

Create `.env.production.local` with production values:
```
POLAR_CLIENT_ID=prod_client_id
POLAR_CLIENT_SECRET=prod_client_secret
POLAR_REDIRECT_URI=https://dashboard.shim.so/api/auth/callback/polar
NEXTAUTH_URL=https://dashboard.shim.so
NEXTAUTH_SECRET=different_secret_for_production
CLOUDFLARE_ACCOUNT_ID=same_as_dev
CLOUDFLARE_D1_DATABASE_ID=production_database_id
CLOUDFLARE_KV_NAMESPACE_ID=production_kv_id
CLOUDFLARE_API_TOKEN=production_api_token
```

### 5.2 Deploy to Vercel

```bash
vercel --prod
```

During deployment, add all environment variables from `.env.production.local` to Vercel dashboard.

---

## Troubleshooting

### "Unauthorized" Error
- Check that Polar OAuth is configured correctly
- Verify redirect URI matches exactly
- Ensure user has an API key in D1

### "Failed to fetch stats" Error
- Verify Cloudflare API token has correct permissions
- Check that D1 database ID is correct
- Ensure tables exist in D1

### No Data Showing
- Verify KV namespace ID is correct
- Check that repair logs exist for the user's `api_key_id`
- Ensure timestamps are in milliseconds (not seconds)

### Session Not Persisting
- Verify `NEXTAUTH_SECRET` is set
- Check browser cookies are enabled
- Ensure `NEXTAUTH_URL` matches your domain

---

## Next Steps

1. **API Key Display Page:** Create `/keys` page to show masked API key
2. **Logs Page:** Create `/logs` page with pagination and filtering
3. **Real-time Updates:** Add WebSocket or polling for live stats
4. **Charts:** Integrate Recharts for usage trends
5. **Alerts:** Add email notifications for usage limits

---

## Architecture Summary

```
User Browser
    ↓
Next.js Dashboard (Vercel)
    ↓ (NextAuth Session)
    ↓
Cloudflare REST API
    ↓
D1 Database + KV Storage
```

**Key Security Features:**
- ✅ OAuth authentication via Polar
- ✅ Session-based authorization
- ✅ Row-level security (queries filtered by `api_key_id`)
- ✅ No API keys exposed to client
- ✅ Direct D1/KV access (no proxy through Worker)

**Performance:**
- ✅ KV for real-time usage counters
- ✅ D1 for historical analytics
- ✅ SWR for client-side caching
- ✅ Server-side session caching
