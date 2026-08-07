# Leamington Spray Foam — SitePanda to Cloudflare Migration Runbook

Everything that could be done without your accounts is done. This covers the rest,
in the order it has to happen.

**Do not cancel SitePanda until step 6 is verified.**

---

## Where things stand

| Item | Status |
|---|---|
| Site built from the Grimsby generator, Chatham epoxy layout | Done — 16 pages in `site/` |
| All 10 indexed URLs preserved | Done — verified, zero redirects needed |
| Two new pages added (commercial, greenhouse) | Done |
| Copy written fresh and Leamington-specific | Done — 9,900 words, 2.39% prose overlap with the next-closest site |
| All 7 sponsor outbound links carried over | Done |
| Links to your own city sites removed | Done — verified zero |
| Photos reassigned so no slot matches Windsor, Grimsby or Chatham | Done |
| New brand: tomato red on lake teal | Done — 28 contrast pairs computed, all pass |
| Logo, favicon, app icons | Done — generated from `Logo.png` |
| Contact form wired to a live endpoint | **Not done — step 1 below** |
| Deployed | **Not done — steps 3 to 5** |
| DNS moved | **Not done — step 6** |

---

## Step 1 — Google Sheet and Apps Script

Do this signed in as the Google account that should **send** the notification
emails. Apps Script sends as whoever authorises it, regardless of what
`NOTIFY_EMAIL` says.

1. Go to <https://sheets.google.com> and create a **blank spreadsheet**. Name it
   `Leamington Spray Foam Leads`.
   - It must be a native Google Sheet. An uploaded `.xlsx` has no Extensions
     menu and Apps Script cannot read it.
2. **Extensions → Apps Script.**
3. Delete whatever is in `Code.gs`. Open `google-apps-script.gs` from this
   folder, copy all of it, paste it in.
4. `NOTIFY_EMAIL` is already set to `tiffindevelopments@gmail.com`. Change it
   only if you want a different inbox.
5. **Deploy → New deployment.** Gear icon → **Web app**.
   - Description: `Leamington leads`
   - Execute as: **Me**
   - Who has access: **Anyone**
   - Deploy, then authorise when prompted. Google warns about an unverified
     app — expected for your own script. Advanced → Go to project.
6. Copy the **Web app URL**. It ends in `/exec`.

Send me that URL and I will paste it in and rebuild. Or do it yourself: open
`site\script.js`, find `SHEET_ENDPOINT`, replace `'YOUR-APPS-SCRIPT-EXEC-URL'`
with the `/exec` URL, save, then **rerun `python build.py`**. The rebuild is not
optional — the cache fingerprint has to change or nobody's browser picks up the
new file.

> Any later edit to the `.gs` file needs **Deploy → Manage deployments → edit →
> Version: New version → Deploy.** Saving alone leaves the live site on old code.

---

## Step 2 — Look at the site before it goes anywhere

In File Explorer, open:

```
C:\Users\Lenovo\Documents\Tiffin Developments Lead Generation\Spray Foam Content\Leamington Spray Foam Insulation\site\index.html
```

Double-click it. It opens in your browser from disk — no server needed.

Click through the nav to every page. What I could not check is how it *looks*,
so this is the part I need your eyes on. Batch the feedback into one list rather
than one item at a time — each round is a rebuild.

Two things worth a specific look:

- **The colours.** This build is deep lake teal with a tomato red accent, not
  the graphite and amber of chathamepoxyflooring.com. The layout and structure
  are copied from Chatham; the palette is deliberately different because
  Windsor, Grimsby and Chatham spray foam are all in adjacent markets and
  should not share a look. Say the word if you want Chatham's exact palette.
- **The commercial and greenhouse page photos.** The shared pool has no real
  warehouse, cold storage or greenhouse image left that isn't already on the
  Chatham spray foam site. Those two pages are using the best available
  substitutes. See "Open items" at the bottom.

---

## Step 3 — Git repository

In PowerShell, one line at a time:

```powershell
cd "C:\Users\Lenovo\Documents\Tiffin Developments Lead Generation\Spray Foam Content\Leamington Spray Foam Insulation"
git init
git add .
git commit -m "Leamington Spray Foam Insulation - migrated from SitePanda"
git branch -M main
```

Then create an empty repo at <https://github.com/new> named
`leamington-spray-foam`. Set it to **Private**. **No README, no .gitignore, no
licence** — it must be empty.

Private matters: `build.py` and `google-apps-script.gs` both carry
`tiffindevelopments@gmail.com`. Cloudflare connects to private repos with no
extra setup.

```powershell
git remote add origin https://github.com/dallastiffin/leamington-spray-foam.git
git push -u origin main
```

You should see it count objects and finish with `main -> main`.

> If Cloudflare later fails at "Cloning repository", the repo is empty — the
> push did not work. Check for a `.git` folder before debugging anything else.

---

## Step 4 — Cloudflare Worker

1. Cloudflare dashboard → **Compute (Workers)** → **Create** → **Import a
   repository**.
