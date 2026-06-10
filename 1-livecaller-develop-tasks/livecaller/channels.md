# Operator Routing & Data Visibility Architecture

## Goals

1. **Distribute conversations** to operators based on channel lines and departments
2. **Restrict data visibility** - operators only see their assigned departments/channel lines
3. **Skill-based routing** using dynamic identifiers (tags, custom fields, visitor attributes)

---

## Architecture Options Comparison

### Option A: Department + Channel Line (Independent Assignments)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│                              OPERATOR                                            │
│                                  │                                               │
│                 ┌────────────────┴────────────────┐                              │
│                 │                                 │                              │
│                 ▼                                 ▼                              │
│         ┌──────────────┐                 ┌──────────────┐                        │
│         │  DEPARTMENTS │                 │CHANNEL LINES │                        │
│         │              │                 │              │                        │
│         │ ✓ Sales      │                 │ ✓ WhatsApp#1 │                        │
│         │ ✓ Support    │                 │ ✓ Widget #1  │                        │
│         │ ✗ VIP        │                 │ ✓ inbox@     │                        │
│         │ ✗ Billing    │                 │ ✗ Page 2     │                        │
│         └──────────────┘                 │ ✗ sales@     │                        │
│                                          └──────────────┘                        │
│                                                                                  │
│   VISIBILITY = Department ∩ Channel Line (must match BOTH)                      │
│                                                                                  │
│   Example: Operator sees conversations that are:                                │
│            (Sales OR Support) AND (WhatsApp#1 OR Widget#1 OR inbox@)            │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│ ✓ Maximum granularity - assign specific lines to specific operators            │
│ ✓ Flexible - same line can be in multiple departments                          │
│ ✓ Independent control over departments and channel lines                       │
│ ✗ Two separate assignments to manage per operator                              │
│ ✗ More complex admin UI                                                         │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Option B: Department with Channel Type Restriction

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│                              OPERATOR                                            │
│                                  │                                               │
│                                  ▼                                               │
│                         ┌──────────────┐                                        │
│                         │  DEPARTMENTS │                                        │
│                         └──────┬───────┘                                        │
│                                │                                                 │
│            ┌───────────────────┼───────────────────┐                            │
│            │                   │                   │                            │
│            ▼                   ▼                   ▼                            │
│     ┌────────────┐      ┌────────────┐      ┌────────────┐                      │
│     │   Sales    │      │  Support   │      │    VIP     │                      │
│     │ [email]    │      │   [all]    │      │ [web,pbx]  │                      │
│     └────────────┘      └────────────┘      └────────────┘                      │
│          │                   │                   │                              │
│          ▼                   ▼                   ▼                              │
│     Only email         All channels        Web + PBX only                       │
│     in Sales           in Support          in VIP                               │
│                                                                                  │
│   VISIBILITY = Department + Channel TYPE per Dept                               │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│ ✓ Simpler - only department assignment needed                                  │
│ ✓ Channel restrictions at department level                                     │
│ ✓ All operators in dept have same channel access                               │
│ ✗ Cannot restrict to specific LINE (e.g., specific WhatsApp number)           │
│ ✗ Less granular control per operator                                           │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Option C: Channel Line → Department Pre-mapping

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│              CHANNEL LINES ─────────────────> DEPARTMENTS                       │
│                    │                               │                             │
│                    │ belongs_to                    │                             │
│                    ▼                               ▼                             │
│             ┌────────────┐                  ┌────────────┐                       │
│             │ inbox@     │─────────────────>│   Email    │                       │
│             │ sales@     │─────────────────>│   Sales    │                       │
│             │ Widget #1  │─────────────────>│  Support   │                       │
│             │ WhatsApp#1 │─────────────────>│   Social   │                       │
│             └────────────┘                  └────────────┘                       │
│                                                   │                              │
│                                                   ▼                              │
│                                            ┌────────────┐                        │
│                                            │ OPERATORS  │                        │
│                                            │            │                        │
│                                            │ John→Email │                        │
│                                            │ Jane→Sales │                        │
│                                            │ Bob→Support│                        │
│                                            └────────────┘                        │
│                                                                                  │
│   VISIBILITY = Department (channel lines inherit from dept)                     │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│ ✓ Simplest to manage - only department assignment                              │
│ ✓ Channel lines auto-inherit from department                                   │
│ ✓ Clear ownership of channel lines                                             │
│ ✗ One channel line can ONLY belong to ONE department                           │
│ ✗ Cannot have same line served by multiple departments                         │
│ ✗ Less flexible routing                                                         │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture Comparison Table

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         COMPARISON MATRIX                                        │
├───────────────────────┬─────────────┬─────────────┬─────────────────────────────┤
│ Feature               │  Option A   │  Option B   │  Option C                   │
│                       │ Dept+Line   │ Dept+Type   │ Line→Dept                   │
├───────────────────────┼─────────────┼─────────────┼─────────────────────────────┤
│ Granularity           │    HIGH     │   MEDIUM    │    LOW                      │
├───────────────────────┼─────────────┼─────────────┼─────────────────────────────┤
│ Admin complexity      │    HIGH     │    LOW      │    LOW                      │
├───────────────────────┼─────────────┼─────────────┼─────────────────────────────┤
│ Per-operator channel  │     ✓       │     ✗       │     ✗                       │
│ line restriction      │             │             │                             │
├───────────────────────┼─────────────┼─────────────┼─────────────────────────────┤
│ Same line in multiple │     ✓       │     ✓       │     ✗                       │
│ departments           │             │             │                             │
├───────────────────────┼─────────────┼─────────────┼─────────────────────────────┤
│ Different operators   │     ✓       │     ✗       │     ✗                       │
│ see different lines   │             │             │                             │
│ in same dept          │             │             │                             │
├───────────────────────┼─────────────┼─────────────┼─────────────────────────────┤
│ New tables needed     │      1      │      0      │      0                      │
│                       │ (channel_   │ (modify     │ (add column                 │
│                       │ line_has_   │ queue_has_  │ to channel_                 │
│                       │ users)      │ users)      │ lines)                      │
├───────────────────────┼─────────────┼─────────────┼─────────────────────────────┤
│ Your diagram shows    │     ✓       │  Partial    │     ✗                       │
│ operator-3[email]     │ Can do this │ Need per-op │ Cannot do                   │
│                       │             │ restriction │                             │
└───────────────────────┴─────────────┴─────────────┴─────────────────────────────┘
```

---

## Alternative Approaches (Thinking Outside the Box)

### Alternative 1: Unified Assignment Table (SIMPLER)

Instead of **two separate tables**, use **ONE table** that combines department + channel line:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   CURRENT APPROACH (Option A): Two Tables                                       │
│                                                                                  │
│   ┌─────────────────────┐     ┌─────────────────────┐                           │
│   │  queue_has_users    │     │channel_line_has_users│                          │
│   ├─────────────────────┤     ├─────────────────────┤                           │
│   │ John → Sales        │     │ John → WhatsApp#1   │                           │
│   │ John → Support      │     │ John → Widget#1     │                           │
│   │ Jane → Sales        │     │ Jane → sales@       │                           │
│   └─────────────────────┘     └─────────────────────┘                           │
│                                                                                  │
│   Problem: To know what John can see, you need to JOIN both tables             │
│            and compute the intersection. Complex!                               │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ALTERNATIVE: One Unified Table                                                │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                      operator_assignments                                │   │
│   ├─────────┬─────────────┬─────────────────┬─────────────┬──────────────────┤  │
│   │ user_id │ department  │ channel_line_id │ auto_assign │ priority         │  │
│   ├─────────┼─────────────┼─────────────────┼─────────────┼──────────────────┤  │
│   │ John    │ Sales       │ NULL            │ true        │ 1                │  │
│   │         │             │ (= ALL lines)   │             │                  │  │
│   ├─────────┼─────────────┼─────────────────┼─────────────┼──────────────────┤  │
│   │ John    │ Support     │ WhatsApp#1      │ true        │ 2                │  │
│   │         │             │ (specific line) │             │                  │  │
│   ├─────────┼─────────────┼─────────────────┼─────────────┼──────────────────┤  │
│   │ John    │ Support     │ Widget#1        │ true        │ 1                │  │
│   │         │             │ (specific line) │             │                  │  │
│   ├─────────┼─────────────┼─────────────────┼─────────────┼──────────────────┤  │
│   │ Jane    │ Sales       │ sales@          │ true        │ 1                │  │
│   │         │             │ (email only)    │             │                  │  │
│   └─────────┴─────────────┴─────────────────┴─────────────┴──────────────────┘  │
│                                                                                  │
│   ✓ ONE table instead of TWO                                                    │
│   ✓ Each row = explicit assignment: "John handles WhatsApp in Support"         │
│   ✓ NULL channel_line = ALL lines in that department                           │
│   ✓ No complex intersection logic needed                                       │
│   ✓ Easy to query: WHERE department = X AND (channel_line = Y OR NULL)        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Alternative 2: Team-Based Model (SIMPLEST for Admins)

Create reusable "Teams" that bundle access:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                           teams                                          │   │
│   ├──────┬───────────────────┬─────────────┬─────────────────────────────────┤  │
│   │  id  │ name              │ departments │ channel_lines                   │  │
│   ├──────┼───────────────────┼─────────────┼─────────────────────────────────┤  │
│   │  1   │ Sales Email Team  │ [Sales]     │ [sales@, inbox@]                │  │
│   │  2   │ Support All       │ [Support]   │ NULL (all lines)                │  │
│   │  3   │ VIP Specialists   │ [VIP]       │ [Widget#1, WhatsApp#1]          │  │
│   │  4   │ Social Media Team │ [Social]    │ [Page1, Page2, WhatsApp#1]      │  │
│   └──────┴───────────────────┴─────────────┴─────────────────────────────────┘  │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                       team_has_users                                     │   │
│   ├─────────┬─────────┐                                                      │  │
│   │ team_id │ user_id │                                                      │  │
│   ├─────────┼─────────┤                                                      │  │
│   │    1    │  Jane   │  ← Jane is in Sales Email Team                      │  │
│   │    2    │  John   │  ← John is in Support All                           │  │
│   │    3    │  John   │  ← John is ALSO in VIP Specialists                  │  │
│   │    3    │  Alice  │  ← Alice is in VIP Specialists                      │  │
│   │    4    │  Bob    │  ← Bob is in Social Media Team                      │  │
│   └─────────┴─────────┘                                                      │  │
│                                                                                  │
│   ✓ VERY simple for admins - just assign to teams                              │
│   ✓ Reusable - same team for multiple operators                                │
│   ✓ Self-documenting - "Sales Email Team" is clear                             │
│   ✓ Easy onboarding - new hire joins existing teams                            │
│   ✗ Less flexible - harder to have per-operator exceptions                     │
│                                                                                  │
│   Operator visibility = UNION of all their teams' access                       │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Alternative 3: Channel Line as Primary Entity

Flip the model - channel lines are the main entity, departments are just labels:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   Instead of:  Operator → Department → (intersect) → Channel Line              │
│                                                                                  │
│   Do this:     Operator → Channel Line → (channel line has department label)   │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                        channel_lines                                     │   │
│   ├──────┬──────────────┬─────────────┬──────────────────────────────────────┤  │
│   │  id  │ name         │ department  │ other config...                      │  │
│   ├──────┼──────────────┼─────────────┼──────────────────────────────────────┤  │
│   │  1   │ sales@       │ Sales       │                                      │  │
│   │  2   │ support@     │ Support     │                                      │  │
│   │  3   │ Widget#1     │ Support     │                                      │  │
│   │  4   │ WhatsApp#1   │ Social      │                                      │  │
│   │  5   │ VIP-Widget   │ VIP         │                                      │  │
│   └──────┴──────────────┴─────────────┴──────────────────────────────────────┘  │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                    channel_line_has_users                                │   │
│   ├─────────────────┬─────────┬─────────────┐                                │  │
│   │ channel_line_id │ user_id │ auto_assign │                                │  │
│   ├─────────────────┼─────────┼─────────────┤                                │  │
│   │        1        │  Jane   │ true        │  ← Jane handles sales@        │  │
│   │        2        │  John   │ true        │  ← John handles support@      │  │
│   │        3        │  John   │ true        │  ← John handles Widget#1      │  │
│   │        4        │  Bob    │ true        │  ← Bob handles WhatsApp#1     │  │
│   └─────────────────┴─────────┴─────────────┘                                │  │
│                                                                                  │
│   Routing: Find conversation's channel_line → find operators for that line    │
│   Visibility: Operators see conversations from their assigned channel lines    │
│   Department: Just a label on channel_line for grouping/reporting              │
│                                                                                  │
│   ✓ Simpler mental model - "who handles this line?"                            │
│   ✓ No intersection logic                                                       │
│   ✓ Department is metadata, not a filter                                       │
│   ✗ Same line can't be in multiple departments                                 │
│   ✗ Department-based routing needs extra logic                                 │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Alternative 4: Inbox Model (Like Help Desk Software)

Think of it like shared email inboxes:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   Each "INBOX" = a combination of channel line + department                     │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                           inboxes                                        │   │
│   ├──────┬─────────────────────┬─────────────┬─────────────────┬─────────────┤  │
│   │  id  │ name                │ department  │ channel_line_id │ settings    │  │
│   ├──────┼─────────────────────┼─────────────┼─────────────────┼─────────────┤  │
│   │  1   │ Sales Email         │ Sales       │ sales@          │ {...}       │  │
│   │  2   │ Support WhatsApp    │ Support     │ WhatsApp#1      │ {...}       │  │
│   │  3   │ Support Widget      │ Support     │ Widget#1        │ {...}       │  │
│   │  4   │ VIP All Channels    │ VIP         │ NULL (all)      │ {...}       │  │
│   └──────┴─────────────────────┴─────────────┴─────────────────┴─────────────┘  │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                       inbox_has_users                                    │   │
│   ├──────────┬─────────┬─────────────┐                                       │  │
│   │ inbox_id │ user_id │ auto_assign │                                       │  │
│   ├──────────┼─────────┼─────────────┤                                       │  │
│   │    1     │  Jane   │ true        │  ← Jane in Sales Email inbox         │  │
│   │    2     │  John   │ true        │  ← John in Support WhatsApp inbox    │  │
│   │    3     │  John   │ true        │  ← John also in Support Widget       │  │
│   │    4     │  Alice  │ true        │  ← Alice in VIP (all channels)       │  │
│   └──────────┴─────────┴─────────────┘                                       │  │
│                                                                                  │
│   ✓ Familiar concept (like Zendesk, Freshdesk, etc.)                           │
│   ✓ Each inbox is a clear, nameable thing                                      │
│   ✓ Easy to understand: "John is in these inboxes"                             │
│   ✓ Flexible: create any combination of dept + channel                         │
│   ✗ More entities to manage (inboxes + assignments)                            │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Comparison of All Approaches

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         ALL APPROACHES COMPARISON                                │
├───────────────────────┬────────────┬────────────┬────────────┬──────────────────┤
│                       │ Option A   │ Alt 1      │ Alt 2      │ Alt 3/4          │
│                       │ Two Tables │ Unified    │ Teams      │ Channel/Inbox    │
├───────────────────────┼────────────┼────────────┼────────────┼──────────────────┤
│ Tables needed         │     2      │     1      │     2      │     1-2          │
├───────────────────────┼────────────┼────────────┼────────────┼──────────────────┤
│ Query complexity      │   HIGH     │    LOW     │   MEDIUM   │     LOW          │
│ (to find visibility)  │ (intersect)│ (simple)   │ (union)    │   (direct)       │
├───────────────────────┼────────────┼────────────┼────────────┼──────────────────┤
│ Admin UX              │  COMPLEX   │   GOOD     │  SIMPLE    │    GOOD          │
│                       │ 2 screens  │ 1 screen   │ team-based │  1 screen        │
├───────────────────────┼────────────┼────────────┼────────────┼──────────────────┤
│ "John handles         │    ✓       │     ✓      │     ✓      │      ✓           │
│  WhatsApp in Support" │ 2 entries  │  1 entry   │ via team   │   1 entry        │
├───────────────────────┼────────────┼────────────┼────────────┼──────────────────┤
│ "John handles ALL     │    ✓       │     ✓      │     ✓      │ Need multiple    │
│  channels in Support" │ 1+0 entries│ 1 NULL row │ team config│   entries        │
├───────────────────────┼────────────┼────────────┼────────────┼──────────────────┤
│ Per-operator          │    ✓       │     ✓      │     ✗      │      ✓           │
│ exceptions            │            │            │ (harder)   │                  │
├───────────────────────┼────────────┼────────────┼────────────┼──────────────────┤
│ Onboarding new hire   │  COMPLEX   │   MEDIUM   │   SIMPLE   │    MEDIUM        │
│                       │ 2 configs  │  1 config  │ add to team│   1 config       │
└───────────────────────┴────────────┴────────────┴────────────┴──────────────────┘
```

---

### Alternative 5: Polymorphic Skills Table (Your Suggestion)

One table with polymorphic reference to channel, channel_line, OR department:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                         user_skills                                      │   │
│   ├─────────┬────────────────┬──────────────┬──────────┬─────────────────────┤  │
│   │ user_id │ skillable_type │ skillable_id │ priority │ auto_assign         │  │
│   ├─────────┼────────────────┼──────────────┼──────────┼─────────────────────┤  │
│   │ John    │ department     │ Sales        │ 1        │ true                │  │
│   │ John    │ department     │ Support      │ 1        │ true                │  │
│   │ John    │ channel_line   │ 4 (WhatsApp) │ 2        │ true                │  │
│   │ John    │ channel        │ 1 (email)    │ 1        │ true                │  │
│   ├─────────┼────────────────┼──────────────┼──────────┼─────────────────────┤  │
│   │ Jane    │ department     │ Sales        │ 1        │ true                │  │
│   │ Jane    │ channel_line   │ 10 (sales@)  │ 1        │ true                │  │
│   ├─────────┼────────────────┼──────────────┼──────────┼─────────────────────┤  │
│   │ Alice   │ department     │ VIP          │ 1        │ true                │  │
│   │ Alice   │ channel        │ 3 (website)  │ 1        │ true                │  │
│   ├─────────┼────────────────┼──────────────┼──────────┼─────────────────────┤  │
│   │ Bob     │ channel        │ 6 (pbx)      │ 1        │ true                │  │
│   └─────────┴────────────────┴──────────────┴──────────┴─────────────────────┘  │
│                                                                                  │
│   skillable_type can be:                                                         │
│   • "department" → skillable_id = queue name                                    │
│   • "channel"    → skillable_id = channel id (email, whatsapp, pbx, etc.)      │
│   • "channel_line" → skillable_id = specific channel line id                   │
│   • "tag"        → skillable_id = tag id (VIP, Technical, etc.)                │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**But there's a question: How do skills combine?**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   OPTION A: UNION (Additive) - Each skill ADDS access                           │
│   ─────────────────────────────────────────────────────                         │
│                                                                                  │
│   John has:                                                                      │
│   • department: Sales                                                            │
│   • department: Support                                                          │
│   • channel_line: WhatsApp#1                                                    │
│                                                                                  │
│   John can see conversations that match ANY:                                    │
│   • Department = Sales (any channel)         ← 50 conversations                 │
│   • Department = Support (any channel)       ← 40 conversations                 │
│   • Channel Line = WhatsApp#1 (any dept)     ← 30 conversations                 │
│                                                                                  │
│   Total visible = UNION = ~100 conversations (with overlap)                     │
│                                                                                  │
│   ✓ Simple to understand                                                        │
│   ✓ Flexible - can grant access to anything                                    │
│   ✗ Might give TOO MUCH access (WhatsApp#1 in ALL departments)                 │
│   ✗ Hard to say "only WhatsApp in Support"                                     │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   OPTION B: INTERSECTION (Restrictive) - Must match ALL skill types            │
│   ─────────────────────────────────────────────────────────────────             │
│                                                                                  │
│   John has:                                                                      │
│   • department: Sales                                                            │
│   • department: Support                                                          │
│   • channel_line: WhatsApp#1                                                    │
│                                                                                  │
│   John can see conversations that match:                                        │
│   • (Department = Sales OR Support)                                             │
│     AND                                                                          │
│   • (Channel Line = WhatsApp#1)                                                 │
│                                                                                  │
│   Total visible = only WhatsApp#1 in Sales or Support                          │
│                                                                                  │
│   ✓ More restrictive / secure                                                   │
│   ✗ Confusing: if John has dept:Sales + channel_line:WhatsApp#1,               │
│     can he see Sales email? (NO - needs channel_line for email too)            │
│   ✗ Need to add skills for each channel line                                   │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   OPTION C: SCOPED SKILLS (Best of both)                                        │
│   ──────────────────────────────────────                                        │
│                                                                                  │
│   Add a "scope" to each skill - it can be scoped to another skill              │
│                                                                                  │
│   ┌─────────┬────────────────┬──────────────┬────────────────┬───────────────┐ │
│   │ user_id │ skillable_type │ skillable_id │ scope_type     │ scope_id      │ │
│   ├─────────┼────────────────┼──────────────┼────────────────┼───────────────┤ │
│   │ John    │ department     │ Sales        │ NULL           │ NULL          │ │
│   │         │                │              │ (global)       │               │ │
│   ├─────────┼────────────────┼──────────────┼────────────────┼───────────────┤ │
│   │ John    │ channel_line   │ WhatsApp#1   │ department     │ Support       │ │
│   │         │                │              │ (only in       │               │ │
│   │         │                │              │  Support)      │               │ │
│   ├─────────┼────────────────┼──────────────┼────────────────┼───────────────┤ │
│   │ Jane    │ channel        │ email        │ department     │ Sales         │ │
│   │         │                │              │ (email only    │               │ │
│   │         │                │              │  in Sales)     │               │ │
│   └─────────┴────────────────┴──────────────┴────────────────┴───────────────┘ │
│                                                                                  │
│   Reading:                                                                       │
│   • John: Sales (all channels) + WhatsApp#1 (only in Support)                  │
│   • Jane: Email (only in Sales)                                                 │
│                                                                                  │
│   ✓ Most flexible                                                               │
│   ✓ Can express any combination                                                │
│   ✗ More complex table structure                                               │
│   ✗ Harder to query                                                            │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### Polymorphic vs Unified: Comparison

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   UNIFIED TABLE (Alt 1)              POLYMORPHIC (Alt 5)                        │
│   ─────────────────────              ──────────────────────                     │
│                                                                                  │
│   operator_assignments               user_skills                                │
│   ┌────────┬──────┬───────┐         ┌────────┬──────────┬──────────┐           │
│   │user_id │dept  │line_id│         │user_id │skill_type│skill_id  │           │
│   ├────────┼──────┼───────┤         ├────────┼──────────┼──────────┤           │
│   │John    │Sales │NULL   │         │John    │department│Sales     │           │
│   │John    │Supp  │WA#1   │         │John    │department│Support   │           │
│   └────────┴──────┴───────┘         │John    │chan_line │WA#1      │           │
│                                      └────────┴──────────┴──────────┘           │
│                                                                                  │
│   ✓ Explicit: "John handles         ✓ More flexible: can add                   │
│     WhatsApp in Support"              channel-type skills                       │
│                                                                                  │
│   ✓ One row = one scope             ✓ Same table for all types                 │
│                                                                                  │
│   ✗ Only dept + line combo          ✗ Need to decide how skills                │
│                                        combine (union/intersection)             │
│                                                                                  │
│   ✗ Can't say "all email"           ✓ Can say "all email" with                 │
│     without listing lines              channel type skill                       │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### ★ HYBRID: Polymorphic with Scope (MOST FLEXIBLE)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   TABLE: user_skills                                                             │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │ id │user│ skillable    │ skillable │ scope        │ scope   │priority│   │  │
│   │    │_id │ _type        │ _id       │ _type        │ _id     │        │   │  │
│   ├────┼────┼──────────────┼───────────┼──────────────┼─────────┼────────┼───┤  │
│   │ 1  │John│ department   │ Sales     │ NULL         │ NULL    │ 1      │   │  │
│   │    │    │              │           │ (all chans)  │         │        │   │  │
│   ├────┼────┼──────────────┼───────────┼──────────────┼─────────┼────────┼───┤  │
│   │ 2  │John│ department   │ Support   │ channel_line │ WA#1    │ 2      │   │  │
│   │    │    │              │           │ (only WA#1)  │         │        │   │  │
│   ├────┼────┼──────────────┼───────────┼──────────────┼─────────┼────────┼───┤  │
│   │ 3  │Jane│ department   │ Sales     │ channel      │ email   │ 1      │   │  │
│   │    │    │              │           │ (only email) │         │        │   │  │
│   ├────┼────┼──────────────┼───────────┼──────────────┼─────────┼────────┼───┤  │
│   │ 4  │Bob │ channel      │ pbx       │ NULL         │ NULL    │ 1      │   │  │
│   │    │    │              │           │ (all depts)  │         │        │   │  │
│   ├────┼────┼──────────────┼───────────┼──────────────┼─────────┼────────┼───┤  │
│   │ 5  │Alice│ tag         │ VIP       │ NULL         │ NULL    │ 10     │   │  │
│   │    │    │ (skill-based │           │ (priority    │         │        │   │  │
│   │    │    │  routing)    │           │  routing)    │         │        │   │  │
│   └────┴────┴──────────────┴───────────┴──────────────┴─────────┴────────┴───┘  │
│                                                                                  │
│   READING THE TABLE:                                                             │
│                                                                                  │
│   Row 1: John → Sales department, ALL channels                                  │
│   Row 2: John → Support department, ONLY WhatsApp#1                             │
│   Row 3: Jane → Sales department, ONLY email channel type                       │
│   Row 4: Bob  → ALL PBX channels, ANY department                                │
│   Row 5: Alice → Has VIP skill (for routing priority, not visibility)          │
│                                                                                  │
│   SKILLABLE TYPES:                                                               │
│   • department   → access to a department                                       │
│   • channel      → access to a channel type (email, pbx, whatsapp)             │
│   • channel_line → access to a specific channel line                            │
│   • tag          → skill for routing (VIP, Technical, Spanish)                  │
│                                                                                  │
│   SCOPE TYPES:                                                                   │
│   • NULL         → no restriction (applies everywhere)                          │
│   • department   → only in this department                                      │
│   • channel      → only for this channel type                                   │
│   • channel_line → only for this channel line                                   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### Examples with Polymorphic + Scope

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   FROM YOUR DIAGRAM:                                                             │
│                                                                                  │
│   operator-1 [social-account dept, all social channels]                         │
│   ┌──────────────┬───────────┬───────────┬────────────┐                         │
│   │ skillable    │skillable  │ scope     │ scope      │                         │
│   │ _type        │_id        │ _type     │ _id        │                         │
│   ├──────────────┼───────────┼───────────┼────────────┤                         │
│   │ department   │ social    │ NULL      │ NULL       │  ← all channels        │
│   └──────────────┴───────────┴───────────┴────────────┘                         │
│                                                                                  │
│   operator-3 [Sales dept, email only]                                           │
│   ┌──────────────┬───────────┬───────────┬────────────┐                         │
│   │ department   │ Sales     │ channel   │ email      │  ← email only in Sales │
│   └──────────────┴───────────┴───────────┴────────────┘                         │
│                                                                                  │
│   operator-4 [Support dept, email + messenger]                                  │
│   ┌──────────────┬───────────┬───────────┬────────────┐                         │
│   │ department   │ Support   │ channel   │ email      │  ← email in Support    │
│   │ department   │ Support   │ channel   │ messenger  │  ← messenger in Support│
│   └──────────────┴───────────┴───────────┴────────────┘                         │
│                                                                                  │
│   operator-7 [Viber, VIP (web only), dept-110000] + VIP skill                   │
│   ┌──────────────┬───────────┬───────────┬────────────┐                         │
│   │ department   │ Viber     │ NULL      │ NULL       │  ← all channels        │
│   │ department   │ VIP       │ channel   │ website    │  ← website only in VIP │
│   │ department   │ 110000    │ NULL      │ NULL       │  ← all channels        │
│   │ tag          │ VIP       │ NULL      │ NULL       │  ← VIP skill (routing) │
│   └──────────────┴───────────┴───────────┴────────────┘                         │
│                                                                                  │
│   operator-8 [VIP dept] + VIP skill                                             │
│   ┌──────────────┬───────────┬───────────┬────────────┐                         │
│   │ department   │ VIP       │ NULL      │ NULL       │  ← all channels        │
│   │ tag          │ VIP       │ NULL      │ NULL       │  ← VIP skill (routing) │
│   └──────────────┴───────────┴───────────┴────────────┘                         │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### Updated Comparison

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         FINAL COMPARISON                                         │
├───────────────────────┬────────────┬────────────┬────────────────────────────────┤
│                       │ Alt 1      │ Alt 5      │ Alt 5 + Scope                  │
│                       │ Unified    │ Polymorphic│ (Your suggestion)              │
├───────────────────────┼────────────┼────────────┼────────────────────────────────┤
│ Tables                │ 1          │ 1          │ 1                              │
├───────────────────────┼────────────┼────────────┼────────────────────────────────┤
│ Columns               │ 4          │ 4          │ 6                              │
├───────────────────────┼────────────┼────────────┼────────────────────────────────┤
│ "John: Sales, all"    │ 1 row      │ 1 row      │ 1 row                          │
│                       │ NULL line  │ dept skill │ dept, no scope                 │
├───────────────────────┼────────────┼────────────┼────────────────────────────────┤
│ "John: Support, WA"   │ 1 row      │ 2 rows     │ 1 row                          │
│                       │ line=WA    │ dept+line  │ dept, scope=line               │
├───────────────────────┼────────────┼────────────┼────────────────────────────────┤
│ "Jane: Sales, email"  │ Need each  │ 2 rows     │ 1 row                          │
│ (all email lines)     │ email line │ dept+chan  │ dept, scope=channel            │
├───────────────────────┼────────────┼────────────┼────────────────────────────────┤
│ "Bob: all PBX"        │ ✗ Can't    │ 1 row      │ 1 row                          │
│ (any department)      │            │ chan skill │ channel, no scope              │
├───────────────────────┼────────────┼────────────┼────────────────────────────────┤
│ VIP skill routing     │ Separate   │ Same table │ Same table                     │
│                       │ table      │ tag skill  │ tag skill                      │
├───────────────────────┼────────────┼────────────┼────────────────────────────────┤
│ Query complexity      │ LOW        │ MEDIUM     │ MEDIUM-HIGH                    │
├───────────────────────┼────────────┼────────────┼────────────────────────────────┤
│ Flexibility           │ MEDIUM     │ HIGH       │ HIGHEST                        │
└───────────────────────┴────────────┴────────────┴────────────────────────────────┘
```

---

## ★ NEW RECOMMENDATION: Alternative 1 (Unified Assignment)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   ★ RECOMMENDED: UNIFIED ASSIGNMENT TABLE                                       │
│                                                                                  │
│   Table: operator_assignments                                                    │
│   ┌─────────┬─────────────┬─────────────────┬─────────────┬──────────────────┐  │
│   │ user_id │ queue_name  │ channel_line_id │ auto_assign │ priority         │  │
│   ├─────────┼─────────────┼─────────────────┼─────────────┼──────────────────┤  │
│   │ John    │ Sales       │ NULL            │ true        │ 1                │  │
│   │ John    │ Support     │ 4 (WhatsApp#1)  │ true        │ 2                │  │
│   │ John    │ Support     │ 7 (Widget#1)    │ true        │ 1                │  │
│   │ Jane    │ Sales       │ 10 (sales@)     │ true        │ 1                │  │
│   │ Alice   │ VIP         │ NULL            │ true        │ 1                │  │
│   └─────────┴─────────────┴─────────────────┴─────────────┴──────────────────┘  │
│                                                                                  │
│   WHY THIS IS BETTER:                                                            │
│                                                                                  │
│   1. ONE table instead of TWO                                                   │
│   2. Each row is self-explanatory: "John handles WhatsApp in Support"          │
│   3. NULL channel_line = ALL lines in that department                          │
│   4. Simple query: WHERE queue_name = ? AND (channel_line_id = ? OR IS NULL)   │
│   5. Matches your diagram exactly - one entry per operator-dept-channel combo  │
│   6. Easy admin UI - one screen to manage assignments                          │
│                                                                                  │
│   READING YOUR DIAGRAM:                                                          │
│                                                                                  │
│   • operator-1 in "social account": 1 row with NULL (all social lines)         │
│   • operator-3 in Sales [email]: 1 row with specific email line                │
│   • operator-4 in Support [email,msg]: 2 rows, one per line                    │
│   • operator-7 in Viber, VIP, 110000: 3 rows (one per dept)                    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Visitor → Operator Flow (Complete Journey)

### How a Conversation Reaches an Operator

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   VISITOR initiates contact through a CHANNEL                                   │
│                                                                                  │
│   ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐         │
│   │ WhatsApp   │    │  Website   │    │   Email    │    │    PBX     │         │
│   │  Visitor   │    │  Visitor   │    │  Visitor   │    │   Caller   │         │
│   └─────┬──────┘    └─────┬──────┘    └─────┬──────┘    └─────┬──────┘         │
│         │                 │                 │                 │                 │
│         ▼                 ▼                 ▼                 ▼                 │
│   ╔═════════════════════════════════════════════════════════════════════════╗  │
│   ║                        STEP 1: CHANNEL LINE                              ║  │
│   ║                                                                          ║  │
│   ║   The specific entry point where visitor contacted                       ║  │
│   ║                                                                          ║  │
│   ║   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  ║  │
│   ║   │ WhatsApp     │  │ Widget #1    │  │ support@     │  │ 32 12 00   │  ║  │
│   ║   │ +995599111   │  │ livecaller.ge│  │ company.com  │  │ 00         │  ║  │
│   ║   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘  ║  │
│   ║          │                 │                 │                │         ║  │
│   ╚══════════╪═════════════════╪═════════════════╪════════════════╪═════════╝  │
│              │                 │                 │                │            │
│              ▼                 ▼                 ▼                ▼            │
│   ╔═════════════════════════════════════════════════════════════════════════╗  │
│   ║                        STEP 2: DEPARTMENT SELECTION                      ║  │
│   ║                                                                          ║  │
│   ║   Department is determined by PRIORITY CHAIN (not mutually exclusive):  ║  │
│   ║                                                                          ║  │
│   ║   ┌─────────────────────────────────────────────────────────────────┐   ║  │
│   ║   │ PRIORITY 1: Routing Rules (highest priority, most dynamic)      │   ║  │
│   ║   │                                                                  │   ║  │
│   ║   │ IF visitor.tag = VIP           → route to "VIP" dept            │   ║  │
│   ║   │ IF custom_field.priority=high  → route to "Priority" dept       │   ║  │
│   ║   │ IF visitor.language = spanish  → route to "Spanish" dept        │   ║  │
│   ║   │                                                                  │   ║  │
│   ║   │ If rule matches → USE THIS DEPARTMENT, skip rest               │   ║  │
│   ║   └──────────────────────────────┬──────────────────────────────────┘   ║  │
│   ║                                  │ no rule matched                      ║  │
│   ║                                  ▼                                      ║  │
│   ║   ┌─────────────────────────────────────────────────────────────────┐   ║  │
│   ║   │ PRIORITY 2: Visitor Selection (if widget configured to show)   │   ║  │
│   ║   │                                                                  │   ║  │
│   ║   │ Widget shows: "Please select: [Sales] [Support] [Billing]"      │   ║  │
│   ║   │                                                                  │   ║  │
│   ║   │ If visitor selected a department → USE THIS DEPARTMENT          │   ║  │
│   ║   └──────────────────────────────┬──────────────────────────────────┘   ║  │
│   ║                                  │ no selection / not shown             ║  │
│   ║                                  ▼                                      ║  │
│   ║   ┌─────────────────────────────────────────────────────────────────┐   ║  │
│   ║   │ PRIORITY 3: Channel Line Default                                │   ║  │
│   ║   │                                                                  │   ║  │
│   ║   │ channel_line.extra_attributes.default_department = "Sales"     │   ║  │
│   ║   │                                                                  │   ║  │
│   ║   │ If channel line has default → USE THIS DEPARTMENT              │   ║  │
│   ║   └──────────────────────────────┬──────────────────────────────────┘   ║  │
│   ║                                  │ no default configured                ║  │
│   ║                                  ▼                                      ║  │
│   ║   ┌─────────────────────────────────────────────────────────────────┐   ║  │
│   ║   │ PRIORITY 4: Account Default Department (fallback)              │   ║  │
│   ║   │                                                                  │   ║  │
│   ║   │ account.settings.default_department = "Support"                │   ║  │
│   ║   │                                                                  │   ║  │
│   ║   │ Always exists as final fallback                                │   ║  │
│   ║   └─────────────────────────────────────────────────────────────────┘   ║  │
│   ║                                                                          ║  │
│   ║   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  ║  │
│   ║   │    Sales     │  │   Support    │  │    VIP       │  │  Billing   │  ║  │
│   ║   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘  ║  │
│   ║          │                 │                 │                │         ║  │
│   ╚══════════╪═════════════════╪═════════════════╪════════════════╪═════════╝  │
│              │                 │                 │                │            │
│              └────────────────┬┴─────────────────┴────────────────┘            │
│                               │                                                 │
│                               ▼                                                 │
│   ╔═════════════════════════════════════════════════════════════════════════╗  │
│   ║                        STEP 3: FIND MATCHING OPERATORS                   ║  │
│   ║                                                                          ║  │
│   ║   Filter 1: Who is in this DEPARTMENT?                                   ║  │
│   ║             SELECT * FROM queue_has_users WHERE queue_name = 'Support'   ║  │
│   ║             Result: [John, Jane, Bob, Alice, Tom]                        ║  │
│   ║                                                                          ║  │
│   ║   Filter 2: Who handles this CHANNEL LINE?                               ║  │
│   ║             SELECT * FROM channel_line_has_users                         ║  │
│   ║             WHERE channel_line_id = 'Widget #1'                          ║  │
│   ║             Result: [John, Alice, Tom]                                   ║  │
│   ║                                                                          ║  │
│   ║   Filter 3: INTERSECTION (must be in BOTH)                               ║  │
│   ║             [John, Jane, Bob, Alice, Tom] ∩ [John, Alice, Tom]           ║  │
│   ║             Result: [John, Alice, Tom]                                   ║  │
│   ║                                                                          ║  │
│   ╚══════════════════════════════════════════════════════════════════════════╝  │
│                               │                                                 │
│                               ▼                                                 │
│   ╔═════════════════════════════════════════════════════════════════════════╗  │
│   ║                        STEP 4: SKILL-BASED ROUTING                       ║  │
│   ║                                                                          ║  │
│   ║   Does visitor have tags/attributes that match operator skills?          ║  │
│   ║                                                                          ║  │
│   ║   Visitor has: [VIP tag]                                                 ║  │
│   ║                                                                          ║  │
│   ║   Operator skills:                                                       ║  │
│   ║   • John:  VIP (priority 10)  ← Match!                                  ║  │
│   ║   • Alice: VIP (priority 15)  ← Match! Higher priority                  ║  │
│   ║   • Tom:   (no VIP skill)     ← No match, lower priority                ║  │
│   ║                                                                          ║  │
│   ║   Ranked: [Alice:15, John:10, Tom:0]                                    ║  │
│   ║                                                                          ║  │
│   ╚══════════════════════════════════════════════════════════════════════════╝  │
│                               │                                                 │
│                               ▼                                                 │
│   ╔═════════════════════════════════════════════════════════════════════════╗  │
│   ║                        STEP 5: AVAILABILITY CHECK                        ║  │
│   ║                                                                          ║  │
│   ║   For each operator (in priority order):                                 ║  │
│   ║                                                                          ║  │
│   ║   1. Alice (priority 15):                                                ║  │
│   ║      • Online: ✓                                                        ║  │
│   ║      • Away:   ✓ (is away!)  → SKIP                                     ║  │
│   ║                                                                          ║  │
│   ║   2. John (priority 10):                                                 ║  │
│   ║      • Online: ✓                                                        ║  │
│   ║      • Away:   ✗                                                        ║  │
│   ║      • Current chats: 3/5 (has capacity)  → AVAILABLE ✓                 ║  │
│   ║                                                                          ║  │
│   ╚══════════════════════════════════════════════════════════════════════════╝  │
│                               │                                                 │
│                               ▼                                                 │
│   ╔═════════════════════════════════════════════════════════════════════════╗  │
│   ║                        STEP 6: ASSIGN TO OPERATOR                        ║  │
│   ║                                                                          ║  │
│   ║                         ┌─────────────┐                                  ║  │
│   ║                         │    JOHN     │                                  ║  │
│   ║                         │  ◉ Online   │                                  ║  │
│   ║                         │  Chats: 4/5 │                                  ║  │
│   ║                         └─────────────┘                                  ║  │
│   ║                                                                          ║  │
│   ║   Conversation assigned to John                                          ║  │
│   ║   John receives notification                                             ║  │
│   ║   Conversation appears in John's inbox                                   ║  │
│   ║                                                                          ║  │
│   ╚══════════════════════════════════════════════════════════════════════════╝  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Flow Diagram (Simplified)

```
    VISITOR
       │
       │ contacts via
       ▼
  ┌─────────────┐
  │CHANNEL LINE │  ← e.g., WhatsApp +995599111, Widget #1, support@email.com
  │             │
  │ Determines: │
  │ • Channel   │
  │ • Entry     │
  │   point     │
  └──────┬──────┘
         │
         │ selects (or pre-configured)
         ▼
  ┌─────────────┐
  │ DEPARTMENT  │  ← e.g., Sales, Support, VIP
  │             │
  │ Contains:   │
  │ • Operators │
  │ • Settings  │
  │ • Queue     │
  └──────┬──────┘
         │
         │ filters by (Department ∩ Channel Line)
         ▼
  ┌─────────────┐
  │ CANDIDATE   │  ← Operators who:
  │ OPERATORS   │    • Are in this department
  │             │    • Handle this channel line
  │ [John,      │
  │  Alice,     │
  │  Tom]       │
  └──────┬──────┘
         │
         │ ranks by (skill match + availability)
         ▼
  ┌─────────────┐
  │  SELECTED   │
  │  OPERATOR   │  ← First available with best skill match
  │             │
  │   [John]    │
  └─────────────┘
```

### Example Flow: WhatsApp VIP Customer

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  1. VISITOR                                                                      │
│     └── VIP customer sends WhatsApp message                                     │
│         └── Visitor has tag: [VIP]                                              │
│                                                                                  │
│  2. CHANNEL LINE                                                                 │
│     └── WhatsApp #1 (+995599111111)                                             │
│         └── channel_line_id = 4                                                 │
│                                                                                  │
│  3. DEPARTMENT                                                                   │
│     └── Widget configured to show: [Sales] [Support]                            │
│     └── Visitor selected: "Support"                                             │
│         └── department_id = "dept-support"                                      │
│                                                                                  │
│  4. FIND OPERATORS                                                               │
│     └── In Support dept: [John, Jane, Bob, Mike, Sarah]                         │
│     └── Handle WhatsApp #1: [John, Mike, Sarah]                                 │
│     └── Intersection: [John, Mike, Sarah]                                       │
│                                                                                  │
│  5. SKILL MATCHING                                                               │
│     └── Visitor tag VIP requires VIP skill                                      │
│     └── John: has VIP skill (priority 10) ✓                                     │
│     └── Mike: no VIP skill                                                      │
│     └── Sarah: has VIP skill (priority 8) ✓                                     │
│     └── Ranked: [John:10, Sarah:8, Mike:0]                                      │
│                                                                                  │
│  6. AVAILABILITY                                                                 │
│     └── John: Online ✓, Not Away ✓, Capacity 3/5 ✓                             │
│                                                                                  │
│  7. RESULT                                                                       │
│     └── Conversation assigned to JOHN                                           │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Example Flow: Email to Specific Inbox

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  1. VISITOR                                                                      │
│     └── Customer sends email to sales@company.com                               │
│         └── No special tags                                                     │
│                                                                                  │
│  2. CHANNEL LINE                                                                 │
│     └── sales@company.com (email)                                               │
│         └── channel_line_id = 10                                                │
│         └── default_department = "Sales" (pre-configured)                       │
│                                                                                  │
│  3. DEPARTMENT                                                                   │
│     └── Auto-selected: "Sales" (from channel line config)                       │
│         └── department_id = "dept-sales"                                        │
│                                                                                  │
│  4. FIND OPERATORS                                                               │
│     └── In Sales dept: [Jane, Bob, Tom]                                         │
│     └── Handle sales@ email: [Jane, Bob]                                        │
│     └── Intersection: [Jane, Bob]                                               │
│                                                                                  │
│  5. SKILL MATCHING                                                               │
│     └── No special skills required                                              │
│     └── Both operators equal priority                                           │
│                                                                                  │
│  6. AVAILABILITY & LOAD BALANCING                                                │
│     └── Jane: 2 active chats                                                    │
│     └── Bob: 4 active chats                                                     │
│     └── Select operator with fewer chats: JANE                                  │
│                                                                                  │
│  7. RESULT                                                                       │
│     └── Conversation assigned to JANE                                           │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### What Happens If No Operators Match?

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  Scenario: WhatsApp message, but no operators handle that line in the dept     │
│                                                                                  │
│  1. VISITOR contacts WhatsApp #2                                                 │
│  2. Department: Support                                                          │
│  3. Operators in Support who handle WhatsApp #2: [ NONE ]                       │
│                                                                                  │
│  FALLBACK OPTIONS:                                                               │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ Option A: Use operators with NO channel line restrictions               │    │
│  │                                                                          │    │
│  │ If operator has no entries in channel_line_has_users,                   │    │
│  │ they can handle ALL channel lines in their departments                  │    │
│  │                                                                          │    │
│  │ → Find operators in Support with no line restrictions                   │    │
│  │ → Route to them                                                         │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ Option B: Queue the conversation                                         │    │
│  │                                                                          │    │
│  │ → Add to department queue                                                │    │
│  │ → Send "serve request" notification to all Support operators            │    │
│  │ → First operator to accept gets the conversation                        │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ Option C: Route to chatbot/AI first                                      │    │
│  │                                                                          │    │
│  │ → Start with chatbot                                                    │    │
│  │ → Chatbot collects info, transfers to human when available             │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ Option D: Fallback to different department                               │    │
│  │                                                                          │    │
│  │ → Department has fallback_department configured                         │    │
│  │ → Route to fallback department instead                                  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Entity Model

### Core Entities

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                    ACCOUNT                                       │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ CHANNELS (Types)                                                          │   │
│  │                                                                           │   │
│  │  messenger   whatsapp   website   email   viber   pbx   telegram   sms   │   │
│  │      │          │          │        │       │      │        │       │    │   │
│  └──────┼──────────┼──────────┼────────┼───────┼──────┼────────┼───────┼────┘   │
│         │          │          │        │       │      │        │       │        │
│         ▼          ▼          ▼        ▼       ▼      ▼        ▼       ▼        │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ CHANNEL LINES (Specific Instances)                                        │   │
│  │                                                                           │   │
│  │  Page1    WhatsApp   Widget1   inbox@     Viber    32120000  Bot1   SMS   │   │
│  │  Page2    Number1    Widget2   support@   577555   32110000  Bot2   +995  │   │
│  │  Page3    Number2              sales@                                     │   │
│  │                                                                           │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ DEPARTMENTS (Queues)                                                      │   │
│  │                                                                           │   │
│  │  Sales    Support    VIP    Technical    Billing    Social    PBX        │   │
│  │                                                                           │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ OPERATORS (Users)                                                         │   │
│  │                                                                           │   │
│  │  John    Jane    Bob    Alice    Tom    Sarah    Mike    Anna            │   │
│  │                                                                           │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ SKILLS / TAGS (Dynamic Routing Criteria)                                  │   │
│  │                                                                           │   │
│  │  VIP    Premium    Technical    Billing    Spanish    Enterprise         │   │
│  │                                                                           │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Assignment Relationships

### 2.1 Operator → Department

**Table: `queue_has_users`** (existing)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        OPERATOR → DEPARTMENT                                     │
│                                                                                  │
│                         ┌─────────────┐                                         │
│                         │   OPERATOR  │                                         │
│                         │    (John)   │                                         │
│                         └──────┬──────┘                                         │
│                                │                                                 │
│                    ┌───────────┼───────────┐                                    │
│                    │           │           │                                    │
│                    ▼           ▼           ▼                                    │
│              ┌─────────┐ ┌─────────┐ ┌─────────┐                                │
│              │ Sales   │ │ Support │ │   VIP   │                                │
│              └─────────┘ └─────────┘ └─────────┘                                │
│                                                                                  │
│   One operator can belong to MULTIPLE departments                               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

Table: queue_has_users
┌──────────────┬─────────┬─────────────┐
│ queue_name   │ user_id │ auto_assign │
├──────────────┼─────────┼─────────────┤
│ dept-sales   │ 1 (John)│ true        │
│ dept-support │ 1 (John)│ true        │
│ dept-vip     │ 1 (John)│ false       │  ← Can view but not auto-assigned
└──────────────┴─────────┴─────────────┘
```

### 2.2 Operator → Channel Line

**Table: `channel_line_has_users`** (NEW)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        OPERATOR → CHANNEL LINE                                   │
│                                                                                  │
│                         ┌─────────────┐                                         │
│                         │   OPERATOR  │                                         │
│                         │    (John)   │                                         │
│                         └──────┬──────┘                                         │
│                                │                                                 │
│            ┌───────────┬───────┼───────┬───────────┐                            │
│            │           │       │       │           │                            │
│            ▼           ▼       ▼       ▼           ▼                            │
│       ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐                   │
│       │ Page 1 │ │WhatsApp│ │Widget 1│ │ inbox@ │ │ Viber  │                   │
│       │  (msg) │ │  #1    │ │ (web)  │ │(email) │ │        │                   │
│       └────────┘ └────────┘ └────────┘ └────────┘ └────────┘                   │
│                                                                                  │
│   One operator can handle MULTIPLE channel lines                                │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

Table: channel_line_has_users
┌─────────────────┬─────────┬─────────┬──────────┬────────────────┐
│ channel_line_id │ user_id │ enabled │ priority │ max_concurrent │
├─────────────────┼─────────┼─────────┼──────────┼────────────────┤
│ 1 (Page 1)      │ 1 (John)│ true    │ 1        │ 5              │
│ 4 (WhatsApp #1) │ 1 (John)│ true    │ 2        │ 3              │
│ 7 (Widget 1)    │ 1 (John)│ true    │ 1        │ 10             │
│ 9 (inbox@)      │ 1 (John)│ true    │ 1        │ 10             │
│ 11 (Viber)      │ 1 (John)│ true    │ 1        │ 5              │
└─────────────────┴─────────┴─────────┴──────────┴────────────────┘
```

### 2.3 Operator → Skills

**Table: `operator_skills`** (NEW)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          OPERATOR → SKILLS                                       │
│                                                                                  │
│                         ┌─────────────┐                                         │
│                         │   OPERATOR  │                                         │
│                         │    (John)   │                                         │
│                         └──────┬──────┘                                         │
│                                │                                                 │
│                    ┌───────────┼───────────┐                                    │
│                    │           │           │                                    │
│                    ▼           ▼           ▼                                    │
│              ┌─────────┐ ┌─────────┐ ┌─────────┐                                │
│              │   VIP   │ │Technical│ │ Spanish │                                │
│              │priority:│ │priority:│ │priority:│                                │
│              │   10    │ │    5    │ │    3    │                                │
│              └─────────┘ └─────────┘ └─────────┘                                │
│                                                                                  │
│   One operator can have MULTIPLE skills with different priorities               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

Table: operator_skills
┌─────────┬────────┬──────────┬────────────┐
│ user_id │ tag_id │ priority │ queue_name │
├─────────┼────────┼──────────┼────────────┤
│ 1 (John)│ 1 (VIP)│ 10       │ dept-vip   │  ← VIP skill in VIP dept only
│ 1 (John)│ 2(Tech)│ 5        │ NULL       │  ← Technical in all departments
│ 1 (John)│ 3(Span)│ 3        │ NULL       │  ← Spanish in all departments
└─────────┴────────┴──────────┴────────────┘
```

---

## Part 3: Complete Assignment Matrix

### Visual: Operator Assignment Grid

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    ASSIGNMENT MATRIX                                                 │
│                                                                                                      │
│  LEGEND:  ✓ = Assigned    · = Not Assigned    [skill] = Has skill for that context                 │
│                                                                                                      │
├─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                      │
│                         CHANNEL LINES                                                                │
│           ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────┐                   │
│           │ Page 1  │ Page 2  │WhatsApp1│WhatsApp2│ Widget1 │ inbox@  │ sales@  │                   │
│           │messenger│messenger│         │         │ website │  email  │  email  │                   │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┤                   │
│ OPERATORS │         │         │         │         │         │         │         │                   │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┤                   │
│ John      │    ✓    │    ·    │    ✓    │    ·    │    ✓    │    ✓    │    ·    │                   │
│ Skills:   │         │         │         │         │         │         │         │                   │
│ VIP,Tech  │         │         │         │         │         │         │         │                   │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┤                   │
│ Jane      │    ·    │    ·    │    ·    │    ·    │    ·    │    ·    │    ✓    │                   │
│ Skills:   │         │         │         │         │         │         │         │                   │
│ (none)    │         │         │         │         │         │         │         │                   │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┤                   │
│ Bob       │    ·    │    ·    │    ·    │    ·    │    ·    │    ✓    │    ✓    │                   │
│ Skills:   │         │         │         │         │         │         │         │                   │
│ Billing   │         │         │         │         │         │         │         │                   │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┤                   │
│ Alice     │    ✓    │    ✓    │    ✓    │    ✓    │    ·    │    ·    │    ·    │                   │
│ Skills:   │         │         │         │         │         │         │         │                   │
│ VIP       │         │         │         │         │         │         │         │                   │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┤                   │
│ Tom       │    ·    │    ·    │    ·    │    ·    │    ✓    │    ·    │    ·    │                   │
│ Skills:   │         │         │         │         │         │         │         │                   │
│ VIP,Prem  │         │         │         │         │         │         │         │                   │
└───────────┴─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘                   │
                                                                                                       │
├─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                      │
│                         DEPARTMENTS                                                                  │
│           ┌─────────┬─────────┬─────────┬─────────┬─────────┐                                       │
│           │  Sales  │ Support │   VIP   │Technical│ Billing │                                       │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┤                                       │
│ OPERATORS │         │         │         │         │         │                                       │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┤                                       │
│ John      │    ✓    │    ✓    │    ✓    │    ✓    │    ·    │                                       │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┤                                       │
│ Jane      │    ✓    │    ·    │    ·    │    ·    │    ·    │                                       │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┤                                       │
│ Bob       │    ·    │    ✓    │    ·    │    ·    │    ✓    │                                       │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┤                                       │
│ Alice     │    ·    │    ·    │    ✓    │    ·    │    ·    │                                       │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┤                                       │
│ Tom       │    ·    │    ·    │    ✓    │    ·    │    ·    │                                       │
└───────────┴─────────┴─────────┴─────────┴─────────┴─────────┘                                       │
                                                                                                       │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Visibility Rules

### How Visibility Works

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           VISIBILITY FORMULA                                     │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                                                                          │   │
│   │   VISIBLE CONVERSATIONS = Conversations WHERE:                           │   │
│   │                                                                          │   │
│   │       department_id  ∈  operator's departments                          │   │
│   │                      AND                                                 │   │
│   │       channel_line_id  ∈  operator's channel lines                      │   │
│   │                      (OR no channel line restrictions)                   │   │
│   │                                                                          │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Visibility Scenarios

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ SCENARIO 1: John's Visibility                                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  John's Assignments:                                                             │
│  ├── Departments: [Sales, Support, VIP, Technical]                              │
│  └── Channel Lines: [Page 1, WhatsApp1, Widget1, inbox@]                        │
│                                                                                  │
│  ═══════════════════════════════════════════════════════════════════════════    │
│                                                                                  │
│  ALL CONVERSATIONS (Example: 200 total)                                          │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │        │ Page 1 │ Page 2 │WhatsApp1│WhatsApp2│Widget1│inbox@ │sales@ │    │  │
│  ├────────┼────────┼────────┼─────────┼─────────┼───────┼───────┼───────┤    │  │
│  │ Sales  │   5    │   5    │    8    │    7    │  10   │   3   │  15   │ 53 │  │
│  │Support │   8    │   4    │   12    │    6    │  15   │  10   │   2   │ 57 │  │
│  │ VIP    │   3    │   2    │    5    │    3    │   8   │   2   │   0   │ 23 │  │
│  │Tech    │   2    │   3    │    4    │    2    │   6   │   5   │   0   │ 22 │  │
│  │Billing │   0    │   0    │    2    │    3    │   4   │  20   │  16   │ 45 │  │
│  └────────┴────────┴────────┴─────────┴─────────┴───────┴───────┴───────┘    │  │
│                                                                                  │
│  JOHN CAN SEE (highlighted cells only):                                          │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │        │ Page 1 │ Page 2 │WhatsApp1│WhatsApp2│Widget1│inbox@ │sales@ │    │  │
│  ├────────┼────────┼────────┼─────────┼─────────┼───────┼───────┼───────┤    │  │
│  │ Sales  │  [5]   │   ·    │   [8]   │    ·    │ [10]  │  [3]  │   ·   │ 26 │  │
│  │Support │  [8]   │   ·    │  [12]   │    ·    │ [15]  │ [10]  │   ·   │ 45 │  │
│  │ VIP    │  [3]   │   ·    │   [5]   │    ·    │  [8]  │  [2]  │   ·   │ 18 │  │
│  │Tech    │  [2]   │   ·    │   [4]   │    ·    │  [6]  │  [5]  │   ·   │ 17 │  │
│  │Billing │   ·    │   ·    │    ·    │    ·    │   ·   │   ·   │   ·   │  0 │  │
│  └────────┴────────┴────────┴─────────┴─────────┴───────┴───────┴───────┘    │  │
│                                                                                  │
│  John sees 106 of 200 conversations (53%)                                        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│ SCENARIO 2: Jane's Visibility (Single Channel Line)                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  Jane's Assignments:                                                             │
│  ├── Departments: [Sales]                                                        │
│  └── Channel Lines: [sales@]                                                     │
│                                                                                  │
│  JANE CAN SEE:                                                                   │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │        │ Page 1 │ Page 2 │WhatsApp1│WhatsApp2│Widget1│inbox@ │sales@ │    │  │
│  ├────────┼────────┼────────┼─────────┼─────────┼───────┼───────┼───────┤    │  │
│  │ Sales  │   ·    │   ·    │    ·    │    ·    │   ·   │   ·   │ [15]  │ 15 │  │
│  │Support │   ·    │   ·    │    ·    │    ·    │   ·   │   ·   │   ·   │  0 │  │
│  │ VIP    │   ·    │   ·    │    ·    │    ·    │   ·   │   ·   │   ·   │  0 │  │
│  │Tech    │   ·    │   ·    │    ·    │    ·    │   ·   │   ·   │   ·   │  0 │  │
│  │Billing │   ·    │   ·    │    ·    │    ·    │   ·   │   ·   │   ·   │  0 │  │
│  └────────┴────────┴────────┴─────────┴─────────┴───────┴───────┴───────┘    │  │
│                                                                                  │
│  Jane sees 15 of 200 conversations (7.5%)                                        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│ SCENARIO 3: Alice's Visibility (No Channel Line Restriction)                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  Alice's Assignments:                                                            │
│  ├── Departments: [VIP]                                                          │
│  └── Channel Lines: [NONE - NO RESTRICTIONS]                                    │
│                                                                                  │
│  ALICE CAN SEE (ALL channel lines in VIP department):                           │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │        │ Page 1 │ Page 2 │WhatsApp1│WhatsApp2│Widget1│inbox@ │sales@ │    │  │
│  ├────────┼────────┼────────┼─────────┼─────────┼───────┼───────┼───────┤    │  │
│  │ Sales  │   ·    │   ·    │    ·    │    ·    │   ·   │   ·   │   ·   │  0 │  │
│  │Support │   ·    │   ·    │    ·    │    ·    │   ·   │   ·   │   ·   │  0 │  │
│  │ VIP    │  [3]   │  [2]   │   [5]   │   [3]   │  [8]  │  [2]  │  [0]  │ 23 │  │
│  │Tech    │   ·    │   ·    │    ·    │    ·    │   ·   │   ·   │   ·   │  0 │  │
│  │Billing │   ·    │   ·    │    ·    │    ·    │   ·   │   ·   │   ·   │  0 │  │
│  └────────┴────────┴────────┴─────────┴─────────┴───────┴───────┴───────┘    │  │
│                                                                                  │
│  Alice sees 23 of 200 conversations (11.5%)                                      │
│  NOTE: No channel line restriction = ALL channels in assigned departments       │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Skill-Based Routing

### Dynamic Routing Criteria Sources

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    SKILL-BASED ROUTING SOURCES                                   │
│                                                                                  │
│  A conversation can have routing criteria from MULTIPLE sources:                 │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                                                                          │    │
│  │  1. VISITOR TAGS                                                         │    │
│  │     ├── VIP                    (manually added or from CRM)              │    │
│  │     ├── Premium                                                          │    │
│  │     └── Enterprise                                                       │    │
│  │                                                                          │    │
│  │  2. WIDGET CUSTOM FIELDS                                                 │    │
│  │     ├── priority = "high"                                                │    │
│  │     ├── language = "spanish"                                             │    │
│  │     ├── product = "enterprise"                                           │    │
│  │     └── customer_type = "vip"                                            │    │
│  │                                                                          │    │
│  │  3. CONVERSATION TAGS                                                    │    │
│  │     ├── Urgent                                                           │    │
│  │     ├── Escalated                                                        │    │
│  │     └── Follow-up                                                        │    │
│  │                                                                          │    │
│  │  4. VISITOR ATTRIBUTES                                                   │    │
│  │     ├── is_authenticated = true                                          │    │
│  │     ├── total_purchases > 1000                                           │    │
│  │     ├── country = "Georgia"                                              │    │
│  │     └── account_tier = "gold"                                            │    │
│  │                                                                          │    │
│  │  5. CHANNEL LINE CONTEXT                                                 │    │
│  │     ├── From VIP email inbox                                             │    │
│  │     ├── From enterprise widget                                           │    │
│  │     └── From specific phone number                                       │    │
│  │                                                                          │    │
│  │  6. EXTERNAL DATA (via API/Webhook)                                      │    │
│  │     ├── CRM customer tier                                                │    │
│  │     ├── Support ticket history                                           │    │
│  │     └── Subscription status                                              │    │
│  │                                                                          │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Routing Rules Table

**Table: `routing_rules`** (NEW - for complex dynamic routing)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         ROUTING RULES                                            │
│                                                                                  │
│  Instead of just tag matching, we can have flexible routing rules:              │
│                                                                                  │
│  Table: routing_rules                                                            │
│  ┌────┬────────────────────────────────────────────────────┬──────────┬────────┐│
│  │ id │ condition                                          │ action   │priority││
│  ├────┼────────────────────────────────────────────────────┼──────────┼────────┤│
│  │ 1  │ visitor.tags CONTAINS 'VIP'                        │ skill:VIP│ 100    ││
│  │ 2  │ widget.custom_fields.priority = 'high'             │ skill:VIP│ 90     ││
│  │ 3  │ visitor.extra_attributes.is_vip = true             │ skill:VIP│ 90     ││
│  │ 4  │ widget.custom_fields.language = 'spanish'          │skill:Span│ 80     ││
│  │ 5  │ visitor.country = 'DE'                             │skill:Germ│ 70     ││
│  │ 6  │ channel_line.extra_attributes.tier = 'enterprise'  │skill:Ent │ 85     ││
│  │ 7  │ visitor.extra_attributes.total_purchases > 1000    │ skill:VIP│ 75     ││
│  │ 8  │ conversation.tags CONTAINS 'Escalated'             │skill:Seni│ 95     ││
│  └────┴────────────────────────────────────────────────────┴──────────┴────────┘│
│                                                                                  │
│  When a conversation arrives, ALL matching rules are evaluated.                  │
│  The resulting skills are collected and used to find matching operators.         │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Simple Approach: operator_skills Table

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      SIMPLER APPROACH: operator_skills                           │
│                                                                                  │
│  Map operators to skills/tags directly:                                          │
│                                                                                  │
│  Table: operator_skills                                                          │
│  ┌─────────┬────────┬──────────┬────────────┐                                   │
│  │ user_id │ tag_id │ priority │ queue_name │                                   │
│  ├─────────┼────────┼──────────┼────────────┤                                   │
│  │ John    │ VIP    │ 10       │ dept-vip   │  ← John handles VIP in VIP dept  │
│  │ John    │ VIP    │ 5        │ dept-sales │  ← John also VIP in Sales (lower)│
│  │ John    │ Tech   │ 8        │ NULL       │  ← John handles Tech in ALL depts│
│  │ Alice   │ VIP    │ 15       │ NULL       │  ← Alice is VIP specialist (high)│
│  │ Bob     │Billing │ 10       │ NULL       │  ← Bob handles Billing anywhere  │
│  └─────────┴────────┴──────────┴────────────┘                                   │
│                                                                                  │
│  Skill matching:                                                                 │
│  1. Visitor has tags [VIP, Technical]                                           │
│  2. Find operators with those skills in the conversation's department           │
│  3. Order by priority DESC                                                       │
│  4. Route to highest priority available operator                                 │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Routing Flow (Complete)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        COMPLETE ROUTING FLOW                                     │
│                                                                                  │
│                    NEW CONVERSATION ARRIVES                                      │
│                            │                                                     │
│                            │  Metadata:                                          │
│                            │  • channel_line_id: WhatsApp #1                    │
│                            │  • department_id: Support                          │
│                            │  • visitor_tags: [VIP]                             │
│                            │  • custom_fields: {language: "spanish"}            │
│                            │                                                     │
│                            ▼                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ STEP 1: DEPARTMENT FILTER                                                │    │
│  │                                                                          │    │
│  │ SELECT operators FROM queue_has_users                                    │    │
│  │ WHERE queue_name = 'Support' AND auto_assign = true                     │    │
│  │                                                                          │    │
│  │ Result: [John, Bob, Mike, Sarah]                                        │    │
│  └──────────────────────────────────┬──────────────────────────────────────┘    │
│                                     │                                            │
│                                     ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ STEP 2: CHANNEL LINE FILTER                                              │    │
│  │                                                                          │    │
│  │ Filter operators who handle WhatsApp #1 channel line                    │    │
│  │ (OR have no channel line restrictions)                                  │    │
│  │                                                                          │    │
│  │ • John: has WhatsApp #1 ✓                                               │    │
│  │ • Bob: only has inbox@, sales@ ✗                                        │    │
│  │ • Mike: no restrictions (null) ✓                                        │    │
│  │ • Sarah: has WhatsApp #1 ✓                                              │    │
│  │                                                                          │    │
│  │ Result: [John, Mike, Sarah]                                             │    │
│  └──────────────────────────────────┬──────────────────────────────────────┘    │
│                                     │                                            │
│                                     ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ STEP 3: COLLECT ROUTING CRITERIA                                         │    │
│  │                                                                          │    │
│  │ From visitor tags:     [VIP]                                            │    │
│  │ From custom fields:    language=spanish → [Spanish]                     │    │
│  │ From visitor attrs:    (none)                                           │    │
│  │                                                                          │    │
│  │ Combined criteria: [VIP, Spanish]                                       │    │
│  └──────────────────────────────────┬──────────────────────────────────────┘    │
│                                     │                                            │
│                                     ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ STEP 4: SKILL MATCHING                                                   │    │
│  │                                                                          │    │
│  │ Find operators with matching skills:                                    │    │
│  │                                                                          │    │
│  │ • John: has VIP (priority 10), Spanish (priority 3) → score: 13        │    │
│  │ • Mike: has Spanish (priority 5) → score: 5                             │    │
│  │ • Sarah: has VIP (priority 8) → score: 8                                │    │
│  │                                                                          │    │
│  │ Ordered by skill score: [John:13, Sarah:8, Mike:5]                      │    │
│  └──────────────────────────────────┬──────────────────────────────────────┘    │
│                                     │                                            │
│                                     ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ STEP 5: AVAILABILITY CHECK                                               │    │
│  │                                                                          │    │
│  │ For each operator (in skill priority order):                            │    │
│  │                                                                          │    │
│  │ • John:  Online: ✓  Away: ✗  Capacity: 4/5 → AVAILABLE                  │    │
│  │ • Sarah: Online: ✓  Away: ✓  (is away)    → SKIP                        │    │
│  │ • Mike:  Online: ✓  Away: ✗  Capacity: 5/5 → FULL                       │    │
│  │                                                                          │    │
│  │ First available: John                                                    │    │
│  └──────────────────────────────────┬──────────────────────────────────────┘    │
│                                     │                                            │
│                                     ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ STEP 6: ROUTE OR QUEUE                                                   │    │
│  │                                                                          │    │
│  │ ✓ Available operator found: ROUTE TO JOHN                               │    │
│  │                                                                          │    │
│  │ OR if no one available:                                                 │    │
│  │ • Add to department queue                                               │    │
│  │ • Send serve request to matching operators                              │    │
│  │ • First operator to accept gets the conversation                        │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: All Routing Scenarios

### Scenario Matrix

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         ALL ROUTING SCENARIOS                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│ Scenario 1: SIMPLE - No skills, single channel                                  │
│ ────────────────────────────────────────────────                                │
│ • Conversation from: Widget #1                                                  │
│ • Department: Support                                                           │
│ • Visitor tags: (none)                                                          │
│ • Custom fields: (none)                                                         │
│                                                                                  │
│ Routing: Find operators in Support + assigned to Widget #1 + available          │
│ Result: Route to first available                                                │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│ Scenario 2: VIP TAG                                                             │
│ ────────────────────                                                            │
│ • Conversation from: WhatsApp #1                                                │
│ • Department: Sales                                                             │
│ • Visitor tags: [VIP]                                                           │
│ • Custom fields: (none)                                                         │
│                                                                                  │
│ Routing:                                                                        │
│   1. Filter: Sales dept + WhatsApp #1 line                                     │
│   2. Among those, prefer operators with VIP skill                              │
│   3. Order by VIP skill priority                                               │
│                                                                                  │
│ Result: Route to highest-priority VIP specialist, or any available if none     │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│ Scenario 3: CUSTOM FIELD (Widget priority=high)                                 │
│ ───────────────────────────────────────────────                                 │
│ • Conversation from: Widget #2                                                  │
│ • Department: Support                                                           │
│ • Visitor tags: (none)                                                          │
│ • Custom fields: {priority: "high", product: "enterprise"}                      │
│                                                                                  │
│ Routing:                                                                        │
│   1. Filter: Support dept + Widget #2 line                                     │
│   2. Evaluate routing rules:                                                   │
│      - priority=high → maps to VIP skill                                       │
│      - product=enterprise → maps to Enterprise skill                           │
│   3. Prefer operators with [VIP, Enterprise] skills                            │
│                                                                                  │
│ Result: Route to operator with matching skills                                  │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│ Scenario 4: LANGUAGE ROUTING                                                    │
│ ────────────────────────────                                                    │
│ • Conversation from: Page 1 (Messenger)                                         │
│ • Department: Support                                                           │
│ • Visitor tags: (none)                                                          │
│ • Custom fields: {language: "spanish"}                                          │
│ • OR: Visitor browser language: "es"                                            │
│                                                                                  │
│ Routing:                                                                        │
│   1. Filter: Support dept + Page 1 line                                        │
│   2. Prefer operators with Spanish skill                                       │
│   3. If no Spanish speaker available, route to any                             │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│ Scenario 5: CHANNEL LINE SPECIFIC ROUTING                                       │
│ ──────────────────────────────────────────                                      │
│ • Conversation from: vip-inbox@company.com (special VIP email line)            │
│ • Department: Support                                                           │
│ • Visitor tags: (none)                                                          │
│                                                                                  │
│ The channel line itself has: extra_attributes.is_vip = true                    │
│                                                                                  │
│ Routing:                                                                        │
│   1. Filter: Support dept + vip-inbox@ line                                    │
│   2. Channel line attribute triggers VIP skill requirement                     │
│   3. Route to VIP specialists only                                             │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│ Scenario 6: AUTHENTICATED VISITOR                                               │
│ ─────────────────────────────────                                               │
│ • Conversation from: Widget #1                                                  │
│ • Department: Support                                                           │
│ • Visitor: is_authenticated = true, plan = "enterprise"                         │
│                                                                                  │
│ Routing:                                                                        │
│   1. Filter: Support dept + Widget #1 line                                     │
│   2. Visitor is authenticated enterprise → Premium skill                        │
│   3. Route to Premium specialist                                               │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│ Scenario 7: MULTIPLE SKILLS MATCH                                               │
│ ─────────────────────────────────                                               │
│ • Conversation from: WhatsApp #1                                                │
│ • Department: VIP                                                               │
│ • Visitor tags: [VIP, Enterprise]                                               │
│ • Custom fields: {language: "german", priority: "critical"}                     │
│                                                                                  │
│ Required skills: [VIP, Enterprise, German, Critical]                            │
│                                                                                  │
│ Operator skill scores:                                                          │
│ • John: VIP(10) + German(5) = 15                                               │
│ • Alice: VIP(15) + Enterprise(10) = 25                                         │
│ • Tom: VIP(8) + Enterprise(8) + German(3) + Critical(10) = 29                  │
│                                                                                  │
│ Result: Route to Tom (highest combined skill score)                             │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│ Scenario 8: NO MATCHING OPERATORS                                               │
│ ─────────────────────────────────                                               │
│ • Conversation from: WhatsApp #2                                                │
│ • Department: Support                                                           │
│ • Visitor tags: [VIP]                                                           │
│                                                                                  │
│ But: No operators assigned to WhatsApp #2 in Support                           │
│                                                                                  │
│ Options:                                                                        │
│ A) Queue the conversation, send serve request                                   │
│ B) Fallback to operators with NO channel line restrictions                     │
│ C) Fallback to different department (if configured)                            │
│ D) Route to chatbot/AI until operator available                                │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│ Scenario 9: SKILL-SPECIFIC DEPARTMENT                                           │
│ ─────────────────────────────────────                                           │
│ • Department: VIP (requires VIP skill)                                          │
│ • Conversation from: Widget #1                                                  │
│ • Visitor tags: [VIP]                                                           │
│                                                                                  │
│ The VIP department is configured to ONLY accept VIP-skilled operators:         │
│   queues.extra_attributes.required_skills = ["VIP"]                            │
│                                                                                  │
│ Routing:                                                                        │
│   1. Filter: VIP dept + Widget #1 line                                         │
│   2. MUST have VIP skill (department requirement)                              │
│   3. Route only to VIP specialists                                             │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│ Scenario 10: TRANSFER WITH SKILL PRESERVATION                                   │
│ ─────────────────────────────────────────────                                   │
│ • Original: Sales dept, routed to Jane                                          │
│ • Transfer to: Technical dept                                                   │
│ • Visitor still has: [VIP] tag                                                  │
│                                                                                  │
│ Routing after transfer:                                                         │
│   1. New filter: Technical dept + same channel line                            │
│   2. VIP tag still applies → prefer VIP specialists in Technical               │
│   3. Route to Technical + VIP skilled operator                                 │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Data Visibility by Feature

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    DATA VISIBILITY BY FEATURE                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│ ┌─────────────────────────────────────────────────────────────────────────────┐ │
│ │ INBOX                                                                        │ │
│ ├─────────────────────────────────────────────────────────────────────────────┤ │
│ │                                                                              │ │
│ │ Shows conversations where:                                                   │ │
│ │ • department ∈ operator's departments                                       │ │
│ │ • channel_line ∈ operator's lines (or no line restrictions)                │ │
│ │ • conversation is OPEN                                                      │ │
│ │ • operator is member OR conversation is unassigned (waiting)                │ │
│ │                                                                              │ │
│ │ Admin with conversations:viewAny → sees ALL open conversations             │ │
│ │                                                                              │ │
│ └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│ ┌─────────────────────────────────────────────────────────────────────────────┐ │
│ │ CONVERSATIONS (List/Search)                                                  │ │
│ ├─────────────────────────────────────────────────────────────────────────────┤ │
│ │                                                                              │ │
│ │ Shows all conversations (open + closed) where:                              │ │
│ │ • department ∈ operator's departments                                       │ │
│ │ • channel_line ∈ operator's lines (or no line restrictions)                │ │
│ │                                                                              │ │
│ │ Admin with conversations:viewAny → sees ALL conversations                  │ │
│ │                                                                              │ │
│ └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│ ┌─────────────────────────────────────────────────────────────────────────────┐ │
│ │ SINGLE CONVERSATION VIEW                                                     │ │
│ ├─────────────────────────────────────────────────────────────────────────────┤ │
│ │                                                                              │ │
│ │ Operator can view if:                                                        │ │
│ │ • Is a member of the conversation                                           │ │
│ │ • OR has department + channel line access                                   │ │
│ │                                                                              │ │
│ │ Admin with conversations:viewAny → can view ANY conversation               │ │
│ │                                                                              │ │
│ └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│ ┌─────────────────────────────────────────────────────────────────────────────┐ │
│ │ VISITORS                                                                     │ │
│ ├─────────────────────────────────────────────────────────────────────────────┤ │
│ │                                                                              │ │
│ │ Shows visitors who have conversations in operator's visible scope:          │ │
│ │ • Has conversations in operator's departments                               │ │
│ │ • Has conversations from operator's channel lines                           │ │
│ │                                                                              │ │
│ │ Admin with visitors:viewAny → sees ALL visitors                            │ │
│ │                                                                              │ │
│ └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│ ┌─────────────────────────────────────────────────────────────────────────────┐ │
│ │ REPORTS                                                                      │ │
│ ├─────────────────────────────────────────────────────────────────────────────┤ │
│ │                                                                              │ │
│ │ Report data filtered to:                                                     │ │
│ │ • Only conversations in operator's departments                              │ │
│ │ • Only conversations from operator's channel lines                          │ │
│ │                                                                              │ │
│ │ Available reports:                                                           │ │
│ │ • Own performance → only own data                                           │ │
│ │ • Department performance → visible departments                               │ │
│ │ • Channel analytics → visible channel lines                                 │ │
│ │                                                                              │ │
│ │ Admin with reports:viewAny → sees ALL report data                          │ │
│ │                                                                              │ │
│ └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│ ┌─────────────────────────────────────────────────────────────────────────────┐ │
│ │ DASHBOARD                                                                    │ │
│ ├─────────────────────────────────────────────────────────────────────────────┤ │
│ │                                                                              │ │
│ │ Widgets show data scoped to operator's visibility:                          │ │
│ │                                                                              │ │
│ │ • Open conversations: count from visible scope                              │ │
│ │ • Waiting queue: only visible departments/lines                             │ │
│ │ • My chats: only own conversations                                          │ │
│ │ • Avg response time: calculated from visible conversations                  │ │
│ │ • Channel breakdown: only visible channel lines                             │ │
│ │ • Department breakdown: only visible departments                            │ │
│ │                                                                              │ │
│ │ Admin with dashboard:viewAny → sees ALL dashboard data                     │ │
│ │                                                                              │ │
│ └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│ ┌─────────────────────────────────────────────────────────────────────────────┐ │
│ │ CHANNEL LINES (Settings)                                                     │ │
│ ├─────────────────────────────────────────────────────────────────────────────┤ │
│ │                                                                              │ │
│ │ Operators see only channel lines they are assigned to:                      │ │
│ │ • Can view settings of assigned lines                                       │ │
│ │ • Cannot see or modify other lines                                          │ │
│ │                                                                              │ │
│ │ Admin with channel-lines:viewAny → sees ALL channel lines                  │ │
│ │                                                                              │ │
│ └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│ ┌─────────────────────────────────────────────────────────────────────────────┐ │
│ │ DEPARTMENTS (Settings)                                                       │ │
│ ├─────────────────────────────────────────────────────────────────────────────┤ │
│ │                                                                              │ │
│ │ Operators see only departments they are assigned to:                        │ │
│ │ • Can view department settings                                              │ │
│ │ • Can see other operators in same department                                │ │
│ │ • Cannot see or modify other departments                                    │ │
│ │                                                                              │ │
│ │ Admin with departments:viewAny → sees ALL departments                      │ │
│ │                                                                              │ │
│ └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Database Schema

### New Tables

```sql
-- 1. Channel Line → User assignments (which users handle which channel lines)
CREATE TABLE channel_line_has_users (
    channel_line_id BIGINT UNSIGNED NOT NULL,
    user_id BIGINT UNSIGNED NOT NULL,
    enabled BOOLEAN DEFAULT TRUE,
    priority INT UNSIGNED DEFAULT 1,
    max_concurrent INT UNSIGNED DEFAULT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    PRIMARY KEY (channel_line_id, user_id),
    FOREIGN KEY (channel_line_id) REFERENCES channel_lines(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_enabled (user_id, enabled)
);

-- 2. Operator → Skills (Tag-based routing)
CREATE TABLE operator_skills (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    tag_id BIGINT UNSIGNED NOT NULL,
    priority INT UNSIGNED DEFAULT 1,
    queue_name VARCHAR(255) DEFAULT NULL,  -- NULL = applies to all departments
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY unique_user_tag (user_id, tag_id, queue_name),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE,
    FOREIGN KEY (queue_name) REFERENCES queues(name) ON DELETE SET NULL,
    INDEX idx_tag_priority (tag_id, priority DESC)
);

-- 3. Dynamic routing rules (optional, for complex scenarios)
CREATE TABLE routing_rules (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT UNSIGNED NOT NULL,
    name VARCHAR(255) NOT NULL,
    condition_type ENUM('visitor_tag', 'custom_field', 'visitor_attribute', 
                        'channel_line_attribute', 'conversation_tag') NOT NULL,
    condition_field VARCHAR(255) NOT NULL,      -- e.g., "priority", "language"
    condition_operator ENUM('equals', 'contains', 'greater_than', 
                            'less_than', 'in', 'not_in') NOT NULL,
    condition_value JSON NOT NULL,              -- e.g., "high", ["VIP", "Premium"]
    action_type ENUM('require_skill', 'prefer_skill', 'route_to_department') NOT NULL,
    action_value VARCHAR(255) NOT NULL,         -- skill tag name or department
    priority INT UNSIGNED DEFAULT 1,
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (account_id) REFERENCES accounts(id) ON DELETE CASCADE,
    INDEX idx_account_enabled (account_id, enabled, priority DESC)
);
```

### Modified Tables

```sql
-- Modify queues table extra_attributes to include:
-- {
--     "type": "default|social_hub|pbx|email",
--     "required_skills": ["VIP"],           -- Skills required for this dept
--     "preferred_skills": ["Technical"]     -- Skills preferred but not required
-- }

-- Modify channel_lines table extra_attributes to include:
-- {
--     "is_vip": true,                       -- Conversations here trigger VIP skill
--     "default_skills": ["Enterprise"],     -- Default skills for this line
--     "auto_route_department": "dept-vip"   -- Override department routing
-- }
```

---

## Part 10: Summary

### Architecture Decision

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           RECOMMENDED ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ASSIGNMENT MODEL:                                                               │
│                                                                                  │
│      OPERATOR ────┬──── DEPARTMENTS (queue_has_users)                           │
│                   │         └── Which departments can they work in?             │
│                   │                                                              │
│                   ├──── CHANNEL LINES (channel_line_has_users)                  │
│                   │         └── Which specific lines can they handle?           │
│                   │         └── (Empty = ALL lines in their departments)        │
│                   │                                                              │
│                   └──── SKILLS (operator_skills)                                │
│                             └── What skills do they have?                       │
│                             └── Priority per skill for routing                  │
│                                                                                  │
│  VISIBILITY = Departments ∩ Channel Lines (intersection)                        │
│                                                                                  │
│  ROUTING = Department → Channel Line → Skills → Availability                    │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  SKILL ROUTING SOURCES:                                                          │
│                                                                                  │
│      ┌─────────────────┐                                                        │
│      │ Visitor Tags    │──┐                                                     │
│      ├─────────────────┤  │                                                     │
│      │ Custom Fields   │──┤                                                     │
│      ├─────────────────┤  │     ┌─────────────────┐     ┌─────────────────┐    │
│      │ Visitor Attrs   │──┼────>│ Collected Skills │────>│ Match Operators │    │
│      ├─────────────────┤  │     └─────────────────┘     └─────────────────┘    │
│      │ Channel Line    │──┤                                                     │
│      │ Attributes      │  │                                                     │
│      ├─────────────────┤  │                                                     │
│      │ Routing Rules   │──┘                                                     │
│      └─────────────────┘                                                        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Key Behaviors

| Rule | Behavior |
|------|----------|
| No channel line assignments | Operator sees ALL lines in their departments |
| No department assignments | Operator sees ONLY conversations they're member of |
| No skill match | Route to any available operator (skills are preference, not requirement) |
| Department requires skill | ONLY operators with that skill can receive conversations |
| Multiple skills match | Sum priorities, route to highest score |
| Operator at capacity | Skip, try next operator |
| No operators available | Queue conversation, send serve request |
