# Dispatch: Build Guide for Claude Code

This file is the source of truth for building Dispatch. Read all of it at the start of every session. When something here conflicts with your own habits, this file wins. When something here conflicts with what Emmanuel says in the session, Emmanuel wins, and you update this file to match.

---

## 0. How to work with Emmanuel

Emmanuel is a senior product designer, not an engineer. He leads product and design; you write the code.

- **Explain in plain English.** After every change, say what you did and why in two or three sentences a non-engineer understands. No jargon without a one-line explanation.
- **Plan before you build.** At the start of each task, write a short plan (what you will create or change, in which files, and how he will test it). Wait for his "go" before writing code.
- **One slice per session.** Work through the build order in section 15. Do not start the next slice until the current one passes its "done when" checks.
- **Show, don't claim.** End each slice by telling him exactly what to click in the browser to see it working.
- **Ask before adding anything that costs money** or creates an account (a paid plan, a new service, a new third-party API).
- **Never put secrets in code.** API keys live only in `.env.local` (never committed) and in the hosting dashboard.
- **Commit after every working slice** with a clear message, so he can always go back.
- **Keep the Progress Log (section 18) current** at the end of every session.
- **Verify fast-moving facts.** Library versions, Resend API limits, Supabase and Vercel plan limits change. Check the current official docs before relying on a number in this file, and correct this file if it is out of date.
- **No em-dashes** in any UI copy or email template copy.

---

## 1. What Dispatch is

Dispatch is a personal email studio for designed email: newsletters, announcements, reminders and celebratory emails, sent to a chosen group in a few clicks.

It exists because Intercom, Mailchimp, Bunce and similar tools make two simple things hard:

1. **Designing an email.** Their editors do not let you clearly control sections: where images go, which text is a heading vs body, spacing between sections, image left / right / centre, text alignment, a proper footer.
2. **Sending to a group.** You should not need segments and workflow builders to email 40 people.

**Core promise:** open Dispatch, pick a template, edit blocks with precise control, pick who gets it, preview, send. Under five minutes for a familiar email.

**Owner and stage:** Emmanuel's personal project. Single user for now. It may later be pitched to his employer for team adoption, so build cleanly enough that multi-user can be added later, but do not build team features now.

**Hard boundary:** no employer data, brand files, domains or client lists go into this project. Sample brands only (OpenDraft, or invented ones).

---

## 2. Scope

### In v1 (build this)

| Capability | What it means |
| --- | --- |
| Block composer | Stack blocks (section 6). Every block has alignment, padding above and below, background, and type-specific controls. Drag to reorder, duplicate, delete, undo / redo. |
| Live preview | Desktop (600px) and mobile (375px) preview, plus a dark mode preview. Preview is the real rendered email HTML, not an approximation. |
| Brand presets | Saved brand: logo, colours, fonts (email-safe), button style, default footer. New emails inherit the brand. |
| Templates | Save any email as a template. Start new emails from a template or blank. Ship 4 starter templates: Newsletter, Announcement, Reminder, Celebration. |
| Audiences | Contacts with tags. CSV import with column mapping. Choose recipients by tag (any / all) or by picking individuals. Consent recorded per contact. |
| Send | Test send to yourself, send now, or schedule. Unsubscribe link and headers added automatically. |
| Reports | Per campaign: sent, delivered, opened, clicked, bounced, complained, unsubscribed. Per recipient status list. |
| Unsubscribe | One-click unsubscribe (email header) plus a visible unsubscribe link and a simple preferences page. |

### v2 (do not build yet)

- Recurring sends ("every month on the 25th").
- Dispatch API: other apps call a template by name and Dispatch sends it.
- Multi-column layout block.
- Subject line A/B tests.

### Never (out of scope)

Live chat, support tickets, SMS, complex branching automations / journey builders, landing page builder.

---

## 3. Tech stack (fixed decisions)

Do not swap these without asking Emmanuel. Use the latest stable versions and check current docs.