2. Pick `leamington-spray-foam`.
3. Settings:
   - **Worker name:** `leamington-spray-foam` — this must match `name` in
     `wrangler.toml` exactly or the deploy fails.
   - **Build command:** leave empty.
   - **Deploy command:** `npx wrangler deploy`
4. Deploy. You get a `*.workers.dev` URL.

---

## Step 5 — Test on the workers.dev URL, with the real site still live

This is the whole point of doing it in this order. On the temporary URL, check:

- Home page loads at the root, not a 404
- All eight service pages, plus Services, About, Contact and FAQ
- `/about-us` and `/contact-us` work — those are the migrated slugs
- `/crawl-spaces-insulation` works — note it is **spaces**, plural
- `/insulation-removal-service` works — note **service**, singular, on the end
- Submit the contact form. Confirm a row lands in the Sheet **and** an email
  arrives.
- Open it on your phone.

Do not proceed until the form has actually delivered a test lead.

---

## Step 6 — Domain and DNS

I could not query DNS from the sandbox, so unlike the Grimsby runbook this
section tells you what to look at rather than what you will find.

### Check the zone first

Cloudflare dashboard → is `leamingtonsprayfoaminsulation.com` already listed as
a zone?

- **If yes** (which is what Grimsby turned out to be): nameservers are already
  Cloudflare's, traffic already proxies through Cloudflare to the SitePanda
  origin, and only the origin changes. The cutover is close to instant and
  reversible.
- **If no**: you will need to add the site to Cloudflare and change the
  nameservers at Namecheap first, then wait for propagation before continuing.

### Records that must survive — do not "clear all records"

Before you delete anything, screenshot the DNS records page. On Grimsby the
rows that had to survive were:

```
MX   10 eforward1 / eforward2 / eforward3.registrar-servers.com
MX   15 eforward4.registrar-servers.com
MX   20 eforward5.registrar-servers.com
TXT  v=spf1 include:spf.efwd.registrar-servers.com ~all
TXT  google-site-verification=...
```

MX and SPF are Namecheap email forwarding — deleting them kills email to the
domain. The `google-site-verification` row is the Search Console verification
for this property; delete it and you lose the account you need for submitting
the new sitemap.

### The error you will hit, and why

Adding a custom domain from the Worker screen returns:

> Hostname 'leamingtonsprayfoaminsulation.com' already has externally managed
> DNS records (A, CNAME, etc). Delete them first or try a different hostname.

Cloudflare will not attach a Worker to a hostname that already has a
conflicting record, and it will not delete that record for you from that
screen. It is not a permissions problem.

### Procedure

1. **Wire up the lead form first (step 1).** Attaching the domain makes the new
   site live the moment it succeeds. A live site whose form posts to a
   placeholder is worse than the old site.
2. Cloudflare dashboard → click the domain → **DNS → Records**.
3. Find the row for `www`. **Screenshot it or write down its target first** —
   that is your undo.
4. Delete **only** that row. Leave every MX and TXT row alone.
5. Worker → **Settings → Domains & Routes → Add custom domain**. Type `www` in
   the **Subdomain** box.
   - **www first, not the root.** `DOMAIN` in `build.py` is the www host and it
     feeds every canonical tag, Open Graph URL, the sitemap and the schema.
     Every canonical on the current SitePanda site points at www too. Leaving
     the Subdomain box empty targets the apex, which is the wrong primary.
   - Cloudflare recreates the DNS record itself, pointed at the Worker.
6. Once www is serving the new site, repeat for the apex: delete the apex `A`
   row, then create a **Redirect Rule** — apex →
   `https://www.leamingtonsprayfoaminsulation.com/$1`, 301 permanent,
   preserving the path. Do not attach the apex to the Worker as a second
   custom domain.

---

## Step 7 — After the domain resolves

1. Visit all ten original URLs on the live domain. Every one should load its
   own page, not a redirect and not a 404:

   ```
   /                                   /crawl-spaces-insulation
   /services                           /new-construction-insulation
   /attic-insulation                   /insulation-removal-service
   /garage-insulation                  /about-us
   /basement-insulation                /contact-us
   ```

2. Google Search Console → add the property if it is not there → submit
   `https://www.leamingtonsprayfoaminsulation.com/sitemap.xml`.
3. Update the website link on the Google Business Profile if it points at a
   SitePanda URL rather than the domain.
4. Submit a real enquiry through the form on the live site.

**Only now cancel SitePanda.** Export anything you want to keep first — once it
is gone, the `lirp.cdn-website.com` image URLs die with it. That does not affect
this site, since every image is self-hosted, but any other place you pasted
those URLs will break.

---

## Faults found on the live SitePanda site

These are all fixed in the new build. Listed because several of them probably
exist on your other city sites too.

1. **Dead phone links.** Four CTA buttons link to `tel:555-555-5239`... to be
   exact, `tel:555-555-5555`. Two on `/services` and two on `/contact-us`. A
   visitor tapping "CONTACT US TODAY" on a phone dials nothing.
