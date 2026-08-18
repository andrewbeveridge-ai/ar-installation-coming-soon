# Another Realm Installation — capture-lead Form Contract

**For:** Work — Website Build (`andrewbeveridge-ai/ar-installation-coming-soon`)
**Authored:** 2026-08-03
**Basis:** live reads, not cache — capture-lead v30 source (`ezbr_sha256 3adf09a4b6e03f6d2a832808f12b0d9b4ec3b870d9a7e063eaafc33bda13a6a9`), `public.leads` `information_schema` dump, and an OPTIONS preflight probe of the live endpoint.
**Confidence:** High on everything in §1–§5 except where explicitly noted.

---

## 0. Endpoint

```
POST https://bgswqjgswlvdazseyhvu.supabase.co/functions/v1/capture-lead
Content-Type: application/json
```

- `verify_jwt = false`. **No `Authorization` header, no `apikey` header, no `x-capture-token`.** `CAPTURE_TOKEN` is deliberately unset (see TechOps StateDoc 2026-07-16 — do not reintroduce it).
- Success: `200 {"ok":true}`.
- Failure codes the form must handle: `403 forbidden_origin`, `400 entity_required` / `dba_required` / `contact_required` / `invalid_json`, `413 payload_too_large`, `429 rate_limited` (per-IP, 10 per 600s, sends `Retry-After: 600`), `500 insert_failed`.
- A honeypot hit returns `200 {"ok":true}` — indistinguishable from success by design. Do not "verify" the form by filling the honeypot.

---

## 1. Source values and the welcome-email gate

### 1a. The gate, as it actually exists in v30

```
routeFor(dba, source):
  if source === "web:home" AND INTROS[dba] exists  -> "intro"
  if QUOTES[dba + "|" + source]  exists            -> "quote"
  if LISTJOIN[dba + "|" + source] exists           -> "listjoin"
  otherwise                                        -> "unrouted"
```

`INTROS` contains `"Another Realm Installation"`. `QUOTES`, `LISTJOIN`, and `BRAND` **do not**.

Consequence: **any source value other than `web:home` is `unrouted` for this DBA today.** Unrouted is fail-safe, not fail-silent-and-lose — the lead still inserts, the owner alert still fires, and it carries an amber `⚠ UNROUTED SOURCE` banner. But the customer receives nothing at all.

### 1b. Primary form — LOCKED

| Key | Value |
|---|---|
| `source` | `web:home` |

Do not change, do not "improve" to `web:lead` or `web:quote`. The live welcome email is gated on this exact string plus an exact `dba` match. Both must hold or the intro silently does not send.

### 1c. If quote / booking / contact forms are added

Proposed source values, and what each requires:

| Form | `source` | Route it would need | Code change required |
|---|---|---|---|
| Quote request | `web:quote` | `quote` | Yes — two map entries |
| Contact | `web:quote` (reuse) or `web:contact` | `quote` | Yes — two map entries |
| Booking | `web:booking` | `quote` | Yes — two map entries |

**The gate change, precisely.** Adding a route is *two* edits, not one, and missing the second one fails silently:

1. `QUOTES["Another Realm Installation|web:quote"] = { subject, lines: [...] }`
2. `BRAND["Another Realm Installation"] = { from: "info@anotherrealminstallation.com", website: "anotherrealminstallation.com", accent: "#39FF14", entity: CBUS_ENTITY, phone: <line>, signerTitle: "Founder & CEO", theme: THEME_DARK-derived }`

The send path is guarded by `if (cfg && brand)`. A `QUOTES` entry without a `BRAND` entry produces **no email and no error** — the route resolves, the guard fails, the function returns. This is the single most likely way to build this wrong.

**Recommendation for v1: ship exactly two sources — `web:home` (unchanged) and `web:quote` (one code change, one route, reused by both the quote form and the contact form).** Booking gets its own source at v4, not before. Fewer routes, fewer silent-failure surfaces.

**Two behaviours to decide before adopting the quote route:**

- The quote route **auto-subscribes** the email address to `list_members` (`dba` + list `updates`) unless suppressed, and appends a CAN-SPAM transparency line plus `List-Unsubscribe` headers to the ack. A quote request becomes a marketing list membership. Acceptable, but it is a consent-provenance decision, not a default.
- `BRAND` also drives `preferred_contact` / `service_wanted` / `location` / `notes` read-back in the ack ("Please review the info you sent us"). Whatever fields the form collects will be echoed to the customer. Write field labels accordingly.

**Hard constraint:** do not send `timeframe: "emergency"`. The Pushover priority-2 siren branch is hardcoded to `dba === "Another Realm Compliance"` so Installation cannot fire it today — but do not reuse that field name expecting inert behaviour.

---

## 2. Exact `dba` string

```
Another Realm Installation
```

