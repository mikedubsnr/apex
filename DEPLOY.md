# ApexVisa UK — Supabase + Resend Deployment Guide

This package wires up your visa risk checker so that:

1. **On-screen result is instant** — the deterministic engine runs in the browser, no network call.
2. **Email contains a richer AI-written report** — Anthropic generates a personalised report inside your Supabase Edge Function.
3. **Email is sent via Resend** from `contact@apexvisa.uk`.
4. **Rate-limited** to one email per address per 24 hours.

---

## What's in this package

```
deploy/
├── index.html                                 ← Updated frontend
├── DEPLOY.md                                  ← This file
└── supabase/
    ├── functions/send-sample/index.ts         ← Edge Function (Deno/TS)
    └── migrations/20260428_email_log.sql      ← Rate-limit table
```

---

## One-time setup

### 1. Install the Supabase CLI

```bash
# macOS
brew install supabase/tap/supabase

# Or via npm globally
npm install -g supabase
```

### 2. Link the CLI to your project

```bash
cd /path/to/this/deploy
supabase login
supabase link --project-ref ulliivthsjtktywdmdqj
```

### 3. Push the database migration

This creates the `email_log` table used for rate limiting.

```bash
supabase db push
```

If `db push` complains about existing migrations, you can run the SQL directly:

```bash
psql "$(supabase db remote-url)" -f supabase/migrations/20260428_email_log.sql
```

Or paste the contents of `supabase/migrations/20260428_email_log.sql` into the Supabase dashboard → **SQL Editor** → **Run**.

### 4. Set the Edge Function secrets

Replace each placeholder with your real key. **Never commit these or paste them into chat.**

```bash
supabase secrets set RESEND_API_KEY=re_YOUR_RESEND_KEY \
  --project-ref ulliivthsjtktywdmdqj

supabase secrets set ANTHROPIC_API_KEY=sk-ant-YOUR_ANTHROPIC_KEY \
  --project-ref ulliivthsjtktywdmdqj

supabase secrets set FROM_EMAIL=contact@apexvisa.uk \
  --project-ref ulliivthsjtktywdmdqj

supabase secrets set ALLOWED_ORIGIN=https://apexvisa.uk \
  --project-ref ulliivthsjtktywdmdqj
```

Verify:

```bash
supabase secrets list --project-ref ulliivthsjtktywdmdqj
```

You should see all four names listed (values are hidden).

### 5. Deploy the Edge Function

```bash
supabase functions deploy send-sample --project-ref ulliivthsjtktywdmdqj
```

The function URL will be:

```
https://ulliivthsjtktywdmdqj.supabase.co/functions/v1/send-sample
```

This is already hard-coded in `index.html` — no further change needed.

---

## Verify it works

### Test the Edge Function directly with curl

```bash
curl -i -X POST https://ulliivthsjtktywdmdqj.supabase.co/functions/v1/send-sample \
  -H "Content-Type: application/json" \
  -H "apikey: sb_publishable_EcRGa2uu4STu2clld-jFqQ_bZVQPM02" \
  -H "Authorization: Bearer sb_publishable_EcRGa2uu4STu2clld-jFqQ_bZVQPM02" \
  -H "Origin: https://apexvisa.uk" \
  -d '{
    "email": "your-test-inbox@example.com",
    "visaType": "Spouse / Partner Visa",
    "riskLevel": "MEDIUM",
    "factors": [
      {"label":"Borderline Financial Evidence","text":"Income is close to threshold but not clearly above."},
      {"label":"Limited Travel History","text":"A thin travel record is a moderate risk factor."},
      {"label":"Cover Letter Recommended","text":"A cover letter would address several risk areas."}
    ]
  }'
```

You should get:

```
HTTP/1.1 200 OK
{"ok":true}
```

…and an email arrives within seconds at the test address.

### Test via the website

1. Upload `index.html` to your hosting at `https://apexvisa.uk/`
2. Visit `https://apexvisa.uk/`
3. Click **Free Risk Check**
4. Answer all 6 questions
5. Enter your test email
6. Confirm: on-screen result appears instantly + email arrives within seconds

---

## Hosting `index.html` on apexvisa.uk

Any static host works — the file is fully self-contained.

**Cloudflare Pages** (recommended, free, fast):
```bash
# Install wrangler if needed
npm install -g wrangler
wrangler pages deploy . --project-name=apexvisa
```

**Vercel:**
```bash
npx vercel --prod
```

**Netlify:**
Drag `index.html` into the dashboard upload area.

**Any other static host:** just upload `index.html` to the root.

Make sure your DNS for `apexvisa.uk` points at the host.

---

## Monitoring

### Edge Function logs

```bash
supabase functions logs send-sample --project-ref ulliivthsjtktywdmdqj --tail
```

Or in the dashboard: **Edge Functions → send-sample → Logs**.

### Email delivery (Resend dashboard)

`https://resend.com/emails` — shows every email sent, delivery status, opens, bounces.

### Rate-limit data

In Supabase **Table Editor → email_log**, or via SQL:

```sql
select email, count(*) as sends, max(sent_at) as last
from public.email_log
group by email
order by sends desc;
```

### Daily summary query

```sql
select
  date_trunc('day', sent_at) as day,
  risk_level,
  count(*) as sends
from public.email_log
where sent_at > now() - interval '30 days'
group by day, risk_level
order by day desc;
```

---

## Security model

| Asset | Where it lives | Exposed to browser? |
|---|---|---|
| Supabase URL | `index.html` | Yes (safe) |
| Supabase anon key | `index.html` | Yes (safe by design) |
| Resend API key | Supabase secret | **No** |
| Anthropic API key | Supabase secret | **No** |
| Service role key | Auto-injected to Edge Function | **No** — never put in browser |
| User email addresses | Postgres (RLS-locked) | No |

The Edge Function:
- ✅ Locks CORS to `https://apexvisa.uk` only — other domains can't call it
- ✅ Validates email format server-side
- ✅ Rate-limits 1 email per address per 24h via Postgres
- ✅ Falls back to a deterministic report template if Anthropic is down
- ✅ Logs failures without leaking secrets

---

## If something goes wrong

**Email isn't sending:**
1. Check `supabase functions logs send-sample --tail`
2. Check Resend dashboard for the email — if it appears as "delivered" it's there (check spam)
3. Verify `FROM_EMAIL` matches a verified Resend domain — `contact@apexvisa.uk` requires `apexvisa.uk` to show "verified" in Resend

**CORS errors in browser console:**
- Make sure `ALLOWED_ORIGIN` secret matches your actual hosted URL exactly (including `https://`, no trailing slash)
- For local testing temporarily set `ALLOWED_ORIGIN=*` then revert before going live

**"Invalid email" returns 400:**
- The function rejects malformed addresses — check the payload

**Anthropic call fails:**
- Function falls back to a templated report automatically
- Check `ANTHROPIC_API_KEY` is set correctly: `supabase secrets list`
- Check your Anthropic credit balance

**429 / rate-limited:**
- Same email tried within 24h — by design
- Override during testing by deleting the row: `delete from public.email_log where email = 'test@example.com';`