2. **A link to a page that does not exist.** `/services` links to
   `/spray-foam-insulation`, which returns an empty page.
3. **Copy-paste errors between service pages.** The garage page is headed "Is
   Your **Attic** Insulation In Need Of Service?". The mould section on the
   basement, crawl space and insulation-removal pages is word-for-word about a
   garage. The crawl space page's FAQ is headed "FAQ for **Basement**
   Insulation".
4. **A mangled paragraph** on the crawl space page: "...appeal of your homeA lot
   of people focus on..." — two paragraphs run together with no space.
5. **Price typo** on the garage page: "between $2,00 and $5,000".
6. **Spelling**: "commerial" on the home page, "Leamington, ONtario" on the
   attic page.

---

## Things that will bite you if you forget them

- **Rerun `python build.py` after editing `style.css` or `script.js`.** The HTML
  carries a content hash of each. Skip the rebuild and your change reaches
  nobody who has already visited.
- **`html_handling` in `wrangler.toml` must stay `auto-trailing-slash`.** Set it
  to `none` and `/` stops mapping to `index.html` and the home page 404s.
- **Never hand-edit anything in `site/`** except `style.css` and `script.js`.
  Everything else is regenerated from the markdown.
- **Do not use `Caledon Storefront.png`.** It is a photograph of another
  company's premises. It came with the shared pool and has been deleted from
  this folder.
- **Do not add links to your other city sites.** Four of them were on the old
  home page (Windsor, Bradford, Caledon, Milton) and they have been removed on
  purpose. Ten sites linking to each other is a recognisable network pattern.

---

## A decision you should make deliberately: sponsor links

All seven third-party outbound links from the old site are carried over:

| Page | Anchor text | Destination |
|---|---|---|
| Home | spray foam insulation | conroesprayfoaminsulation.com |
| Home | insulation advice | seattleinsulationcompany.com |
| Home | Closed cell spray foam | kirklandsprayfoaminsulation.com |
| Home | Attic spray foam | puyallupsprayfoaminsulation.com |
| Home | drywall contractor | santanvalleystuccorepair.com |
| About | basement renovation | buckeyestuccorepair.com |
| About | drywall contractor | madisonbasementfinishing.com |

They are **follow** links, which is what the SitePanda site does today.
Preserving current behaviour is the safe default for a migration.

But two of them are labelled "This site is sponsored by" in the visible copy.
If any of these are paid placements, Google's guidance is that they should carry
`rel="sponsored"`; a paid follow link is a link-scheme violation that can hurt
this site's own rankings. That is a commercial decision, not a technical one, so
I have not made it for you.

To change all seven at once, open `build.py`, find `EXTERNAL_REL` near the top of
the parser section, and change:

```python
EXTERNAL_REL = "noopener"
```

to:

```python
EXTERNAL_REL = "noopener sponsored"
```

Then rerun `python build.py`.

---

## Open items

- **Photos.** 27 of the 28 images supplied for Leamington are byte-identical to
  images already assigned on Windsor, Grimsby or Chatham spray foam. Every one
  is now in a different *slot* than it holds elsewhere, which is the part that
  reads as a network, but the pictures themselves are shared. **Windsor's own
  service area list contains "Leamington"**, so this is the closest overlap in
  the whole portfolio. Original photography is the single highest-value thing
  you could add, and the hero is the first slot to replace.
- **No gallery on this build.** After the eleven page slots were filled, every
  remaining image large enough for the lightbox was already in the Windsor or
  Grimsby gallery. Six original photos at 1000px or wider turns it back on; the
  note above `GALLERY_PHOTOS` in `build.py` says exactly how.
- **No commercial or greenhouse photography.** Those two new pages are using the
  closest available substitutes. A single warehouse interior and a single
  greenhouse headhouse shot would fix both.
- **Business hours** are not published anywhere, so `HOURS_TEXT` in `build.py`
  is the template default (Mon–Sat, 7am–6pm). Tell me the real hours and I will
  correct the schema.
- **`STREET_ADDRESS` and `POSTAL_CODE`** in `build.py` are `PLACEHOLDER`. Fine
  for a service-area business with no storefront, but if you have an address you
  want indexed it should go in.
- **`MAP_EMBED_URL`** is a hand-built Google Maps embed centred on Leamington
  rather than one copied out of the Maps share dialog. Worth replacing with a
  real one: maps.google.com → Share → Embed a map → copy the `src` value.
- **Pricing figures** in the FAQ are 2026 Ontario market ranges from research,
  not your prices. Swap in your own numbers when you can.
- **The Home Renovation Savings Program** details are current as of August 2026
  and the program runs to 30 November 2026. Diary a review before then, or the
  rebate copy on the home page, attic page and FAQ becomes wrong.
- **Territory overlap.** `chathamsprayfoaminsulation.com` covers Wheatley and
  Tilbury and runs greenhouse and agricultural pages. This site deliberately
  excludes Wheatley and Tilbury from its service area, but the new greenhouse
  page does compete with Chatham's. Worth a decision about which site you want
  to own greenhouse work in the county.
