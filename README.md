# cinesmith.io — waitlist site

The public waitlist for CineSmith. Single static page hosted on **GitHub
Pages**. The page's CTA opens a hosted **Google Form** in a new tab — Google
captures the lead and writes it to a bound Sheet. No backend, no secrets, no
infrastructure to maintain.

```
[GitHub Pages: cinesmith.io]
        │ click CTA
        ▼
[Google Form (forms.gle/…)]
        │ submit
        ▼
[Google Sheet — bound to the form]
```

All emails come from `cinesmith@cineartai.com` (no MX setup needed on
`cinesmith.io` itself).

---

## Repo layout

```
cinesmith.io/
├── index.html              # The waitlist page
├── assets/                 # Hero, favicon, OG cover
├── CNAME                   # GitHub Pages → cinesmith.io
├── robots.txt
├── sitemap.xml
└── .github/workflows/
    └── pages.yml           # Auto-deploy on push to main
```

<!-- Earlier Cloudflare Worker + Apps Script stack, if revived, is restorable
     from a local-only archive branch — see "Custom backend" under Rollback. -->

---

## First-time deploy

### 1. Create the Google Form

In a Google account you control:

1. https://forms.google.com → blank form
2. Title: **CineSmith — Closed Beta Waitlist**
3. Description: short blurb about the beta
4. Add fields (suggested):
   - **Email** — Short answer, *required*, response validation = "Email"
   - **First name** — Short answer, *required*
   - **Production company** — Short answer
   - **Role** — Dropdown: Producer / Director / DOP / Editor / VFX / Founder / Other
   - **What are you working on?** — Paragraph, optional
5. **Settings tab → Responses:**
   - "Collect email addresses" → **Responder input** (do **not** pick "Verified" — it forces Google sign-in and kills conversion for non-Google users)
   - "Limit to 1 response" → **OFF** (also requires sign-in)
   - "Allow response editing" → **OFF**
6. **Settings tab → Presentation:**
   - Confirmation message: *"You're on the list. We'll be in touch from cinesmith@cineartai.com when the next cohort opens."*
7. **Responses tab → 3-dot menu → "Get email notifications for new responses"** → ON
8. **Responses tab → green Sheets icon** → creates `cinesmith_waitlist (Responses)` Sheet automatically. Keep the Sheet private; share read-only with the team if needed.
9. **Send button → link icon → toggle "Shorten URL"** → copy `https://forms.gle/xxxxx`
10. The current production URL is wired into `index.html` on the `.waitlist-cta` anchor. Update that one href if the form is ever replaced.

### 2. Create the GitHub repo

In GitHub Desktop or CLI: name `cinesmith.io`, public, push to `main`.

### 3. Add `cinesmith.io` to Cloudflare

- Cloudflare Dashboard → **Add a site** → `cinesmith.io` → Free plan
- Update nameservers at your registrar to the two Cloudflare ones
- Wait for activation (usually < 1 hour)

### 4. Configure DNS in Cloudflare

| Type   | Name | Content                        | Proxy            |
|--------|------|--------------------------------|------------------|
| A      | @    | `185.199.108.153`              | DNS only         |
| A      | @    | `185.199.109.153`              | DNS only         |
| A      | @    | `185.199.110.153`              | DNS only         |
| A      | @    | `185.199.111.153`              | DNS only         |
| CNAME  | www  | `<your-github-user>.github.io` | DNS only         |

> GitHub Pages records must be **DNS only** — orange-cloud proxying breaks
> the SSL handshake.

### 5. Enable GitHub Pages

In the repo: **Settings → Pages**:
- Source: **GitHub Actions**
- Custom domain: `cinesmith.io`
- Enforce HTTPS (after DNS propagates and the SSL cert issues — usually < 1 hour)

The first push to `main` triggers `.github/workflows/pages.yml` and the site
goes live at `https://cinesmith.io`.

### 6. Test end-to-end

- Open `https://cinesmith.io`
- Click **Request early access** → opens the Google Form in a new tab
- Submit a test response
- Confirm a row appears in the bound Sheet
- Confirm the email notification arrives

---

## Day-to-day

### Editing copy / design

Edit `index.html` → push to `main` → GitHub Actions deploys in ~1 min.

### Editing the form fields

Edit the Google Form directly. The Sheet picks up new columns automatically.
The CTA URL on `index.html` does not change.

### Sync into Company AI

A separate cron job in the Company AI project polls the bound Sheet and:
- writes new rows to `data/cinesmith_waitlist.db`
- fires a confirmation email from `cinesmith@cineartai.com`

That sync lives in the main `The Company` repo (not in this one) so the public
site stays a static, self-contained artefact.

---

## Security model

There is no secret material in this repo. The CTA is a public link to a public
Google Form — anyone with the URL can submit, which is the intended behaviour
for a waitlist. Things to know:

- **Spam protection:** Google's built-in anti-abuse handles bot floods. If
  you see junk responses, delete them in the Sheet — no infrastructure
  damage possible.
- **Sheet privacy:** the bound Sheet is owned by the Google account that
  created the Form. Share with team members as **Viewer**, never **Editor**
  (Editors can install rogue Apps Script).
- **URL discoverability:** `forms.gle/...` URLs are unguessable. Treat the
  shortened link as semi-public — it's safe to print on marketing material.
- **Brand spoofing:** anyone can create a Google Form pretending to be
  CineSmith. We mitigate by linking to the canonical form from the
  `cinesmith.io` page only and including the CineSmith logo at the top of
  the form so users can verify they landed on the right place.

---

## Rollback

- **Bad deploy?** Revert the commit on `main` — Pages redeploys automatically.
- **Form broken / spammed?** Settings → Form is no longer accepting responses
  → toggle off. Re-enable when ready.
- **Custom backend (Worker + Apps Script + HMAC) needed later?** The
  implementation is preserved on local-only refs — never pushed to the
  public remote. From the owner's workstation, restore with:
  `git checkout archive/worker-stack-hmac -- worker apps-script`
  (HMAC-hardened version) or `archive/worker-stack-original` for the
  pre-HMAC scaffold. Both tags live on the local clone only — if it's been
  wiped, the code is gone for everyone except whoever still has a clone.

---

## Domain + email

- **Domain:** cinesmith.io (registered, on Cloudflare)
- **Inbound email:** `cinesmith@cineartai.com` — there is no MX on `cinesmith.io`.
  All transactional + reply email runs through `cineartai.com`.
- **From-address everywhere:** `cinesmith@cineartai.com`.
