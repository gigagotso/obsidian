# Contact Correlation Architecture

> **Status:** Proposal / RFC
> **Author:** Engineering
> **Last updated:** 2026-06-09
> **Scope:** Unify per-channel `visitors` into a single cross-channel **Contact** record so that the same human contacting us by phone, WhatsApp, web chat, Messenger, email, etc. is recognized as one person.

---

## 1. Problem statement

Today a **visitor** is created per *channel touchpoint*. The same human appears as **many unrelated visitor rows**:

- Calls the PBX from `+995 555 12 34 56` → visitor A (`mobile = 995555123456`, `extra_attributes.pbx.extension`).
- Messages us on WhatsApp from the **same** number → visitor B (`extra_attributes.social_hub.profile.id = ...`).
- Chats on the website anonymously → visitor C (anonymous, no identifier).
- Emails `john@acme.com` → visitor D (`email = john@acme.com`).
- Messages on Facebook Messenger → visitor E (`social_hub.profile.id`).

There is **no entity that ties A–E together.** Agents see five strangers; reporting double-counts people; history is fragmented; routing/CRM context is lost.

We want a **`contacts`** table (the *person*) where every visitor row correlates by `contact_id`, plus the matching machinery to populate and maintain it.

This is a classic **identity resolution / entity resolution** problem. The rest of this document covers the current state, the proposed data model, the matching strategy, merge/split mechanics, integration points, backfill, edge cases, and the trade-offs of the alternatives.

---

## 2. Current state (as built)

### 2.1 Multi-tenancy
Everything is scoped by **`account_id`** (FK → `accounts`, cascade). **All correlation must be scoped to `account_id`.** Two different accounts may legitimately have the same phone number; they must **never** merge.

### 2.2 The `visitors` table
`app/Models/Visitor.php` — table `visitors`:

| Column                         | Type               | Meaning                                                         |
| ------------------------------ | ------------------ | --------------------------------------------------------------- |
| `id`                           | bigint PK          |                                                                 |
| `uuid`                         | varchar unique     | public id                                                       |
| `account_id`                   | bigint FK          | tenant                                                          |
| `name`                         | varchar null       | display name                                                    |
| `email`                        | varchar null       | email identity (any channel)                                    |
| `mobile`                       | varchar null       | phone / PBX extension                                           |
| `remote_user_id`               | varchar null       | external/CRM id (unique per account)                            |
| `social_hub_profile_id`        | bigint generated   | from `extra_attributes->social_hub.profile.id`                  |
| `social_hub_profile_id_unique` | bigint null unique | denormalized for dedup                                          |
| `settings`                     | json               |                                                                 |
| `extra_attributes`             | json               | schemaless: `pbx.extension`, `social_hub.{profile,provider}`, … |
| timestamps + `deleted_at`      |                    | soft delete                                                     |

Key observation: **the visitor row already plays two roles at once** — it is both "the person" (name/email) *and* "a channel endpoint" (pbx extension, social-hub profile). The correlation work is essentially splitting these two roles apart.

### 2.3 How each channel identifies a visitor today

| Channel | Resolve/create logic | Identifier stored |
|---|---|---|
| **Web / widget** | `Widgets/VisitorController` → `firstOrCreate(['remote_user_id' => jwt.sub])` or anonymous `create()` | `remote_user_id` or none |
| **PBX** | `Actions/PBX/GetVisitor` → match on `extra_attributes->pbx->extension` **and** `mobile` | `mobile`, `pbx.extension` |
| **WhatsApp / Messenger / Instagram / Viber / Telegram / Email (Social Hub)** | `Actions/SocialHub/GetVisitor` → match on `social_hub.profile.id` (unique constraint) | `social_hub.profile.id` |
| **Email** | as Social Hub, plus `email` column | `email`, `social_hub.profile.id` |

These three **`GetVisitor` actions are the choke points** — the only places a visitor is resolved or created. They are exactly where contact resolution must hook in.

### 2.4 Channels
- `channels` table — 9 types: `website, messenger, telegram, viber, whatsapp, instagram, email, pbx, kiosk` (`app/Models/Channel.php`).
- `channel_lines` — per-account channel instances (a WhatsApp number, a PBX trunk, a widget…).

### 2.5 Conversations
- `conversations` linked to visitors via the polymorphic pivot `conversation_members` (`member_type = visitor`).
- Messages in `conversation_messages` carry `member_type_id` + `member_id`.
- **Important:** conversations point at *visitors*, not contacts. Correlation must not break this; `contact_id` is an **additive** layer above visitors.

