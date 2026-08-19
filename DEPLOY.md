# Deployment Runbook — Depression TMS Landing Page

Everything a developer needs to take this page live on karmatms.com.

---

## What you are deploying

One file plus an assets folder. No build step, no framework, no npm install, no
database. Open `index.html` in a browser and it works.

```
index.html          the entire page (HTML + CSS + JS inline, 143 KB / 36 KB gzipped)
assets/             logo, doctor portrait, favicon
deploy/.htaccess    Apache config, copy next to index.html
deploy/submit.php   optional fallback form endpoint
```

Current measured scores (Lighthouse, mobile emulation, served gzipped):
Performance 100 · Accessibility 100 · SEO 100 · FCP 0.9s · LCP 1.0s · CLS 0.

---

## Two things that will break it if skipped

**1. Serve it compressed.** Without gzip or brotli the page ships 143 KB instead of
36 KB and Lighthouse performance drops from 100 to 96, FCP from 0.9s to 1.6s. The
supplied `.htaccess` handles this on Apache. Netlify, Vercel and Cloudflare do it
automatically.

**2. Connect the form.** Out of the box the form is in demo mode: it validates,
shows the success screen, and sends nothing anywhere. Until step 4 is done, every
lead is silently lost.

---

## Step 1 — Add the three brand files

Drop these into `assets/`. Exact lowercase filenames, no spaces.

| File | Used where | Spec |
|---|---|---|
| `karma-tms-logo.png` | Nav bar and footer | White version (both sit on dark purple). `.svg` and `.jpg` also accepted. Renders 42px tall in nav, 46px in footer, up to 230px wide |
| `dr-keerthy-sunder.jpg` | Provider section | 4:5 portrait crop, min 800×1000px, under 300 KB. `.png` / `.jpeg` / `.webp` also accepted |
| `favicon.png` | Browser tab | 32×32 or 512×512 PNG |

The page tries each extension in turn and falls back to plain text if a file is
missing, so nothing breaks visually while you wait for assets.

---

## Step 2 — Pick the URL

**Option A (recommended): a folder on the existing site.**
`https://www.karmatms.com/depression-tms/`

Inherits the domain's existing authority, needs no DNS change, and keeps
everything on one certificate. This is what the page's canonical tag is already
set to.

**Option B: a subdomain on a static host.**
`https://start.karmatms.com` via Netlify or Cloudflare Pages.

Use this if the main host is slow, if you want deploy previews, or if marketing
wants to iterate without touching the production site.

---

## Step 3 — Deploy

### Option A — cPanel / Apache (the existing karmatms.com host)

1. In File Manager, create `public_html/depression-tms/`.
2. Upload `index.html` and the `assets/` folder into it.
3. Upload `deploy/.htaccess` into the same folder (enable "show hidden files" to
   see it afterwards).
4. Create `secure-leads/` **one level above** `public_html/` if you plan to use
   the PHP fallback endpoint in step 4. It must not be web-reachable.
5. Visit `https://www.karmatms.com/depression-tms/` and confirm it loads over
   HTTPS.

Verify compression is actually on:

```bash
curl -sI -H "Accept-Encoding: gzip" https://www.karmatms.com/depression-tms/ | grep -i content-encoding
# expected: content-encoding: gzip
```

If that returns nothing, mod_deflate is off — ask the host to enable it, or the
page loses roughly 0.7s of load time.

### Option B — Netlify or Cloudflare Pages

