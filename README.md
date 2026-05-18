# Fan Rescue — Internal Proposal Portal

A single-page internal admin tool listing every live cleaning proposal and installation quote.

**Live at:** `https://portal.fanrescue.co.uk` (Cloudflare Access protected — staff only)

---

## How it works

The portal is a single static HTML page. On load, browser-side JavaScript:

1. Calls GitHub's REST API for the contents of both proposal repos:
   - `github.com/FanRescue/fan-rescue-proposals` (cleaning)
   - `github.com/FanRescue/fanrescue-quotes` (installation)
2. Filters for HTML files matching the `{slug}-FR-{NNNN}.html` pattern
3. Derives client name + reference from the filename
4. Renders into two tabs, sorted by reference number descending

No database, no backend, no manual indexing. Drop a new HTML file into either repo, push, and it appears on the portal within seconds (subject to GitHub's API cache, usually under a minute).

---

## Filename convention

Files must match this pattern to appear on the portal:

```
{client-slug}-FR-{NNNN}.html
{client-slug}-FR-{YYYY}-{NNN}.html  (legacy installation refs)
```

Examples:
- ✅ `burgrill-clapton-FR-2035.html` → "Burgrill Clapton — FR-2035"
- ✅ `joy-king-lau-restaurant-FR-2036.html` → "Joy King Lau Restaurant — FR-2036"
- ✅ `battersea-bloom-FR-2026-030.html` → "Battersea Bloom — FR-2026-030" (legacy ref)
- ❌ `quote_for_pub.html` → won't appear (doesn't match pattern)
- ❌ `FR-2041-piccadilly.html` → won't appear (ref before slug)

If a file *doesn't* match the pattern, it's silently skipped — useful for `index.html`, `_template/`, etc.

---

## Cloudflare Access setup (REQUIRED — do this before going live)

The portal contains commercially sensitive information (client names, full quote URLs). It **must** be protected. Cloudflare Access provides email one-time-password (OTP) login that's free and reliable.

### Step-by-step

1. **Cloudflare Dashboard** → **Zero Trust** (in the left sidebar — separate section from the main Pages dashboard)
2. **Access → Applications → Add an application → Self-hosted**
3. **Application name:** `Fan Rescue Portal`
4. **Session duration:** `24 hours` (or whatever feels right)
5. **Application domain:**
   - Subdomain: `portal`
   - Domain: `fanrescue.co.uk`
   - Path: leave blank
6. Click **Next**
7. **Policy name:** `Fan Rescue Staff`
8. **Action:** `Allow`
9. **Configure rules → Selector:** `Emails`
10. **Value:** add each address one at a time:
    - `office@fanrescue.co.uk`
    - `huzaifa@fanrescue.co.uk`
    - `irfan.nakip@fanrescue.co.uk`
    - `anas@fanrescue.co.uk`
11. **Next → Authentication: One-time PIN** (this is the default email-OTP method)
12. **Save**

That's it. From now on, anyone visiting `portal.fanrescue.co.uk` will be intercepted: they enter their email, Cloudflare emails them a 6-digit PIN, they enter it, they're in for 24 hours.

### Adding or removing staff later

Zero Trust → Access → Applications → Fan Rescue Portal → Policies → edit the policy and add/remove emails.

### What clients see

Nothing — clients aren't given the portal URL. They continue to use direct quote links emailed to them, which remain publicly accessible by URL (no Access protection on `proposals.*` or `quotes.*`).

---

## Sister repos

| Subdomain | Repo | Purpose | Auth |
|---|---|---|---|
| `assets.fanrescue.co.uk` | `fanrescue-assets` | Logos, brand tokens, company info | Public |
| `proposals.fanrescue.co.uk` | `fan-rescue-proposals` | Cleaning proposals | Public (URL-obscured) |
| `quotes.fanrescue.co.uk` | `fanrescue-quotes` | Installation quotes | Public (URL-obscured) |
| `portal.fanrescue.co.uk` | **`fanrescue-portal`** (this repo) | Internal index of all proposals/quotes | **Cloudflare Access** |

---

## Deployment

Standard pattern — same as the other repos.

1. Push to a new GitHub repo: `fanrescue-portal`
2. Cloudflare → Workers & Pages → Create project → connect repo → leave build settings blank → deploy
3. Custom domains → add `portal.fanrescue.co.uk` (DNS CNAME already exists per the May 2026 setup)
4. **THEN do the Cloudflare Access setup above** before sharing the URL with anyone

---

## Known limits

- **GitHub API rate limit:** 60 unauthenticated requests/hour, per IP. The portal uses 1 contents call + 1 commits call per file, per repo. With ~10 files in each repo that's ~22 calls per portal load across both. Each staff member on their own IP can load the portal 2-3 times per hour before hitting the limit. In normal internal use this is fine; if it becomes a problem we'd add a GitHub Action that pre-builds an `index.json` in this repo on every push to the others, cutting the portal to a single API call.
- **Sort order:** by GitHub commit date descending — i.e. when the file was last pushed. Works correctly across both `FR-XXXX` and legacy `FR-YYYY-NNN` ref formats.
- **Filename = source of truth:** if you rename a quote's client *without* renaming the file, the portal still shows the filename's slug. Easy to enforce — keep filename and quote in sync.

---

## What it does NOT do (yet)

- Show open/accept tracking data from the Cloudflare Worker beacon — the beacon writes events to a separate KV/database; the portal currently doesn't read from it. If useful, a future enhancement.
- Edit or delete quotes — read-only by design. Edits happen in GitHub.
- Notify on new quote — push to the repo and refresh the portal to see it.

---

_Maintained by Ben — Fan Rescue Ltd._