### 2.6 What does NOT exist today
- No contact / customer / person / lead / identity model (the only `Contact` is `App\Models\PBX\Contact` → Asterisk `ps_contacts`, unrelated).
- No phone-number normalization (E.164).
- No merge / dedup / suggested-match logic.

---

## 3. Design goals & principles

1. **Additive, non-breaking.** `contact_id` sits *above* visitors. Conversations, messages, PBX, social hub keep working untouched. Every visitor keeps belonging to exactly one contact.
2. **One choke point per write.** Resolution happens only in the three `GetVisitor` actions; nothing else creates visitors.
3. **Normalize before you match.** Phone → E.164, email → lowercased canonical. Matching on raw strings is the #1 source of both false splits and false merges.
4. **Deterministic first, probabilistic later.** Auto-link only on strong, verified identifiers. Everything fuzzy becomes a *suggestion* a human confirms.
5. **Tenant isolation is absolute.** All uniqueness and matching keyed by `account_id`.
6. **Reversible.** Every auto-merge is logged and undoable. Identity resolution *will* make mistakes; design for split/unmerge from day one.
7. **The visitor becomes a "channel identity."** Long-term mental model: `contact` = person, `visitor` = one of the person's endpoints on one channel.

---

## 4. Proposed data model (recommended: Option B)

Three pieces: a **`contacts`** table (the person), a normalized **`contact_identities`** table (the matchable keys), and a new **`visitors.contact_id`** FK.

```
                        ┌────────────────────┐
                        │      contacts       │  the person / org
                        │  id, account_id,    │
                        │  display_name,      │
                        │  primary_email,     │
                        │  primary_phone,     │
                        │  merged_into_id ────┼──┐ self-ref (tombstone)
                        └─────────┬──────────┘  │
                                  │ 1            │
                 ┌────────────────┼──────────────┘
                 │ N              │ N
   ┌─────────────▼──────────┐   ┌▼───────────────────────┐
   │   contact_identities    │   │        visitors         │
   │  contact_id, channel_id,│   │  + contact_id (FK)      │
   │  type, value_normalized,│   │  (existing table,       │
   │  value_raw, confidence, │   │   one row per channel   │
   │  verified_at, source    │   │   touchpoint)           │
   └─────────────────────────┘   └─────────────────────────┘
```

### 4.1 `contacts`

```php
Schema::create('contacts', function (Blueprint $table) {
    $table->bigIncrements('id');
    $table->uuid('uuid')->unique();
    $table->foreignId('account_id')->constrained()->cascadeOnDelete();

    $table->string('display_name')->nullable();
    $table->string('primary_email')->nullable();      // denormalized convenience
    $table->string('primary_phone')->nullable();      // E.164, denormalized convenience
    $table->string('avatar_url')->nullable();

    // Merge tombstone: when this contact is merged INTO another, point here.
    $table->unsignedBigInteger('merged_into_id')->nullable()->index();

    $table->extraAttributes();                         // schemaless extras (matches house style)
    $table->timestamps();
    $table->softDeletes();

    $table->foreign('merged_into_id')->references('id')->on('contacts')->nullOnDelete();
    $table->index(['account_id', 'primary_phone']);
    $table->index(['account_id', 'primary_email']);
});
```

Notes:
- `merged_into_id` makes a contact a **tombstone** pointing at the surviving contact (soft merge — reversible, auditable). A "live" contact has `merged_into_id IS NULL`.
- `primary_*` are denormalized for fast list/search rendering; the source of truth is `contact_identities`.

### 4.2 `contact_identities` — the matching index

This is the heart of correlation. **One row per (channel, identifier) the contact is known by.** A single visitor can contribute several identities (a WhatsApp visitor yields both a `whatsapp` identity *and* a `phone` identity — see §5.3).