Send it exactly. Matching is normalised (`toLowerCase`, non-alphanumerics collapsed to spaces, trimmed), so casing and punctuation variance is tolerated and canonicalised on write.

**`"Another Realm Digital Installation"` is not a variant — it is a different string.** It does not normalise to a registry entry. Result: `dbaUnknown`, lead accepted and flagged `[UNKNOWN DBA]`, stored under the literal wrong name, `INTROS` lookup misses, **no welcome email**, and the entity auto-correct guard has nothing to correct against. Fails quietly on the customer-facing side. Same failure shape for any other drift.

---

## 3. Full field list

### Required

| JSON key | Type | Value / constraint | Failure if wrong |
|---|---|---|---|
| `entity` | string | `"CBUS"` — compared uppercase; must be `ARAI` or `CBUS` | `400 entity_required` |
| `dba` | string | `"Another Realm Installation"`, ≤120 chars | `400 dba_required` |
| `phone` **or** `email` | string | at least one must be non-empty | `400 contact_required` |

`dba` is authoritative over `entity`. A wrong `entity` is auto-corrected from the registry and flagged `[ENTITY FIXED]` — never rejected. Send `CBUS` anyway; do not rely on the guard.

### Optional — mapped to real columns

| JSON key | Max | Column | Notes |
|---|---|---|---|
| `source` | 200 | `source` | see §1 |
| `name` | 200 | `name` | |
| `email` | 320 | `email` | MX-probed; no-MX domain ⇒ lead saved, ack skipped, alert flagged `[UNDELIVERABLE EMAIL]` |
| `phone` | 50 | `phone` | rendered as a `tel:` link in the owner alert |
| `service_wanted` | 500 | `service_wanted` | |
| `location` | 200 | `location` | |
| `notes` | 2000 | `notes` | over-length truncated + flagged; untruncated original preserved in `raw` |
| `preferred_contact` | — | `preferred_contact` | **enum, lowercased: `phone` \| `email` \| `text` \| `any`.** Anything else is silently coerced to `null` |

### Required anti-spam field

| JSON key | Value |
|---|---|
| `_hp` | empty string — present in the payload, hidden in the DOM, never populated by a human |

Any non-empty `_hp` returns `200 {"ok":true}` and drops the submission entirely. **Do not omit this field in the rebuild.** It is the only spam control on the write path.

### `sms_consent` — the Arch City defect, do not repeat

Not a column. Passed through into `raw` jsonb. `leads.sms_ok` is a **generated** boolean derived from it. Live definition:

```sql
CASE
  WHEN (raw ->> 'sms_consent') IS NULL          THEN NULL
  WHEN lower(raw ->> 'sms_consent') = 'true'    THEN true
  WHEN lower(raw ->> 'sms_consent') = 'false'   THEN false
  ELSE NULL
END
```

Rules, in order of how easy they are to get wrong:

1. **Send a JSON boolean `true` / `false`.** The strings `"true"` / `"false"` also work (case-insensitive). Everything else lands `NULL`.
2. **`"on"` lands `NULL`.** That is the default value an HTML checkbox serialises to. Naïve `FormData` → JSON serialisation of a checkbox produces `"on"` and silently destroys the consent record. This is very likely the exact Arch City failure.
3. **`"yes"`, `"1"`, `"checked"`, `""` all land `NULL`.**
4. **Always send the key, even when unchecked.** Unchecked ⇒ send `false`. Omitting the key ⇒ `NULL` ⇒ "we never asked" ⇒ the record is unusable for 10DLC and indistinguishable from an Arch City-class bug in an audit.

Correct pattern:

```js
sms_consent: document.getElementById('sms_consent').checked   // boolean, always sent
```

Three-state meaning downstream: `true` = explicit consent, `false` = explicitly declined, `NULL` = unknown/not asked. Any future SMS send **must** filter `sms_ok IS TRUE`.

### Any other key

Passed straight into `raw` and auto-surfaced in the owner alert under "Additional submitted fields". No code change needed to add a field — but nothing downstream reads it either. Useful for Installation: `tv_size`, `mount_type`, `wall_type`, `preferred_date`.

### Reference payload

```json
{
  "entity": "CBUS",
  "dba": "Another Realm Installation",
  "source": "web:home",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "614-555-0142",
  "service_wanted": "TV mounting — 65\" over fireplace",
  "location": "Worthington, OH",
  "preferred_contact": "text",
  "notes": "Drywall over brick, needs cable concealment",
  "sms_consent": true,
  "_hp": ""
}
```

---

## 4. Origin allowlist — confirmed live

Server-side origin enforcement has been in place since v16: an unlisted or absent `Origin` header returns **403 before the insert and before any email**. This is a real access control, not just a CORS header.

Probed against the live endpoint 2026-08-03:

| Origin | Result | Verdict |
|---|---|---|
| `https://anotherrealminstallation.com` | echoed back | **allowed** |
| `https://www.anotherrealminstallation.com` | echoed back | **allowed** |
| `http://localhost:8888` | echoed back | **allowed** |
| `https://ar-installation-coming-soon.netlify.app` | `null` | **403** |
| `https://evil.example.com` | `null` | **403** |

Apex, www, and the local-dev path are all confirmed working. **No allowlist change is needed for this build.**

Two operational consequences:

- **Netlify deploy previews and branch deploys will 403.** Do not attempt end-to-end form validation on a preview URL and conclude the form is broken. Test locally on `http://localhost:8888` (`netlify dev` default port — do not change it) or on the live domain after deploy.
- The effective list is governed by the `ALLOWED_ORIGINS` Supabase secret when set, which shadows the code's `DEFAULT_ALLOWED`. That secret is **write-only and cannot be read back**. The probe above is empirical proof of effective behaviour regardless of which tier is governing, so this does not block the build — but anyone editing the allowlist must also update the Bitwarden mirror ("Supabase Allowed Origins Secret") in the same action.

---

## 5. Does v30 need a change?

**Split answer — this is the load-bearing part of the spec.**

| Scope | v30 change needed? |
|---|---|
| v1 primary form on `web:home` | **No.** The form adapts. Route, welcome email, DBA registry entry, entity mapping, and origin allowlist are all already live and correct. |
| `sms_consent` capture | **No.** It is a `raw` passthrough; `sms_ok` is a generated column that derives it automatically and retroactively. No redeploy, no migration. Purely a form-side obligation. |
| Any new source (`web:quote`, `web:booking`, `web:contact`) | **Yes.** `QUOTES` entry **and** `BRAND` entry, per §1c. Repo `andrewbeveridge-ai/edge-functions`, edit → commit → pull on Realm-CEO → deploy → curl-verify. One deploy, per D1. |

**Net: ship v1 with zero backend changes.** That is the correct sequencing — get the site live, then add the quote route as a separate scoped deploy once the pricing catalog exists.

### One v30 change worth considering when the quote route is built

For the `intro` route, `OWNER_LABEL` is `"LIST JOIN"`. On a live services site, a home-page form submission is a lead, and the owner alert will read `LIST JOIN - Another Realm Installation`. Cosmetic, but it will misfile in triage. Fold the relabel into the same deploy as the quote route rather than taking a deploy for it alone.

---

## 6. Acceptance tests before the Website Build thread closes

Run on `http://localhost:8888`, then repeat once on the live domain.

1. Submit the primary form with `sms_consent` **checked**. Verify: `200 {"ok":true}`; row lands with `entity='CBUS'`, `dba='Another Realm Installation'`, `source='web:home'`, **`sms_ok = true`**; welcome email arrives from `info@anotherrealminstallation.com`; owner alert arrives with no warning banners.
2. Submit with `sms_consent` **unchecked**. Verify **`sms_ok = false`**, not `NULL`. *This is the test Arch City did not have.*
3. Submit with the honeypot populated. Verify `200 {"ok":true}` and **no row**.
4. Submit with `email` omitted and `phone` present. Verify `200`.
5. Submit with both omitted. Verify `400 contact_required` and a usable inline error, not a silent failure.
6. Confirm every test row is deleted afterward. The CRM is currently a pristine slate — zero genuine leads have ever been captured portfolio-wide — and it should stay that way until real traffic begins.

---

## RISK FLAG: Execution — the preserved welcome email is now wrong copy

Holding `source = "web:home"` preserves the live welcome email, which is the correct call mechanically. But that email is coming-soon copy: it closes with *"We are booking our first clients soon. You will hear from us when we open the schedule."*

The moment the site goes from coming-soon to a real Tier-1 services site, every person who fills in the primary form asking to have a TV mounted receives a message telling them the business is not open yet. The gate is preserved and the messaging is broken — which is worse than a hard failure, because nothing alerts on it.

**Mitigation:** rewrite the `INTROS["Another Realm Installation"]` `hook` / `body1` / `body2` / `cta` strings to live-services copy, and fold that edit into the same deploy as the quote route. It is a four-string change with no structural risk. Do not let it ride to v3 — it goes live the day the site does.

---

## Handoff notes

- Site code lives in GitHub only. No `.html` / `.css` / `.js` in Drive (D4).
- Rebuild the Netlify project **in place**. Never delete and recreate — a deleted project leaves dangling DNS and arms a subdomain-takeover vector on an Another Realm hostname.
- 10DLC readiness at v1 (build it, do not submit): SMS opt-in checkbox wired per §3, privacy policy carrying an explicit **SMS sharing** carve-out (not merely "we do not sell"), Terms and Privacy linked **in-form**, not footer-only.
- Business address for Terms/Privacy remains **blocked** on the unresolved CBX registered-address conflict. Do not guess it into the pages.