| Layer | Choice | Plain-English job |
| --- | --- | --- |
| App framework | Next.js (App Router) + TypeScript | The app itself: pages and server code |
| UI | opendraft design system (shadcn registry) + Tailwind CSS v4 | Every button, input, table and panel (section 4) |
| Editor state | Zustand (with an undo / redo history) | Remembers the email being edited |
| Drag and drop | dnd-kit | Reordering blocks |
| Rich text inside blocks | Tiptap, limited to bold, italic, underline, link, bulleted and numbered lists, line breaks | Editing body text |
| Validation | Zod | Checks every email document and form is well-formed |
| Database, login, file storage | Supabase (Postgres, Auth, Storage) | Stores everything; magic-link login; image hosting |
| Email rendering | React Email (`@react-email/components` and its `render` function) | Turns blocks into email HTML that works in Gmail, Outlook and Apple Mail |
| Email sending | Resend (REST API / Node SDK) | Delivers the emails and reports what happened |
| Webhook verification | Resend's signed webhooks (Svix-style signatures, check current docs) | Confirms delivery events really came from Resend |
| Scheduling | Supabase scheduled jobs (pg_cron) calling a server endpoint, OR Vercel Cron if the plan allows per-minute runs. Check plan limits first and propose one. | Sends scheduled campaigns on time |
| Hosting | Vercel | Puts the app online |
| CSV parsing | Papa Parse | Reads uploaded contact lists |
| Image compression | browser-image-compression (client side) | Keeps uploaded images small |

Package manager: `pnpm` unless Emmanuel already uses something else.

---

## 4. UI: opendraft design system

All app UI uses Emmanuel's opendraft design system. Full rules: https://raw.githubusercontent.com/nuelmdesign/UI-System/HEAD/public/llms.txt (and `llms-full.txt` for component props). Read it before the first UI slice.

### Setup (once)

1. `npx shadcn@latest init` if there is no `components.json`.
2. Add to `components.json`:
   ```json
   {
     "registries": {
       "@opendraft": "https://raw.githubusercontent.com/nuelmdesign/UI-System/HEAD/public/r/{name}.json"
     }
   }
   ```
3. `npx shadcn@latest add @opendraft/theme` first.
4. Clean `app/globals.css`: remove the starter `prefers-color-scheme` block, the starter `body` rule, and starter `:root` colour values.
5. Load fonts via `next/font/google`: Geist as `--font-geist`, Geist Mono as `--font-geist-mono`, Newsreader as `--font-newsreader`.
6. Dark mode via the `dark` class on `<html>` (use `next-themes` with `attribute="class"`).
7. Wrap the app in `<MotionConfig reducedMotion="user">`.

### Rules

- Use an opendraft component whenever one fits. Pass real data through props; never leave sample/demo content. Set `demo={false}` where that prop exists.
- Style only with tokens (`bg-background`, `bg-card`, `bg-muted`, `text-muted-foreground`, `bg-primary`, `text-brand`, `border-border`, `rounded-control`, `rounded-surface`, etc.). No Tailwind palette colours, no `dark:` colour twins, no hard-coded hex in the app UI.
- Hairlines over shadows. Shadows only on floating layers.
- Headings use the `heading` utility; labels and counts use `font-mono`; small section labels use `eyebrow`.
- Motion from `@/lib/motion`. Keep it short; animate transform and opacity only.
- Icon-only buttons get an `aria-label`.

**Important distinction:** opendraft styles the *Dispatch app*. The *emails* Dispatch produces are styled by the user's brand preset and rendered by React Email with inline styles. Never put opendraft tokens or Tailwind classes into email output.

### Components to reach for

Sidebar or Sidebar Nav (app shell), Button, Card, Input, Textarea, Select, Switch, Tabs, Radio Group, Checkbox, Dialog, Drawer, Dropdown Menu, Popover, Tooltip, Toast (sonner), Data Table (contacts, recipients), Filter Table, Badge (statuses), Date Range Picker (scheduling, reports), Fine-tune Card (inspector-style number scrubbing for padding and sizes), Command Palette (quick actions), Skeleton, Separator, Kbd, Animated Number (report counts), Copy Button.

---

## 5. Screens and navigation

App shell: left sidebar with **Campaigns, Templates, Audiences, Brand, Settings**. Top right: theme toggle, account menu.

### 5.1 Campaigns (home)
- Table of campaigns: name, subject, status badge (Draft, Scheduled, Sending, Sent, Failed), audience size, sent date, open rate, click rate.
- Primary action: **New campaign** (opens a dialog: start from template or blank; name it).
- Filter chips by status.

### 5.2 Composer (the heart of the product)
Three panels:

- **Left: Block library.** Grouped list of block types (section 6) with small icons. Click to add below the selected block, or drag into the canvas.
- **Centre: Canvas.** The email rendered at true width (600px) on a neutral background. Click a block to select it (brand-coloured outline + a small floating toolbar: move up, move down, duplicate, delete). Drag handle to reorder. Hover shows a "+" between blocks to insert. Toggle at top: Desktop / Mobile / Dark.
- **Right: Inspector.** Controls for the selected block, grouped as:
  - **Content** (text, image, link, button label)
  - **Layout** (alignment left / centre / right as a segmented control; image side left / right for image+text; width)
  - **Spacing** (padding top, bottom, left, right with a link toggle; scrub-able number fields)
  - **Style** (background colour, text colour override, border radius for images and buttons)
  - **Visibility** (hide on mobile)
  When nothing is selected, the inspector shows **Email settings**: subject, preview text (preheader), from name, reply-to, brand preset, content width, page background, content background.

Top bar of the composer: back, campaign name (editable), autosave status ("Saved"), Undo / Redo, **Send test**, **Next: Audience**.

Keyboard: Cmd/Ctrl+Z undo, Shift+Cmd/Ctrl+Z redo, Delete removes selected block, Cmd/Ctrl+D duplicates, arrow keys move selection.

Autosave every change after a 1 second pause.

### 5.3 Audience step
- Choose: **By tags** (any of / all of, with live recipient count) or **Pick people** (searchable table with checkboxes).
- Always show: total selected, how many will be skipped (unsubscribed, bounced, complained, no consent for marketing) and why.
- Campaign type toggle: **Marketing / newsletter** (requires marketing consent) or **Service / reminder** (sent to anyone in the audience who has not hard-bounced or complained, still with unsubscribe link). Default: Marketing.

### 5.4 Review and send
- Summary card: subject, preheader, from, reply-to, audience count, campaign type.
- Rendered preview (desktop / mobile).
- Pre-flight checks with pass / warn / fail: subject present, preheader present, every image has alt text, every link valid (https), unsubscribe present (always auto-added), audience not empty, sending domain verified, no unreplaced merge tags.
- Actions: **Send now** (confirmation dialog stating the exact count) or **Schedule** (date and time picker, show timezone; default Africa/Lagos).

### 5.5 Campaign report
- Counts with Animated Number: sent, delivered, opened, clicked, bounced, complained, unsubscribed. Rates as percentages of delivered.
- Recipient table: email, name, status, opened at, clicked at.
- Top clicked links.
- Note on the page in small text: open rates are approximate because some mail apps (notably Apple Mail) load images automatically.

### 5.6 Templates
- Grid of template cards with thumbnail (rendered snapshot), name, last edited.
- Actions: Use, Edit, Duplicate, Delete.

### 5.7 Audiences
- Contacts table: email, first name, last name, company, tags, marketing consent (yes / no / unknown), status (active, unsubscribed, bounced, complained), added date.
- Actions: Add contact, Import CSV, Bulk tag, Export CSV, Delete.
- Tags manager: create, rename, colour (from a fixed token-based set), delete.
- **CSV import flow:** upload, preview first 10 rows, map columns to fields (email required), choose tags to apply, choose consent status for this import and record the source (free text, e.g. "Sections signup form, Oct 2026"), then import. Show results: added, updated, skipped (invalid email, duplicate) with a downloadable error list.

### 5.8 Brand
- One or more brand presets. Fields: name, logo (upload), primary colour, text colour, link colour, page background, content background, heading font stack, body font stack, button style (radius, fill or outline), default footer (company name, postal address, social links).
- Live preview of a sample email using the preset.

### 5.9 Settings
- Sending: from name, from email, reply-to, verified sending domain status (read from Resend), test-send address.
- Defaults: timezone (Africa/Lagos), default brand.

### 5.10 Public pages (no login)
- `/u/[token]`: unsubscribe confirmation page, branded with the sender's brand preset. "You have been unsubscribed from [sender]." Option to re-subscribe.
- One-click unsubscribe POST endpoint for mail clients (section 11).

---

## 6. The email document model

Every email is a JSON document validated with Zod. Store it in the database as JSON. The composer edits it; the renderer turns it into HTML. Add a `schemaVersion` so the format can evolve.

