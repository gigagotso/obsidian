# Security Review: PR #651 + PR #661

## Scope
- Commits: `263cd02a9545f5399c1019c517da750aa7216ec2` (PR #651), `84e0ab4f67371e789ea55324d84e2becad511ad0` (PR #661)
- Areas touched: reports exports, conversation events, SocialHub ingestion, channel lines, AI assistant transfer state

## OWASP Top 10 Check Summary (2021)
- A01 Broken Access Control: **Potential issue found** (see Finding 1)
- A02 Cryptographic Failures: **No changes observed**
- A03 Injection: **No direct injection found** (input validation gap noted in Finding 2)
- A04 Insecure Design: **No direct issue observed**
- A05 Security Misconfiguration: **No direct issue observed**
- A06 Vulnerable and Outdated Components: **Not assessed** (dependency review out of scope)
- A07 Identification and Authentication Failures: **No direct issue observed**
- A08 Software and Data Integrity Failures: **No direct issue observed**
- A09 Security Logging and Monitoring Failures: **No direct issue observed**
- A10 Server-Side Request Forgery (SSRF): **No direct issue observed**

## Findings (ordered by severity)

### 1) Missing authorization gate for channel line listing
- Severity: **Medium**
- OWASP: **A01 Broken Access Control**
- Location: `app/Http/Requests/ChannelLine/IndexChannelLineRequest.php:11` (authorize always true)
- Detail: Any authenticated user can access `GET /v1/channel-lines` without any role/permission gating. If channel lines or identifiers are intended to be restricted (e.g., admin-only or channel managers), this leaks configuration data across all users of the account.
- Recommendation: Add a policy/permission check in `authorize()` (or controller) to restrict access based on role/permission. If all authenticated users are allowed, document this explicitly.

### 2) Channel line identifier is used without validation
- Severity: **Low**
- OWASP: **A03 Injection (input validation gap)**
- Location: `app/Actions/SocialHub/ReceiveMessage.php:34-82`
- Detail: `channel_line_identifier` is validated at the root key, but the handler reads `data.channel_line_identifier` and passes it to `GetConversation`. This bypasses validation and allows unvalidated input to drive the channel line lookup.
- Recommendation: Either validate `data.channel_line_identifier` in the rules or pass the validated root `channel_line_identifier`. Add length/format constraints if identifiers are expected to be numeric or snowflake-style.

### 3) ConversationEventService can emit malformed member attributes
- Severity: **Low**
- OWASP: **A04 Insecure Design (data integrity issue)**
- Location: `app/Services/ConversationEventService.php:105-131`
- Detail: `getMorphAlias()` returns `null` when the model class is not in the morph map, but `getExtraAttributes()` still appends `"_id"` as a key. This results in event rows with `member_id` set but `member_type_id` null and an ambiguous `_id` extra attribute, which can break reporting or downstream attribution.
- Recommendation: Guard against a null alias before appending, or ensure all relevant models are mapped in the morph map and in `conversation_member_types`.

### 4) Provider name not validated against known channels before channel-line lookup
- Severity: **Low**
- OWASP: **A04 Insecure Design**
- Location: `app/Actions/SocialHub/GetConversation.php:59-63`
- Detail: `provider.name` is only validated as a string. If it does not map to a known channel, `Channel::getIdByName()` returns null and the query becomes `where channel_id IS NULL`, which can lead to incorrect linkage if any records have a null `channel_id`.
- Recommendation: Validate `provider.name` against `channels.name` or short-circuit if no channel id exists before querying `ChannelLine`.

## Notes (positive security changes)
- Hourly export headers now sanitize Excel formula injection (prefixing `= + - @ \t \r`), which is a good mitigation against CSV/Excel formula injection in `app/Jobs/AbstractHourlyExport.php`.
- Hourly export endpoints are throttled (`throttle:10,60`) to reduce abuse.
