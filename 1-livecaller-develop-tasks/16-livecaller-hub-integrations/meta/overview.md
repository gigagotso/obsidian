# Meta for Developers — Platform Overview

**TL;DR:** Everything you build against Facebook Pages, Messenger, Instagram, or WhatsApp runs through a **Meta App** created in the [App Dashboard](https://developers.facebook.com/apps/). An app has an **App ID** (public) and **App Secret** (confidential), starts in **Development mode** (role users only), and must pass **App Review** — and usually **Business Verification** — to get **Advanced Access** to permissions so it can serve real customers. Server integrations should be owned by a verified **Business Portfolio** (Business Manager) and use System User tokens (see `tokens.md`).

---

## 1. Meta Apps

A Meta App is the container for everything: credentials, permissions, products (Messenger, WhatsApp, Instagram, Webhooks…), webhook subscriptions, and review status.

- **App ID** — public numeric identifier, generated at creation, cannot be changed. Used in OAuth dialogs, SDK init, and endpoints that require app context.
- **App Secret** — confidential credential found in **App Dashboard → Settings → Basic**. Used to exchange tokens, sign `appsecret_proof`, validate webhook signatures, and mint app access tokens. Treat like a password; server-side only. Can be reset (which invalidates dependent artifacts).
- **Client Token** — a non-secret identifier (Settings → Advanced → Security) embedded in native apps for a small set of app-level APIs. Always used as `{app-id}|{client-token}`.

### App types / use cases
When creating an app you pick a type or use case ("App types determine which products, permissions, and features are available to your app"):

| Type | Intended for | Notes |
|---|---|---|
| **Business** | Apps managing business assets (Pages, Instagram professional accounts, WhatsApp, Ads). | Required for Facebook Login for Business and the messaging APIs. Business apps use *access levels* (Standard/Advanced) rather than the simple mode toggle as the main gate. |
| **Consumer** | Apps offering Facebook Login to consumers (`public_profile`, `email`). | In Live mode, Advanced Access features work for everyone but Standard Access features remain restricted to role users. |
| Other/None (legacy) | Gaming, workplace, etc. | Meta has been migrating app creation toward "use cases"; exact dashboard wording shifts over time. |

> Note: Meta has been gradually replacing the "app type" picker with a **use-case-based** creation flow. The Business-vs-Consumer distinction above still drives which permissions/products are available, but the exact dashboard UI may differ from older docs.

## 2. App Dashboard

The dashboard (`developers.facebook.com/apps/{app-id}`) is where you:

- **Settings → Basic:** App ID, App Secret, display name, namespace, category, icon, **Privacy Policy URL** (required to go live), Terms of Service URL, app domains, business connection (Verification section).
- **Settings → Advanced:** security options (Require App Secret, server IP allowlist, client token), age/geo restrictions, **upgrade default Graph API version** (used for unversioned calls).
- **App Review → Permissions and Features:** request Advanced Access per permission/feature.
- **Products:** add Messenger, WhatsApp, Webhooks, Facebook Login (for Business), Instagram, etc., each with its own settings panel.
- **Roles:** Administrators, Developers, Testers — these "role users" can use the app before review.

## 3. App Modes: Development vs Live

| | Development | Live |
|---|---|---|
| Who can use it | Only people with a role on the app (or in the business that claimed the app) | Anyone |
| Permissions requestable | Any permission at Standard *or* Advanced access — but **only from role users** | Only permissions with the needed access level; non-role users require **Advanced Access** (granted via App Review) |
| Visibility | Hidden from public search / App Center | Public |
| Data created | Hidden from public; becomes visible to all users when switched to Live | Visible |

- All new apps start in Development mode; switch with the toggle in the dashboard toolbar.
- Going live requires at minimum a valid **Privacy Policy URL**; meaningfully serving non-role users requires App Review approval for each permission.
- For **Business-type apps**, the operative gate is the per-permission **access level** (Standard vs Advanced) more than the mode toggle.

## 4. Access Levels: Standard vs Advanced

- **Standard Access** (default): the permission/feature only works for app users **who have a role on the app or in the business portfolio that has claimed the app**. Fine for development and for integrations that only touch assets your own business owns.
- **Advanced Access**: the permission/feature works for **all app users** (i.e., your customers). Granted per-permission via **Meta App Review**; since Feb 1, 2023 it also requires the app to be connected to a Business that completed **Business Verification**.

Rule of thumb for a messaging platform (you act on *other* businesses' Pages/WABAs): you need **Advanced Access** on every permission you request from customers.

## 5. App Review

App Review "enables Meta to verify that your app uses our products and APIs in an approved manner."

- **Required when** people *without* a role on the app/business will use it. Skippable only for purely internal tools.
- **You submit, per permission:** a description of usage, a **screencast** demonstrating the end-to-end flow using that permission, step-by-step **testing instructions**, and **test credentials** so reviewers can log in. If Meta cannot actually test the flow, the entire submission is rejected.
- **Outcome:** approved permissions get Advanced Access (usable by all users); everything else stays role-user-only.
- Timelines are not contractual; expect days to a few weeks in practice (not specified in docs — uncertain).
- Only request permissions the app demonstrably needs ("only select permissions that your app needs to function as intended").

## 6. Business Verification

- Confirms the legal identity of the business behind the app ("verify your identity as a business entity").
- **Required for:** Advanced Access requests; apps that let other businesses access data; Tech Provider / Solution Partner status on WhatsApp.
- **Where:** initiate the connection in **App Dashboard → Settings → Basic → Verification**, then complete verification in **Business Manager → Security Center** (only a Business Admin can complete it).
- **Documents:** legal name, address, phone/website confirmation — typically business registration/licence documents and domain or phone verification (exact list lives in the Business Help Center and varies by country).
- Individual developers are exempt only if the app exclusively serves role users.

## 7. Meta Business Manager / Business Portfolio

"Business Manager" is now branded **Business Portfolio** (managed in Meta Business Suite / `business.facebook.com`). It:

- Owns/claims business assets: Pages, Instagram accounts, WhatsApp Business Accounts (WABAs), ad accounts, pixels, catalogs — and can **claim apps**.
- Hosts **System Users** (machine identities) and their tokens — the backbone of server-to-server integrations.
- Is the entity that gets **Business Verification**.
- Allows asset sharing between businesses (e.g., a client grants your business partner access to their Page/WABA).

## 8. Typical app lifecycle for a messaging integration

1. Create a Business-type app under your (verified) business portfolio.
2. Build in Development mode using role users / test assets.
3. Complete Business Verification; connect the app to the business.
4. Submit App Review for Advanced Access on each needed permission (`pages_messaging`, `pages_manage_metadata`, `instagram_manage_messages`, `whatsapp_business_messaging`, …).
5. Switch to Live; onboard customers via Facebook Login for Business / Embedded Signup.
6. Maintain: respond to periodic **Data Use Checkup**, keep within Platform Terms, watch for version deprecations.

---

## Key Terms

- **Meta App**: The registered application container holding credentials, products, permissions, and review status.
- **App ID / App Secret**: Public identifier / confidential credential pair of an app.
- **Client Token**: Non-secret token embedded in native clients, used as `app_id|client_token`.
- **Development / Live mode**: Pre-launch (role users only) vs public operating state of an app.
- **Standard / Advanced Access**: Per-permission access level — role users only vs all users.
- **App Review**: Meta's per-permission approval process with screencast + test access.
- **Business Verification**: Legal-identity verification of the business portfolio backing the app.
- **Business Portfolio (Business Manager)**: Container owning business assets, system users, and verification status.
- **Role user**: Admin/Developer/Tester on the app, or someone in the business that claimed the app.

## Official Docs

- App Dashboard: https://developers.facebook.com/docs/development/create-an-app/app-dashboard
- App Modes: https://developers.facebook.com/docs/development/build-and-test/app-modes
- App Review: https://developers.facebook.com/docs/resp-plat-initiatives/individual-processes/app-review
- Access Levels / Features: https://developers.facebook.com/docs/features-reference
- Business Verification (dev docs): https://developers.facebook.com/docs/development/release/business-verification
- Business Verification (Business Help Center): https://www.facebook.com/business/help/2058515294227817
- Create an App: https://developers.facebook.com/docs/development/create-an-app