```ts
type EmailDocument = {
  schemaVersion: 1
  settings: {
    subject: string
    preheader: string
    fromName: string
    replyTo?: string
    brandId: string
    contentWidth: number        // default 600, allowed 480 to 700
    pageBackground: string      // hex
    contentBackground: string   // hex
  }
  blocks: Block[]
}

type Spacing = { top: number; bottom: number; left: number; right: number } // px, 0 to 96, step 4
type Align = 'left' | 'center' | 'right'

type BlockBase = {
  id: string                    // nanoid
  type: BlockType
  padding: Spacing              // default { top: 16, bottom: 16, left: 24, right: 24 }
  background?: string           // hex, optional
  hideOnMobile?: boolean
}
```

### Block types (v1)

| Type | Fields (in addition to BlockBase) | Notes |
| --- | --- | --- |
| `logo` | `src`, `alt`, `width` (px), `align`, `href?` | Usually first block; defaults from brand |
| `hero` | `src`, `alt`, `href?`, `fullBleed` (bool), `radius` | Cover image. Full-bleed ignores left/right padding |
| `heading` | `text`, `level` (1, 2, 3), `align`, `color?` | Plain text only, no rich text. Sizes come from brand: h1 32, h2 24, h3 18 by default |
| `text` | `html` (Tiptap output, sanitised), `align`, `color?`, `fontSize` (14 to 20) | Body copy |
| `image` | `src`, `alt`, `width` (% of content, 10 to 100), `align`, `href?`, `radius`, `caption?` | |
| `imageText` | `image` {src, alt, href?, radius}, `imageSide` ('left' or 'right'), `imageWidth` (% 30 to 60), `heading?`, `html`, `button?` {label, href}, `verticalAlign` ('top', 'middle'), `stackOnMobile` (default true) | Image beside text; stacks on mobile with image first |
| `button` | `label`, `href`, `align`, `style` ('filled', 'outline'), `fullWidth` (bool), `color?` | Default style from brand |
| `divider` | `thickness` (1 to 4), `color`, `width` (% 10 to 100), `align` | |
| `spacer` | `height` (8 to 96) | No padding controls |
| `social` | `links`: [{ network, href }], `align`, `iconStyle` ('mono', 'color') | Networks: x, linkedin, instagram, facebook, youtube, tiktok, website |
| `footer` | `companyName`, `address`, `text?` (html), `showSocial` (bool), `align` | Unsubscribe and "update preferences" links are ALWAYS rendered by the renderer, not editable away |

**Rules**
- Every email must end with a `footer` block. The composer prevents deleting the last footer.
- Images must have `alt` text (pre-flight warns if empty).
- All links must be `https://` or `mailto:`.

### Merge tags
- Syntax: `{{first_name}}`, `{{last_name}}`, `{{company}}`, `{{email}}`, with fallback: `{{first_name|there}}`.
- Allowed in subject, preheader, heading, text, button label.
- Inserted from a small "Insert field" menu in the inspector.
- Pre-flight fails if an unknown tag is used.
- Escape all merge values before inserting into HTML.

---

## 7. Rendering emails

- One React Email component per block type in `/emails/blocks/*`. A `renderEmail(doc, brand, contact, links)` function returns `{ html, text }` using React Email's `render`.
- Email HTML rules (non-negotiable):
  - Table-based layout via React Email primitives (`Container`, `Section`, `Row`, `Column`, `Img`, `Text`, `Heading`, `Button`, `Link`, `Hr`).
  - Inline styles only. No external CSS, no Tailwind, no opendraft tokens.
  - Fixed content width (default 600px), centred, fluid on mobile.
  - Image `src` must be absolute https URLs. Set explicit width. Always include alt.
  - Fonts: use email-safe stacks (e.g. Arial/Helvetica, Georgia, Trebuchet, Verdana, system stack). A brand may add a Google Font `<link>` as an enhancement for clients that support it, always with a safe fallback.
  - Include the preheader as hidden preview text.
  - Add `color-scheme` meta and test dark mode; avoid transparent PNG logos on dark backgrounds without a fallback.
  - `imageText` stacking on mobile uses a responsive technique supported by React Email; verify in Gmail app and Outlook.
- Always generate a plain-text version.
- The canvas preview uses the same `renderEmail` output in a sandboxed iframe (`srcDoc`). Selection outlines and hover handles are drawn by the editor as an overlay mapped to block positions, OR the canvas renders editable React previews of each block that share styles with the email components. Propose the approach in the slice 1 plan; correctness of the final HTML matters more than editing tricks.
- Link tracking: use Resend's click tracking (check current docs) rather than building our own redirect service in v1.