```php
Schema::create('contact_identities', function (Blueprint $table) {
    $table->bigIncrements('id');
    $table->foreignId('account_id')->constrained()->cascadeOnDelete();
    $table->unsignedBigInteger('contact_id');

    $table->unsignedTinyInteger('channel_id')->nullable();   // FK -> channels (origin channel)
    $table->string('type', 32);                              // 'phone' | 'email' | 'whatsapp'
                                                             // | 'messenger_psid' | 'instagram_id'
                                                             // | 'telegram_id' | 'viber_id'
                                                             // | 'social_hub_profile' | 'web_uuid'
                                                             // | 'remote_user_id'
    $table->string('value_normalized');                      // E.164 phone, lowercased email, raw id
    $table->string('value_raw')->nullable();                 // what we actually received

    // 100 = verified/deterministic, lower = inferred/fuzzy
    $table->unsignedTinyInteger('confidence')->default(100);
    $table->string('source', 32)->nullable();                // 'pbx' | 'social_hub' | 'widget' | 'manual' | 'merge'
    $table->timestamp('verified_at')->nullable();

    $table->timestamps();
    $table->softDeletes();

    $table->foreign('contact_id')->references('id')->on('contacts')->cascadeOnDelete();
    $table->foreign('channel_id')->references('id')->on('channels');

    // The critical guard: an identifier maps to AT MOST ONE contact per tenant.
    $table->unique(['account_id', 'type', 'value_normalized'], 'identity_unique');
    $table->index(['contact_id']);
});
```

The `identity_unique` constraint is what makes correlation deterministic and race-safe: when two channels present the same normalized identifier, they collide on this unique key and resolve to the same `contact_id`.

### 4.3 `visitors.contact_id`

```php
Schema::table('visitors', function (Blueprint $table) {
    $table->unsignedBigInteger('contact_id')->nullable()->after('account_id');
    $table->foreign('contact_id')->references('id')->on('contacts')->nullOnDelete();
    $table->index('contact_id');
});
```

Nullable during rollout/backfill; effectively always set once live. **A visitor belongs to exactly one contact; a contact has many visitors.**

### 4.4 (Optional) `contact_merges` — audit log

```php
Schema::create('contact_merges', function (Blueprint $table) {
    $table->bigIncrements('id');
    $table->foreignId('account_id')->constrained()->cascadeOnDelete();
    $table->unsignedBigInteger('survivor_id');     // contact kept
    $table->unsignedBigInteger('merged_id');       // contact tombstoned
    $table->string('reason', 64);                  // 'auto:phone' | 'auto:email' | 'manual'
    $table->unsignedBigInteger('actor_user_id')->nullable();
    $table->json('snapshot')->nullable();          // pre-merge state for undo
    $table->timestamps();
});
```

Enables unmerge and a defensible audit trail (GDPR, disputes, debugging false merges).

---

## 5. Matching / correlation strategy

### 5.1 Normalization (do this before *any* match)

| Type                                                                        | Rule                                                                                                                                                                                      |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **phone / whatsapp / viber**                                                | Parse to **E.164** with `giggsey/libphonenumber-for-php`. Use the account's default region as a hint for numbers without `+`. Store E.164 in `value_normalized`, original in `value_raw`. |
| **email**                                                                   | `trim` + `mb_strtolower`. Optionally canonicalize Gmail (strip dots & `+tag` in local part) — **store both** so you don't lose the raw address.                                           |
| **social ids** (psid, instagram, telegram, viber id, social_hub profile id) | Opaque — store as-is (string). These are stable per platform but **not** cross-platform.                                                                                                  |
| **web uuid / remote_user_id**                                               | As provided.                                                                                                                                                                              |

> ⚠️ **Without phone normalization, PBX↔WhatsApp correlation does not work.** `+995 555 12 34 56`, `995555123456`, and `00995555123456` are the same person but three different strings.

### 5.2 Resolution algorithm (deterministic core)

On every visitor resolve/create, for each identifier the channel gives us:

```
resolveContact(account, identifiers[]):
  candidates = {}
  for id in identifiers:
      norm = normalize(id)
      row  = contact_identities.where(account, id.type, norm).first()
      if row: candidates.add(row.contact_id)

  if candidates is empty:
      contact = Contact::create(account, seedFromIdentifiers)
  elif candidates has 1:
      contact = that contact
  else:
      # multiple existing contacts share these identifiers ->
      # this touchpoint BRIDGES them. Auto-merge (strong signals only),
      # else create suggested-merge and pick the oldest as primary.
      contact = mergeContacts(candidates)   # see §6

  for id in identifiers:
      upsert contact_identities(account, contact, id.type, norm, confidence, source)
  return contact
```

Wrap in a DB transaction; rely on the `identity_unique` constraint to make concurrent first-contact races resolve to one winner (catch duplicate-key, re-read).

### 5.3 The high-value correlations (deterministic, auto-link)