1. Push this repository to GitHub.
2. Create a new site from that repo. Build command: **none**. Publish
   directory: **/** (repository root).
3. Add the custom domain `start.karmatms.com` in the host's dashboard.
4. In your DNS provider add the CNAME the host gives you:
   `start` → `<your-site>.netlify.app` (or the Pages equivalent), proxied/orange
   cloud on if using Cloudflare.
5. Wait for the certificate to issue (usually under 5 minutes), then load the URL.

Compression and HTTP/2 are on by default on both hosts. Nothing else to configure.

---

## Step 4 — Connect the form (do not skip)

The form posts JSON to whatever URL is in the `data-endpoint` attribute:

```html
<!-- index.html, inside the eligibility modal -->
<form class="qz-form" id="leadForm" novalidate data-endpoint="">
```

Set that attribute and the page does the rest. The payload:

```json
{
  "first_name": "Jordan",
  "phone": "(760) 123-4567",
  "email": "jordan@example.com",
  "location": "Twentynine Palms",
  "screening": [
    { "question": 1, "answer": "Diagnosed with MDD", "flag": "ok" },
    { "question": 2, "answer": "3+ medications",     "flag": "ok" },
    { "question": 3, "answer": "40-64",              "flag": "ok" },
    { "question": 4, "answer": "No safety flags",    "flag": "ok" },
    { "question": 5, "answer": "Twentynine Palms",   "flag": "ok" }
  ],
  "screening_outcome": "You qualify for a TMS consultation.",
  "page": "/depression-tms/",
  "submitted_at": "2026-08-18T12:00:00.000Z"
}
```

`flag` values are `ok`, `review` or `review-call`, so leads can be routed or
prioritised automatically.

### Where to point it

**Best: a CRM webhook covered by a signed BAA.** GoHighLevel, HubSpot,
Salesforce Health Cloud, or whatever the practice already uses for intake. Paste
the inbound webhook URL straight into `data-endpoint`.

**Acceptable interim: `deploy/submit.php`.** Upload it next to `index.html`, set
`CLINIC_EMAIL` and the lead directory at the top of the file, then set
`data-endpoint="submit.php"`. It validates, rate-limits, appends each lead to a
JSONL file outside the web root, and emails a minimal notification that
deliberately excludes the clinical answers.

**Not acceptable: plain email with the screening answers in the body.** This form
collects protected health information — name, phone, email and a set of answers
about a person's psychiatric history and medication trials. Ordinary SMTP is not
a HIPAA-compliant transport. Get the destination signed off by whoever owns HIPAA
compliance before launch.

### Test it

Complete the check with a real phone number and confirm the lead arrives at the
destination, with the screening array intact. On failure the form shows an alert
and re-enables the button, so a broken endpoint is visible rather than silent.

---

## Step 5 — Update the page constants

| What | Where in `index.html` | Change to |
|---|---|---|
| Canonical URL | `<link rel="canonical">` line 9 | The live URL, if not `/depression-tms/` |
| Open Graph URL | `og:` meta tags in `<head>` | Same URL, plus a share image if you have one |
| Scarcity number | `<span id="slots">4</span>` in the announcement bar | The real number of free consultations left this week |
| Testimonial videos | `data-yt` on `.vfeature` and `.vid-card` | Only if the video IDs change |

---

## Step 6 — Analytics

Every CTA already carries a `data-cta` attribute and the page fires structured
events. Find `function track(` near the top of the script block and replace its
body with your own call:

```js
function track(name, props){
  gtag('event', name, props || {});          // GA4
  // fbq('trackCustom', name, props || {});  // Meta
}
```

By default it pushes to `window.dataLayer`, so if you install Google Tag Manager
the events flow through with no code change at all.

Events fired:

| Event | Fires when | Useful for |
|---|---|---|
| `cta_click` | Any CTA, with its location | Which CTA earns the opens |
| `quiz_opened` | Modal opens | Top of the qualifier funnel |
| `quiz_answer` | Each question answered | Exactly which question loses people |
| `quiz_completed` | Verdict shown, with the outcome | Qualified vs filtered split |
| `lead_submitted` | Form submitted successfully | Conversions |
| `lead_form_error` | Validation failed | Form friction |
| `video_play` | Any video clicked | Video engagement |
| `faq_open` | Accordion opened | Unanswered objections |
| `scroll_depth` | 25 / 50 / 75 / 100% | Page engagement |

Set the conversion goal on `lead_submitted`, and watch `quiz_answer` — it tells
you which of the five questions is costing the most drop-off.

---

## Step 7 — SEO housekeeping

1. Add the URL to the sitemap and submit it in Search Console.
2. Link to it from the main site: the depression page, the conditions page and
   the main nav are the natural spots.
3. The page ships with `MedicalBusiness` and `FAQPage` structured data already
   filled in — validate it once at `search.google.com/test/rich-results`.
4. If the page will run paid traffic only, add `<meta name="robots"
   content="noindex">` to keep it out of organic results and avoid competing with
   the existing depression page.

---

## Step 8 — Compliance sign-off before launch

These are not developer decisions, but they must be closed before the page is
public.

- **The scarcity line** in the announcement bar ("Only 4 complimentary
  consultations left this week") must be true and kept current. A permanent false
  scarcity claim is deceptive advertising and carries regulatory risk in
  healthcare.
- **The social proof toasts** (`PROOF_EVENTS` array in the script) are placeholder
  data. Wire them to a real feed of recent requests or delete the block entirely.
- **Clinical figures** (85% improvement, ~50% remission, 60-70% response, 12+
  months) need a citable source on file.
- **Video testimonials** need patient authorization on file for public use.
- **"FDA-cleared"** is the correct term and is used throughout. The main site's
  footer currently says "FDA-Approved" — worth correcting there too.
- **The form collects PHI.** Confirm the destination is BAA-covered.

---

## Step 9 — Launch QA

- [ ] Loads over HTTPS with a valid certificate
- [ ] `content-encoding: gzip` present in response headers
- [ ] Logo, doctor portrait and favicon all render
- [ ] Hero video plays; all five testimonial Shorts play
- [ ] Eligibility modal opens from every CTA (nav, hero, benefits bar, prompt
      band, journey, provider, testimonials, final CTA, sticky bar, announcement)
- [ ] All five outcomes reachable: qualified, likely, clinician review, under 18,
      not depression
- [ ] Under-18 and not-depression outcomes show **no** form
- [ ] Form rejects bad input and submits clean input
- [ ] Test lead arrives in the CRM with the screening array intact
- [ ] Modal closes on Esc, backdrop click and the × button
- [ ] Tested on real iOS Safari and Android Chrome, not just desktop devtools
- [ ] No horizontal scroll at 320px, 390px and 768px
- [ ] Analytics events visible in GA4 DebugView / GTM preview
- [ ] Lighthouse mobile run against the live URL scores 90+ on performance

---

## Rollback

Static page, so rollback is trivial: keep the previous `index.html` alongside as
`index.html.bak` before overwriting, or revert the commit and redeploy. There is
no database, no session state and no server dependency to unwind.

---

## Notes for whoever maintains it

- All colours are CSS custom properties at the top of the `<style>` block. Six
  values re-skin the whole page.
- The quiz questions, options and routing live in the `.qz-step` markup plus the
  `WHY` map and `finish()` function in the script. Adding a question means adding
  a `.qz-step` block and its entries in `WHY`.
- The page is deliberately one file. If it ever needs splitting, extract the CSS
  and JS into separate files but keep the critical CSS inline, or the 0.9s FCP
  will regress.