---

## 8. Data model (Supabase / Postgres)

Every table has `id uuid primary key default gen_random_uuid()`, `owner_id uuid references auth.users`, `created_at`, `updated_at`. Enable Row Level Security on every table with policies limiting rows to `owner_id = auth.uid()`. This keeps the door open for multiple users later.

| Table | Key columns |
| --- | --- |
| `brands` | name, logo_url, colors (jsonb), fonts (jsonb), button (jsonb), footer_defaults (jsonb), is_default |
| `templates` | name, document (jsonb), thumbnail_url, brand_id |
| `campaigns` | name, document (jsonb), brand_id, type ('marketing','service'), status ('draft','scheduled','sending','sent','failed','cancelled'), audience_rule (jsonb), scheduled_at, sent_at, stats (jsonb cache) |
| `contacts` | email (unique per owner, lowercased), first_name, last_name, company, fields (jsonb for extra columns), status ('active','unsubscribed','bounced','complained'), marketing_consent ('yes','no','unknown'), consent_source, consent_at, unsubscribed_at |
| `tags` | name, color |
| `contact_tags` | contact_id, tag_id |
| `campaign_recipients` | campaign_id, contact_id, email (snapshot), status ('queued','sent','delivered','opened','clicked','bounced','complained','failed','skipped'), skip_reason, resend_email_id, opened_at, clicked_at, error |
| `email_events` | campaign_recipient_id, resend_email_id, type, payload (jsonb), occurred_at |
| `link_clicks` | campaign_id, url, count (or derive from events) |
| `assets` | url, path, width, height, size_bytes, alt_default |
| `imports` | filename, added, updated, skipped, error_report_url |
| `settings` | from_name, from_email, reply_to, test_email, timezone, default_brand_id |

Storage bucket: `email-assets` (public read, authenticated write). Compress images client-side to max 1200px wide (2x for retina at 600px).

Write migrations as SQL files in `/supabase/migrations`. Never edit the database by hand without a migration.

---

## 9. Sending pipeline

### Test send
- Renders with sample data (or a chosen contact), prefixes subject with `[Test]`, sends to the test address via Resend, does not create recipients or touch stats.

### Campaign send
1. Lock the campaign (`status = 'sending'`), snapshot the document.
2. Resolve the audience from `audience_rule`, then exclude: status unsubscribed / bounced / complained; for marketing campaigns, `marketing_consent != 'yes'`. Record exclusions as `skipped` with a reason.
3. Create `campaign_recipients` rows (`queued`).
4. For each recipient: render personalised HTML + text, build unsubscribe token and URLs.
5. Send in batches using Resend's batch endpoint (check the current max per batch and rate limits; throttle accordingly). Each email includes:
   - `headers`: `List-Unsubscribe: <https://APP_URL/api/unsubscribe/one-click?t=TOKEN>, <mailto:...>` (mailto optional) and `List-Unsubscribe-Post: List-Unsubscribe=One-Click`
   - `tags`: campaign id and recipient id, so webhook events map back
   - an idempotency key per recipient so retries never double-send
6. Save `resend_email_id`, mark `sent`. On per-email error, mark `failed` with the error and continue.
7. When all done, `status = 'sent'`, `sent_at = now()`.
8. The send must survive the request ending: run it as a background job (scheduled worker processing queued recipients in chunks), not one long request. Propose the exact mechanism in the slice plan based on current Vercel / Supabase limits.