| Signal | Why it's strong | Action |
|---|---|---|
| **Same E.164 phone across PBX and WhatsApp/Viber** | WhatsApp/Viber identity **is** a phone number. This is the single biggest win. | Emit both a `phone` and a `whatsapp` identity from the WhatsApp visitor; auto-link to the PBX contact sharing that `phone`. |
| **Same normalized email** across email + web + social hub | Email is a strong global identifier. | Auto-link. |
| **Same `remote_user_id`** (your CRM/app user id) | Explicit external identity. | Auto-link, highest confidence. |
| **Same `social_hub.profile.id`** | Already deduped by existing unique column. | Maps 1:1 to one contact. |

> **Integration prerequisite to verify:** the Social Hub payload must expose the WhatsApp/Viber **phone number** (not just an opaque `profile.id`). If the phone is only in `profile.name` or a provider field, extraction has to be added in `Actions/SocialHub/GetVisitor` for phone-based correlation to fire. **This is the one external dependency that gates the headline feature — confirm it first.**

### 5.4 Weak signals (suggest, never auto-merge)

- Same display name only.
- Same name + same city / partial.
- Two anonymous web sessions.

These produce **suggested merges** surfaced in the agent/admin UI for one-click confirm. Auto-merging on these *will* fuse strangers.

### 5.5 Confidence model
`confidence` on each identity drives policy: `100` (verified phone/email/remote id) → auto-link; `< 100` (name-inferred) → suggestion only. The same threshold decides whether a *bridge* (§5.2) auto-merges or just suggests.

---

## 6. Merge & split mechanics

### Merge (two contacts found to be the same)
1. Pick a **survivor** (oldest, or most complete, or the one with a verified identity).
2. Re-point all `visitors.contact_id` and `contact_identities.contact_id` from `merged` → `survivor` (identities may hit the unique constraint → dedup, keep highest confidence).
3. Fill survivor's empty `primary_*` / `display_name` from the merged contact.
4. Set `merged.merged_into_id = survivor.id`, soft-delete `merged`.
5. Write a `contact_merges` row with a `snapshot` for undo.
6. Fire `ContactsMerged` event.

Reads should **follow `merged_into_id`** (or, simpler, never return tombstoned contacts — re-point everything at merge time so live queries only see survivors). A tombstone exists mainly for unmerge + dangling references.

### Split / unmerge
Use the `contact_merges.snapshot` to recreate the merged contact and re-point the identities/visitors that originally belonged to it. This is why merge must be a logged, structured operation rather than a destructive `UPDATE`.

### Conflict: identifier already on a different contact
When an incoming identity's `(account, type, value)` already points to contact X but the current touchpoint resolved to contact Y:
- **Strong identifier (phone/email/remote_id):** treat as a bridge → merge X and Y (or suggest, per confidence).
- **Weak identifier:** keep on X, raise a suggestion. Never silently move a verified identity.

---

## 7. Integration points (where code changes land)

| Location | Change |
|---|---|
| `app/Actions/PBX/GetVisitor.php` | After resolving the visitor, call `ContactResolver` with a `phone` identity (E.164 of `mobile`/extension). Set `visitor.contact_id`. |
| `app/Actions/SocialHub/GetVisitor.php` | Emit identities for `social_hub_profile`, the channel-native id (`whatsapp`/`messenger_psid`/…), **and a `phone` identity when the provider is WhatsApp/Viber**. Set `visitor.contact_id`. |
| `app/Http/Controllers/Widgets/VisitorController.php` | Emit `web_uuid` and/or `remote_user_id` identities. Anonymous sessions get their own contact, upgraded/merged once an email/phone is captured. |
| **New** `app/Actions/Contacts/ResolveContact.php` | The shared resolver implementing §5.2; the single entry point all three call. |
| **New** `app/Support/PhoneNormalizer.php` | E.164 normalization wrapper around libphonenumber. |
| `app/Models/Visitor.php` | Add `contact(): BelongsTo`. |
| **New** `app/Models/Contact.php`, `ContactIdentity.php` | Models + relations (`contact.visitors()`, `contact.identities()`). |

**Eventing:** `VisitorUpdated` already exists. Add `ContactCreated`, `ContactsMerged`, `ContactMergeSuggested` so the agent UI / reporting can react in real time.

---

## 8. Backfill & rollout

**Phase 0 — schema.** Ship `contacts`, `contact_identities`, `visitors.contact_id` (nullable), models. No behavior change.

**Phase 1 — 1:1 backfill.** For every existing visitor, create one contact and link it. Populate identities from existing columns (`mobile`→phone, `email`→email, `social_hub.profile.id`→social_hub_profile, `pbx.extension`→phone, `remote_user_id`). Idempotent, chunked, runs in the `livecaller-php-api-1` container.

