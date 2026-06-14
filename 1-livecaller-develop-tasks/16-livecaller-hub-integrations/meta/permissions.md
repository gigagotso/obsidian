# Meta Permissions & Access Levels

**TL;DR:** Permissions are "granular, app user-granted Graph API authorization" — scopes a person (or business) grants your app during login. Each permission independently sits at **Standard Access** (works only for role users / your own business) or **Advanced Access** (works for everyone — required to serve customers). Advanced Access is earned per-permission through **App Review** and requires **Business Verification**. A token only carries the scopes the user actually granted — check `granular_scopes` via `/debug_token`, since users can decline individual permissions and grant them only for specific Pages/accounts.

---

## 1. How the model works

1. Your app requests scopes at login (`scope=pages_messaging,pages_show_list,...` or via a Login for Business configuration).
2. The user sees a consent screen and may **decline or partially grant** (e.g., grant access to only one of their Pages — "granular scopes").
3. Granted scopes are baked into the resulting token; API calls fail with code 10 / 200-series if a needed scope is missing or revoked.
4. Whether *non-role* users can grant a scope at all depends on the permission's **access level** in your app.

Also note: many endpoints additionally require the acting identity to have an asset-level role/task (e.g., the user must have the `MESSAGING` task on a Page — permission alone isn't enough).

## 2. Standard vs Advanced Access

| | Standard Access | Advanced Access |
|---|---|---|
| Who can the permission be requested from / work for | Only users **with a role on the app or in the business portfolio that claimed the app** | **All app users** |
| Default? | Yes — automatic for most permissions | No — per-permission grant |
| Requirements | None | **App Review** approval + **Business Verification** (mandatory since 2023-02-01) |
| Typical use | Development; internal tools touching only your own assets | SaaS platforms serving customers |

Some low-sensitivity permissions (`public_profile`, `email`, `pages_show_list` as a dependency) get Advanced Access automatically or with lighter review — check the current Permissions Reference per permission.

## 3. Key permissions for a messaging integration platform

| Permission | What it allows | Review needed for customer use |
|---|---|---|
| `public_profile` | Default public profile fields; auto-granted to all apps | No |
| `email` | Person's primary email (login/communication) | Yes (auto-Advanced for most apps; verify) |
| `pages_show_list` | "Show a person the list of Pages they manage" — needed to enumerate Pages and get Page tokens | Generally auto / dependency permission |
| `pages_messaging` | Send/receive Messenger messages as the Page (support, bookings, customer interactions) | Yes |
| `pages_manage_metadata` | **Subscribe the app to Page webhooks** and update Page settings | Yes |
| `pages_read_engagement` | Read Page content, follower data, profiles of people engaging with the Page | Yes |
| `instagram_basic` | Basic metadata of an Instagram professional account (ID, username) | Yes |
| `instagram_manage_messages` | Read & respond to Instagram Direct messages (CRM/helpdesk use) | Yes |
| `whatsapp_business_management` | Read/manage WhatsApp Business Account assets (phone numbers, templates, analytics) | Yes |
| `whatsapp_business_messaging` | Send/receive messages via Cloud API on behalf of a WABA | Yes |
| `business_management` | Read/write Business Manager API (asset management, claiming, system users) | Yes |

Typical bundles:
- **Messenger:** `pages_show_list`, `pages_messaging`, `pages_manage_metadata`, `pages_read_engagement`
- **Instagram DM:** the Messenger bundle + `instagram_basic`, `instagram_manage_messages`
- **WhatsApp (Tech Provider):** `whatsapp_business_management`, `whatsapp_business_messaging`, often `business_management`

## 4. Features vs permissions

Besides permissions there are **features** (e.g., "Human Agent" for Messenger 7-day response window, Page Public Content Access). Same access-level model: Standard = role users only, Advanced = everyone, granted via App Review. Requested on the same App Review → Permissions and Features page.

## 5. App Review tactics

- Request only what the product visibly uses ("only select permissions that your app needs to function as intended") — unused scopes cause rejection.
- Provide a working **screencast** per permission + reviewer **test credentials**; if reviewers can't reproduce the flow, the whole submission fails.
- Permissions can be submitted incrementally; approval is per-permission.
- After approval, periodic **Data Use Checkup** re-attestation is required to keep access.
- Losing/never getting Advanced Access doesn't break role-user usage — useful for staging.

## 6. Checking what you actually got

```http
GET /v25.0/debug_token?input_token={token}&access_token={app-id}|{app-secret}
```

Inspect `scopes` and `granular_scopes` (per-asset grants). Also `GET /me/permissions` with a user token lists granted/declined permissions; `DELETE /me/permissions/{permission}` revokes one (handy for testing re-consent).

---

## Key Terms

- **Permission (scope)**: User-granted authorization for a category of data/actions.
- **Granular scopes**: Per-asset narrowing of a grant (specific Pages/IG accounts/WABAs).
- **Feature**: Non-scope capability (e.g., Human Agent) with the same Standard/Advanced model.
- **Standard Access**: Permission works only for app/business role users.
- **Advanced Access**: Permission works for all users; needs App Review + Business Verification.
- **Data Use Checkup**: Annual attestation that each permission is still used per policy.
- **Page task**: Asset-level capability (`MESSAGING`, `MANAGE`, …) a user must hold on a Page, orthogonal to app permissions.

## Official Docs

- Permissions Reference: https://developers.facebook.com/docs/permissions
- Features Reference / access levels: https://developers.facebook.com/docs/features-reference
- App Review: https://developers.facebook.com/docs/resp-plat-initiatives/individual-processes/app-review
- Business Verification: https://developers.facebook.com/docs/development/release/business-verification
- Messenger Platform overview: https://developers.facebook.com/docs/messenger-platform
- WhatsApp Business Platform: https://developers.facebook.com/documentation/business-messaging/whatsapp/