### Scheduling
- `status = 'scheduled'` with `scheduled_at` (stored UTC, shown in the user's timezone).
- A job runs every minute, picks due campaigns, starts step 1.
- User can cancel or reschedule any time before it starts.

### Sending domain
- Sending uses a subdomain Emmanuel controls (e.g. `mail.<his-domain>`), verified in Resend with SPF, DKIM and DMARC.
- Settings shows verification status from the Resend API. Send buttons are disabled with an explanation until the domain is verified.

---

## 10. Webhooks and reports

- Endpoint: `POST /api/webhooks/resend`. Verify the signature with the signing secret (reject unsigned). Make processing idempotent (same event twice changes nothing).
- Map events to recipient status: delivered, opened (set `opened_at` first time), clicked (set `clicked_at`, count link), bounced (contact status `bounced` for hard bounces), complained (contact status `complained`, never email again), plus any failed / delayed events in current docs.
- Store every raw event in `email_events`.
- Update `campaigns.stats` cache after processing.

---

## 11. Unsubscribe, consent and compliance

These are requirements, not nice-to-haves.

- **Visible link** in every email footer: "Unsubscribe" and "Update preferences", rendered by the renderer, impossible to remove.
- **One-click unsubscribe** headers on every campaign email (RFC 8058): `List-Unsubscribe` with an https URL and `List-Unsubscribe-Post: List-Unsubscribe=One-Click`. The POST endpoint unsubscribes immediately with no confirmation page and returns 200.
- **Tokens** are signed (HMAC with `UNSUBSCRIBE_SECRET`) and encode contact id + campaign id. No login required. Never expose raw ids.
- Unsubscribes apply immediately (Gmail and Yahoo require honouring within two days; we do it instantly).
- **Consent:** marketing campaigns go only to contacts with `marketing_consent = 'yes'`. Every consent change stores source and timestamp. Imports require choosing consent status and source.
- **Suppression:** unsubscribed, bounced and complained contacts are never sent marketing again. Complained contacts get nothing at all.
- Footer must include a postal address (from brand footer defaults).
- Keep spam complaints under 0.3%; show the complaint rate on each report and warn if it rises above 0.1%.

---

## 12. Login and security

- Supabase Auth, email magic link. Only addresses in `ALLOWED_EMAILS` (comma-separated env var) may sign in; reject everyone else server-side.
- Supabase service role key used only in server code, never sent to the browser.
- RLS on every table.
- Sanitise all rich text HTML (allowlist of tags and attributes) on save and on render.
- Validate every request body with Zod.
- Rate-limit public endpoints (unsubscribe, webhooks).
- Never log full email bodies or contact lists in production logs.

### Environment variables (`.env.local`, and in Vercel)

```
NEXT_PUBLIC_APP_URL=
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
RESEND_API_KEY=
RESEND_WEBHOOK_SECRET=
UNSUBSCRIBE_SECRET=
ALLOWED_EMAILS=
CRON_SECRET=
```

Create a `.env.example` with these names and no values. Make sure `.env.local` is in `.gitignore`.

---

## 13. Starter templates

Build these with real, well-designed content (no lorem ipsum; use the invented brand "Northwind Studio" or OpenDraft). No em-dashes in copy.

1. **Newsletter:** logo, hero, heading, intro text, 2 x imageText (alternating image side), divider, text with link list, button, social, footer.
2. **Announcement:** logo, heading (h1), text, image, button, footer.
3. **Reminder:** logo, heading, text with the key date in bold, button ("Add to calendar" or "View details"), footer.
4. **Celebration:** logo, hero (warm image), heading ("Happy anniversary, {{first_name|friend}}"), text, button, footer.

---

## 14. Quality bar

- Feels calm and precise: the composer should feel like a design tool, not a form.
- Every control has a visible label or tooltip.
- Empty states for every list (no campaigns, no contacts, no templates) with one clear next action.
- Loading states with Skeleton; errors with Toast plus a plain-English message.
- Works in light and dark mode.
- Composer usable at 1280px wide; app usable (view-only acceptable) on mobile.

---

## 15. Build order (slices)

Each slice ends with Emmanuel testing in the browser. Do not move on until "Done when" is true.

### Slice 0: Project setup
- Next.js + TypeScript + Tailwind v4 + opendraft theme + fonts + dark mode + app shell with sidebar and empty pages.
- Supabase project connected, magic-link login with `ALLOWED_EMAILS`.
- `.env.example`, `.gitignore`, README with "how to run locally" in plain English.
- **Done when:** Emmanuel can sign in, see the sidebar, switch light / dark, and someone not on the allow-list is refused.

### Slice 1: Composer and live preview (no saving, no sending)
- Document model + Zod schemas, all v1 block types, React Email block components, `renderEmail`.
- Three-panel composer, add / select / reorder / duplicate / delete blocks, inspector controls, undo / redo, desktop / mobile / dark preview.
- **Done when:** he can build the Newsletter layout from scratch with every block type and see it correctly in all three previews.

### Slice 2: Save, templates, brands, images
- Database tables + migrations for brands, templates, campaigns, assets. Autosave.
- Image upload to Storage with compression.
- Brand page and presets; new emails inherit the brand.
- Templates page; the 4 starter templates seeded.
- **Done when:** he can create a brand, start from a template, edit, close the browser, come back and find it saved.

### Slice 3: Test send
- Settings page with sending details and domain status.
- Send test via Resend, plain-text version, preheader.
- **Done when:** a test email arrives in his Gmail and on his phone looking like the preview, in light and dark mode. Also check Outlook if available.

### Slice 4: Audiences
- Contacts, tags, CSV import with mapping and consent source, export.
- **Done when:** he imports a 50-row CSV, tags contacts, and sees correct counts and errors.

### Slice 5: Campaign send + unsubscribe
- Audience step, review and send with pre-flight checks, background sending, unsubscribe link, headers, token pages, consent and suppression rules.
- **Done when:** a send to 3 to 5 test addresses he owns arrives, unsubscribe works from both the link and Gmail's own unsubscribe button, and the unsubscribed address is skipped on the next send.
- **STOP after this slice** and remind Emmanuel to get a trusted engineer to review security before sending to anyone else.

### Slice 6: Scheduling
- Schedule, cancel, reschedule; background job.
- **Done when:** a campaign scheduled 5 minutes ahead sends on time without the browser open.

### Slice 7: Webhooks and reports
- Webhook endpoint with signature check, event processing, report page, campaign list stats.
- **Done when:** the report shows delivered, opened and clicked for his test sends, and a bounce test (use Resend's test addresses, check docs) marks the contact bounced.

### Slice 8: Polish
- Empty states, keyboard shortcuts, command palette, error handling, accessibility pass, performance pass.
- Deploy to Vercel with production env vars and webhook URL.
- **Done when:** he uses the live URL for a real send to his own opted-in list.

---

## 16. Testing checklist (run before calling a slice done)

- [ ] Type check and lint pass with no errors.
- [ ] Zod schemas have unit tests for each block type (valid and invalid examples).
- [ ] `renderEmail` snapshot test for each starter template.
- [ ] Unsubscribe token sign / verify tested, including tampered tokens.
- [ ] Webhook signature verification tested with a bad signature.
- [ ] Manual check in the browser by Emmanuel, following the steps you give him.

---

## 17. Reference: Bunce and Flodesk

Emmanuel will add screenshots to `/reference/bunce/` and `/reference/flodesk/`. Before slices 1, 2, 4 and 5, look at every screenshot in those folders and write notes below on:

- what each tool does well that Dispatch should match (layout of the composer, block controls, template browsing, audience selection, send flow);
- what is confusing or rigid that Dispatch must avoid (the reason Dispatch exists);
- specific patterns to adopt, described in Dispatch's own terms.

Learn from patterns; never copy their branding, illustrations, copy or exact visual design. Dispatch's look comes from opendraft.

### Notes from Bunce
_(Claude Code fills this in after reviewing `/reference/bunce/`.)_

### Notes from Flodesk
_(Claude Code fills this in after reviewing `/reference/flodesk/`.)_

### What Emmanuel likes and dislikes
_(Emmanuel's own notes go here.)_

---

## 18. Progress log

Update at the end of every session: date, slice, what was done, what is left, anything Emmanuel needs to do (accounts, keys, DNS).

| Date | Slice | Done | Next / blockers |
| --- | --- | --- | --- |
| 2026-10-09 | Repo setup | Repo created with this guide, `.gitignore`, `.env.example`, reference folders. Branch `slice-0-setup` created. | Start slice 0. Emmanuel has Supabase, Resend, Vercel and GitHub accounts. |

---

## 19. Glossary (plain English)

- **Block:** one section of an email (an image, a heading, a button).
- **Rendering:** turning the blocks into the code that email apps understand.
- **Preheader:** the grey preview line shown after the subject in the inbox.
- **Merge tag:** a placeholder like `{{first_name}}` replaced with each person's details.
- **Sending domain:** the web address emails come from; it must be verified so inboxes trust it.
- **SPF, DKIM, DMARC:** three settings on a domain that prove emails really come from you.
- **Webhook:** Resend calling Dispatch to say "this email was delivered / opened / bounced".
- **Bounce:** the email could not be delivered. **Complaint:** the person marked it as spam.
- **Suppression:** a list of people Dispatch will never email again.
- **RLS (Row Level Security):** database rule that makes sure each user only sees their own data.
- **Migration:** a saved script that changes the database structure, so changes are tracked.
- **Environment variables:** secret settings (like API keys) kept outside the code.