**Phase 2 — dedup pass.** Batch job groups visitors within each `account_id` by normalized phone / email / remote_user_id and merges their contacts via the §6 merge path (so it's logged/undoable). Run in dry-run first, eyeball a report of proposed merges, then execute.

**Phase 3 — live resolution.** Switch the three `GetVisitor` actions to call `ResolveContact`. From here, every new touchpoint correlates at write time.

**Phase 4 — suggestions UI.** Surface weak-signal suggested merges to agents; expose manual merge/split.

Backfill ordering matters: do the WhatsApp/Viber **phone extraction** (§5.3) *before* Phase 2 so the dedup pass can actually fuse PBX with WhatsApp.

---

## 9. Edge cases & policies

- **Shared phones / call centers / reception desks.** One number, many humans. Provide a per-contact "do not auto-merge / shared identifier" flag to stop runaway merges.
- **Number recycling.** A phone reassigned to a new person months later. Use `verified_at` recency and allow split; don't treat ancient identities as gospel.
- **Anonymous web visitors.** Each gets a throwaway contact; merge upward when they later identify (login, leave email/phone). Avoid merging two anonymous sessions on weak signals.
- **Cross-tenant collision.** Same number in two accounts → two contacts. Enforced by `account_id` in every unique key and query. Non-negotiable.
- **GDPR erasure.** Deleting a contact must cascade/anonymize its identities and detach visitors; the `contact_merges.snapshot` may also hold PII — include it in erasure.
- **Spoofed caller ID / forwarded numbers.** PBX `source` can be spoofed; keep confidence < 100 for unverified inbound caller-ID if abuse is a concern.
- **Performance.** `contact_identities(account_id, type, value_normalized)` unique index makes resolution an indexed point-lookup. List/search uses `contacts(account_id, primary_*)`. Merges touch few rows. No N+1 in the hot path.

---

## 10. Alternatives considered

| Option | Shape | Verdict |
|---|---|---|
| **A. `contact_id` on visitors only** (no identities table) | Add `contacts` + `visitors.contact_id`; match by re-reading visitor columns. | Simplest, but no normalized match index, can't hold multiple identifiers per visitor, races are hard, weak audit. **Rejected** — correlation quality suffers. |
| **B. contacts + contact_identities + visitors.contact_id** *(recommended)* | This document. | Normalized matching, multi-identifier, race-safe via unique key, reversible merges, additive. **Chosen.** |
| **C. Collapse visitor into contact** | Make `visitors` *be* contacts; treat each row as an identity. | Cleanest conceptually but hugely invasive — `conversation_members`, PBX endpoints, sessions, social hub all reference `visitor_id`. **Rejected** — too risky for the value. |
| **D. External CDP / identity service** | Offload to a dedicated customer-data platform. | Overkill, new infra, latency in the call path, tenant data egress. **Rejected** for now; Option B can feed one later. |

Option B is the sweet spot: it adds the person layer without disturbing the deeply-wired visitor layer, and its `contact_identities` table is exactly the normalized, indexed, tenant-scoped structure that makes deterministic correlation correct and fast.

---

## 11. Open questions

1. **Does Social Hub expose the WhatsApp/Viber phone number?** (Gates §5.3, the headline correlation.) Verify the webhook payload in `Actions/SocialHub/*`.
2. **Auto-merge aggressiveness.** Auto-merge on phone+email, or always require human confirmation for the *first* cross-channel link per contact?
3. **Does `remote_user_id` (host app's user id) already span channels?** If customers pass it on web *and* in PBX/social metadata, it may be the strongest seed identifier — worth auditing usage.
4. **Reporting impact.** Dashboards currently count visitors. Do we switch headline metrics to contacts, keep both, or add a "unique people" metric? Define before Phase 3.
5. **Agent UX for merge/split** — inline in the conversation panel vs. a dedicated contacts admin screen.

---

## 12. TL;DR

- Add **`contacts`** (the person) + **`contact_identities`** (normalized, tenant-unique match keys) + **`visitors.contact_id`**.
- **Normalize phones to E.164 and emails to lowercase** before matching — this is what makes PBX↔WhatsApp↔email correlation possible.
- Resolve contacts at the **three existing `GetVisitor` choke points**; auto-link on strong deterministic identifiers, **suggest** on weak ones.
- Make every merge **logged and reversible**; keep tenant isolation absolute.
- Roll out additively: schema → 1:1 backfill → dedup pass → live resolution → suggestions UI.
- The whole thing is the **visitor = channel endpoint, contact = person** split, done without touching the existing visitor/conversation wiring.
