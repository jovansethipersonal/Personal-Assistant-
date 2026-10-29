# Personal AI Assistant — Scope

**Status:** Draft v0.1 · **Date:** 2026-10-29

---

## 1. Vision

One assistant that **knows my life** (people, vehicles, accounts, places, routines, work, preferences) and **acts on my behalf** (pays bills, books appointments, handles renewals, answers questions), with autonomy that grows as it proves it can be trusted.

You should be able to say things like:

- "When is the registration due on the truck, and can you renew it?"
- "Did the electric bill go up this month? Pay it if it looks normal."
- "Book an oil change for the Tahoe somewhere near where I'll be Thursday."
- "Pay Meinecke $30 for the haircut." (It knows Meinecke is your barber, that you pay him by Venmo, and his handle.)
- "What did I spend on fuel last quarter?"
- "Remind me to call Mike when I'm leaving the office."

### Design principles

1. **Context first, actions second.** An assistant that knows everything but can't act is still useful. One that acts without context is dangerous. Build the memory first.
2. **Money limits are enforced outside the AI.** The model never decides how much it's allowed to spend. Hard caps live in the payment instrument (capped virtual cards, bank limits) and in a policy engine written in code, not in a prompt.
3. **Autonomy is earned per action type.** Every action starts at "ask me first" and moves up only when I promote it.
4. **Everything is auditable and editable.** I can see what it knows and where each fact came from, and correct it. Every action it takes is logged.
5. **Outside content can't give it orders.** Emails, web pages, PDFs and texts are data. "Pay this invoice" in an email never triggers a payment on its own.

### Your setup (known so far)

| | |
|---|---|
| **Phone** | iPhone |
| **Messaging** | Messages (iMessage), WhatsApp, Instagram |
| **Email** | Several accounts (providers to be confirmed) |
| **Banking and cards** | Amex (cards + banking) |
| **Paying in stores and online** | Apple Pay |
| **Paying people** | Venmo (main), Apple Cash (occasionally), PayPal (rarely) |
| **Home base** | A Mac that can stay on (runs the whole system to start, see §4.3) |
| **Bank data** | Existing Plaid developer account (status to check, see §2.3) |
| **Password manager** | Apple Passwords (source of truth; the assistant keeps copies of the bill-site logins you add) |

---

## 2. Capability map

Each area lists what the assistant **knows** (context) and **does** (actions), plus the realistic data source.

### 2.1 Personal profile and memory (the core)

| Knows | Source |
|---|---|
| Identity, family, key contacts and relationships | Onboarding interview, contacts, email |
| Work: role, business, clients, recurring commitments | Onboarding, calendar, email |
| Preferences (vendors, brands, times, communication style) | Learned from conversations, confirmed by me |
| Important documents (license, insurance cards, titles, policies) | Uploaded docs + OCR |

**Does:** answers questions about my life, fills forms, keeps facts current, notices contradictions ("You said the Silverado was sold, but insurance still lists it").

### 2.2 Vehicles

| Knows | Source |
|---|---|
| Year/make/model/trim, VIN, plate | Onboarding + VIN decode (NHTSA vPIC API, free) |
| Registration and inspection due dates | Docs / DMV notices in email |
| Insurance policy, carrier, renewal date, coverage | Policy docs, carrier emails |
| Loan or lease: lender, payment, payoff | Bank data, statements |
| Mileage, fuel level, location, maintenance due | Connected-car API (e.g. Smartcar, OEM apps) where supported, otherwise fuel and service receipts |
| Open recalls | NHTSA recalls API (free) |
| Service history | Receipts, shop emails |

**Does:** reminds about renewals and maintenance, checks recalls, books service, renews registration online (browser agent, with approval), tracks cost of ownership per vehicle.

### 2.3 Bills and finances

| Knows | Source |
|---|---|
| Accounts, balances, transactions | **Amex via Plaid.** Amex has an official data-sharing connection with Plaid: you sign in on Amex's own screen and the assistant gets a revocable read-only token, never your password. Personal-friendly options (SimpleFIN, Teller) only if they cover Amex. |
| Recurring bills: payee, typical amount, due date, autopay status | Derived from transactions + bill emails |
| Subscriptions | Derived from transactions |
| Unusual charges or price changes | Anomaly detection vs. history |

**Does:** makes sure nothing is missed, flags anomalies, pays bills (see §5 for how), cancels unwanted subscriptions, answers spending questions, prepares tax-relevant summaries.

#### Finding your bills from Amex statements

During setup the assistant pulls up to **24 months** of Amex card and checking transactions and finds the recurring ones itself: same payee, regular interval, similar amount. You get a list to confirm:

> *I found 14 recurring bills: Duke Energy (~$140, monthly, around the 12th), State Farm ($212 every 6 months), Verizon ($95, monthly), … Which of these are on autopay?*

What you confirm becomes the **known payee list** (§3).

| Bank transactions show | Filled in from elsewhere |
|---|---|
| Who you pay, how much, when, how often | **Due dates and account numbers** come from bill emails |
| Subscriptions and price increases | **Autopay status**: you confirm once; a charge on the same day every month is a strong hint |
| Your Amex card bill: statement balance, minimum payment, due date (Plaid Liabilities) | **Venmo details**: the bank only shows "VENMO" and a total; who and why come from Venmo receipt emails |
| | **Bills paid from other accounts or by someone else** don't appear; add them by hand |

The recurring-bill detection is built in (Plaid sells its own as a paid add-on, which isn't needed).

#### Plaid: what to check in your existing account

- **Plaid's old "Development" environment is gone.** Plaid now has Sandbox (fake data only) and Production (real accounts), so keys from a previous attempt may not work for real data.
- In the Plaid dashboard, check which plan the account is on:
  - **Limited Production:** the older free tier for existing accounts. Check that **American Express** shows as available on the *OAuth institutions* page.
  - **Trial plan:** the newer free tier (accounts created after April 15, 2026). Up to 10 connected accounts, and it includes Amex.
  - **Full Production:** needs Plaid's registration (company info and a security questionnaire). Institutions usually turn on within hours to a couple of days.
- Products needed: **Transactions** and **Liabilities** (card due dates and balances).
- **Don't paste Plaid keys into chat.** They go straight into the Mac's keychain during setup (see §4.3).

#### Paying people (Venmo, Apple Cash, PayPal)

"Pay Meinecke $30 for haircut" needs four things the assistant has to know or work out:

| Step | What it resolves | How it knows |
|---|---|---|
| **Who** | "Meinecke" → the person you mean | Contacts + memory. Uses context to pick the right one: "haircut" means your barber, not the Meineke car shop. Asks if it's still unsure. |
| **How** | You pay this person by Venmo | Learned from past payments (Venmo receipt emails, bank transactions) or told once and remembered |
| **Where** | Their Venmo handle | Taken only from a past successful payment or from you directly, never from a text or email (see §3) |
| **What's normal** | Usually $30, note "haircut", about every 3 weeks | Payment history |

Then it makes the payment the safest way that payment app allows (see §5.1), confirms it went through, and logs it.

### 2.4 Places, routines and location

| Knows | Source |
|---|---|
| Home, work, frequent places (gym, fuel stops, customers) | Location history clustered into "places" |
| Typical schedule and commute | Location + calendar |
| Where I am now (when I allow it) | Phone location stream |

Note: Google Maps Timeline moved to on-device storage and no longer has a usable cloud history, so **location has to be collected by us**. On iPhone, the free OwnTracks or Overland apps run in the background and post your location to our server, with no custom app needed. Apple Shortcuts automations ("when I arrive at / leave a place") can add exact arrive and leave events.

**Does:** location-based reminders ("when I leave work"), plans errands along existing routes, suggests appointment times and places that fit my routine, logs mileage.

### 2.5 Messaging, email and calendar

Each messaging app could play three different roles, and what's possible varies a lot between apps and roles:

| App | 1. You talk to the assistant | 2. It reads your conversations for context | 3. It sends messages as you |
|---|---|---|---|
| **Messages (iMessage)** | ✅ The assistant gets its own Apple ID and iMessage contact, run from your Mac (see below) | ✅ Your Mac reads your local Messages history. | ✅ From your Mac. Always needs your approval at first. |
| **WhatsApp** | ✅ Official WhatsApp Business Cloud API. The assistant gets its own WhatsApp number. | ❌ No official API for personal chats. Unofficial "linked device" tools break WhatsApp's terms and can get your number banned. | ❌ Same |
| **Instagram** | ⚠️ Meta's messaging API works only for Business and Creator accounts | ❌ No API for personal DMs | ❌ No |
| **Email (all your accounts)** | ✅ Email it or forward things to it | ✅ Gmail API, Microsoft Graph, or IMAP (iCloud, Yahoo, others) for each account | ✅ From the right account, with approval at first |

**Recommendation:**

- **Main channel: iMessage, run from your Mac.** The assistant shows up as a contact in Messages and you text it like a person. Payment links (like the prefilled Venmo payment) open straight in the app. No hosted iMessage service is needed.
  - Messages on a Mac can only be signed in to **one Apple ID per macOS user**. So the Mac gets **two macOS users**:
    - **"Assistant"**: signed in to a **new Apple ID** made just for the assistant. That's the contact you text. An iMessage bridge (BlueBubbles, or a small custom one) runs here.
    - **You**: signed in to your Apple ID. A small read-only process indexes your Messages history for context, and later sends as you, with approval.
  - Your message history includes other people's words. It's searched when needed and stays on the Mac, and you can exclude specific chats.
- **Second channel: WhatsApp**, through the official API. It's the same assistant with the same memory. WhatsApp lets a business number message you freely only within 24 hours of your last message, and after that only with pre-approved templates. That makes iMessage the better channel for reminders the assistant starts on its own.
- **Instagram: skip for now.** If something there matters, share it to the assistant with the iPhone share sheet.
- **Voice: Siri.** An Apple Shortcut ("Hey Siri, ask Assistant…") sends what you say to the assistant and reads the answer back.
- **Money approvals use Face ID.** Payments that need approval are confirmed in the approval app with Face ID (a passkey), not by texting back "yes". A text reply is easier to fake or to send by accident. Venmo payments are confirmed in Venmo itself.

**Several email accounts:**

- Connect each account separately and label it (personal, business, old/shopping). The assistant reads all of them for bills, receipts, renewals and notices, and skips duplicate copies.
- When replying, it uses the account the conversation came in on. For a new email, it uses the account you normally use with that person, and asks if it isn't clear.
- Contacts and calendar come from wherever they live: iCloud (CardDAV/CalDAV with an app-specific password), Google, or Microsoft.

| Knows | Source |
|---|---|
| Schedule, commitments, travel | iCloud / Google / Microsoft calendars |
| Bills, receipts, confirmations, notices | All connected email accounts |
| Who matters and how you talk to them | Contacts, email, and iMessage history if a Mac is available |

**Does:** daily brief, schedules and reschedules, drafts replies (sends only with approval at first), files receipts and documents automatically.

### 2.6 Errands and long-tail tasks

Booking appointments, ordering, renewals, filling web forms, calling businesses.

- **APIs first** where they exist (calendar, payments, some booking platforms).
- **Browser agent** (sandboxed, headless browser driven by the model) for everything else: DMV renewals, utility portals, booking sites.
- **Phone calls** (later phase): voice agent for simple calls like scheduling or checking hours, always disclosed as an assistant.

### 2.7 Out of scope for v1 (revisit later)

Health and medical records · investing and trading · tax filing · smart-home control · acting for other family members · business operations at scale (dispatch, payroll, fleet management).

---

## 3. Autonomy model

Every action type has a level. Everything starts at L1 or L2. I promote action types over time.

| Level | Meaning | Example |
|---|---|---|
| **L0 — Inform** | Tells me, does nothing | "Your registration expires in 30 days." |
| **L1 — Prepare** | Does all the work, I execute | Fills the renewal form, sends me the link |
| **L2 — Approve** | Asks, then acts on one tap | "Pay $142.18 to Duke Energy? ✅ / ❌" |
| **L3 — Act within limits** | Acts, tells me after | Pays known utility bills ≤ 120% of the 6-month average |
| **L4 — Autonomous** | Acts, shows up in the weekly digest | Files receipts, updates memory, renews subscriptions I've whitelisted |

**Policy rules (code, not prompt):**

- Payee must be on the **known payee list**. New payees always require L2 approval and are verified against a source I control (prior statements), never just an email.
- **Person-to-person payments can't be reversed**, so they stay at L2 by default. A payment handle (Venmo, Cash App, PayPal, Zelle phone or email) is trusted only if a past payment to it succeeded or I entered it myself. If anyone says "I have a new Venmo", the assistant asks me before paying it.
- Per-transaction, per-day and per-month spend caps.
- Amount anomaly (for example more than 20% above typical) → drop back to L2.
- Anything irreversible (sending to a new contact, cancelling insurance, wire transfers) → always L2 or higher.
- Kill switch: one command pauses all actions.

---

## 4. Architecture

```mermaid
flowchart TB
    subgraph Interfaces
        CHAT[Chat: iMessage + WhatsApp]
        VOICE[Voice: Siri Shortcut]
        APP[Web app on iPhone:<br/>Face ID approvals, dashboard]
        LOCAPP[OwnTracks / Shortcuts:<br/>location]
    end

    subgraph Core
        ORCH[Agent orchestrator<br/>Claude + tool use]
        POLICY[Policy engine<br/>limits, autonomy levels, payee list]
        SCHED[Proactive engine<br/>schedules + event triggers]
        APPROVE[Approval queue]
    end

    subgraph Memory
        KG[(Structured store<br/>entities: vehicles, accounts,<br/>bills, places, people)]
        VEC[(Semantic store<br/>notes, emails, docs)]
        TL[(Timeline / event log)]
        DOCS[(Document vault)]
    end

    subgraph Integrations
        FIN[Finance aggregator]
        MAIL[Email + calendar]
        CAR[Vehicle APIs + NHTSA]
        LOC[Location ingest]
        PAY[Payment rails<br/>capped virtual cards, bill pay]
        BROWSER[Sandboxed browser agent]
    end

    VAULT[[Secrets vault]]
    AUDIT[[Audit log]]

    CHAT --> ORCH
    VOICE --> ORCH
    APP --> ORCH
    LOCAPP --> LOC
    SCHED --> ORCH
    ORCH <--> Memory
    ORCH --> POLICY
    POLICY -->|allowed| Integrations
    POLICY -->|needs approval| APPROVE --> APP
    Integrations --> Memory
    BROKER[[Secret broker<br/>types secrets in, AI never sees them]]
    BROWSER --> BROKER
    PAY --> BROKER
    BROKER --> VAULT
    POLICY --> AUDIT
```

### 4.1 Components

**Interfaces**
- **Chat** is the main interface: iMessage first, WhatsApp second (see §2.5). Both feed the same conversation and memory.
- **Web app on iPhone** (added to the home screen, so it can send push notifications) handles Face ID approvals, the "what I know about you" page and the action log.
- **Location** comes from OwnTracks or Overland plus Shortcuts automations. No custom iOS app.
- **Voice:** a Siri Shortcut that sends what you say to the assistant.

**Agent orchestrator**
- Claude via the Claude Agent SDK / API with tool use. Tools exposed as MCP servers so each integration is a separate, testable module.
- System context holds a compact **profile summary**. Detailed facts are retrieved on demand.
- Integrations and their tools are grouped by risk class (read / write / money), which the policy engine uses.

**Memory** (the most important part to get right)
- **Structured entities** in Postgres: `Person`, `Vehicle`, `Account`, `Bill`, `Payee`, `Place`, `Document`, `Routine`, `Preference`. Typed, queryable, and exact.
- **Semantic store** (pgvector) for unstructured content: email bodies, notes, document text.
- **Timeline**: append-only event log (trips, payments, appointments, conversations).
- **Every fact carries provenance**: source, timestamp, confidence, and whether I confirmed it. Low-confidence facts get confirmed in conversation before they're relied on.
- **Memory write pipeline**: after each conversation or ingested email, an extraction pass proposes fact updates. High-confidence ones are applied, others are queued for confirmation.

**Proactive engine**
- Scheduled jobs: daily brief, bill-due scans, renewal checks, recall checks, weekly digest.
- Event triggers: new email from a biller, new transaction, arriving at or leaving a place, calendar changes.

**Action layer**
- Prefer official APIs. Fall back to the browser agent in a sandboxed container with per-site credentials pulled from the vault at run time. The model never sees raw passwords.
- Every action goes through the policy engine and lands in the audit log, including screenshots for browser actions.

**Security** (full design in [SECURE-CREDENTIALS.md](SECURE-CREDENTIALS.md))
- **The AI model never sees a password, card number or code.** It refers to secrets by name, and a separate secret broker (plain code) types them in, only on the right site and only for an active task.
- **Saved logins for any bill site you name.** You add each one once through **Secure Entry**: a Face ID page on your iPhone where you fill the login from Apple Passwords, encrypted on the phone. From then on, the broker logs in by itself, only on that site.
- **Verification codes are handled hands-free where possible:** text-message codes arrive on the Mac through Text Message Forwarding, email codes are read from your inbox, and authenticator codes are generated. Only push approvals need you.
- Your Amex login, Apple ID, main email passwords and Venmo/PayPal logins are **never** given to the assistant. It uses revocable tokens or prepares actions for you to finish.
- Your passwords stay in **Apple Passwords**, which no other program can read. The assistant keeps its own copies of only the logins you add, in the macOS Keychain on the Mac (or 1Password). Encrypted database. MFA on everything.
- **Prompt-injection defense**: content from email and the web is tagged untrusted. Untrusted content can inform but can't authorize. Payment and new-recipient tools require the payee to match the known payee list.
- Single-tenant: it's your data, on your own Mac (§4.3).

### 4.2 Suggested stack

| Layer | Choice | Why |
|---|---|---|
| Language | TypeScript (or Python) | Good SDKs for Claude, Plaid, Google, Playwright |
| Agent | Claude Agent SDK + MCP tool servers | Tool use, subagents, browser and computer use |
| Database | Postgres + pgvector | One store for structured data and embeddings |
| Jobs | Postgres-backed queue + cron (Temporal if it grows) | Keeps it simple at first |
| Chat | iMessage bridge on your Mac (BlueBubbles or custom) + WhatsApp Business Cloud API | Where you already message, no monthly fee |
| App | PWA (Next.js) with passkeys | Face ID approvals and push on iPhone, no App Store needed |
| Location | OwnTracks or Overland (iOS) → HTTPS endpoint | Already built, runs in the background |
| Email | Gmail API / Microsoft Graph / IMAP, one connection per account | Covers all your accounts |
| Browser agent | Playwright on the Mac, in its own sandboxed user/profile | Home internet connection gets blocked less than cloud servers; screenshots for the audit log |
| Saved logins | Secret broker + assistant vault in the Mac Keychain (1Password optional), added via Secure Entry from Apple Passwords | Hands-free logins, no new password manager needed |
| Hosting | **Your Mac** + Cloudflare Tunnel or Tailscale for webhooks and the web app, nightly encrypted offsite backups | No hosting bill, data stays home |
| App secrets | macOS Keychain of the "Assistant" macOS user | Separate from your Apple Passwords |


### 4.3 Your Mac as home base

Your Mac runs the whole system to start. You can move the core to a cloud server later if uptime becomes a problem.

| Runs on the Mac | Why there |
|---|---|
| Assistant core, database, scheduler | Your data stays home; no hosting bill |
| iMessage bridge ("Assistant" macOS user) | Only a Mac can send and receive iMessage |
| Your Messages history reader (your macOS user) | Your history only exists on Apple devices |
| Browser agent | Utility and DMV sites block home connections less often than cloud servers |
| Calendar, Contacts, Reminders | Read directly on the Mac if they're in iCloud |
| App secrets (Plaid keys, AI API key) | In the Assistant user's macOS Keychain, separate from your Apple Passwords |

**Reaching it from outside.** Plaid and WhatsApp send updates to the assistant, and the Face ID web app has to reach it from your phone. Cloudflare Tunnel or Tailscale Funnel handles this with **no open ports** on your router.

**Keeping it up:**

- Energy settings: never sleep, and start up automatically after a power failure.
- **FileVault gotcha:** with disk encryption on (it should be), a Mac that restarts unexpectedly waits for a password before anything runs, including both macOS users. A small battery backup (UPS) prevents most unplanned restarts. Planned restarts for macOS updates can skip the password prompt once (`fdesetup authrestart`), but both users still need to log back in.
- An outside uptime monitor alerts you by push notification or email if the Mac goes quiet, since a downed Mac can't iMessage you.
- Nightly encrypted backups to an offsite location.

---

## 5. Paying bills (the hardest part)

There is **no consumer API to drive your bank's bill pay**, and scripting a bank's website is fragile and usually breaks its terms of service. So bill payment is layered:

| Tier | Mechanism | Covers | Risk |
|---|---|---|---|
| **1. Autopay, supervised** | Billers' own autopay. The assistant checks it's on, watches each amount, and alerts or stops on anomalies | Most recurring bills (utilities, phone, insurance, loans, cards) | Lowest |
| **1b. Pay on the biller's site** | The broker logs in with the saved login and pays with the **card or bank account already on file** with that biller | Bills without autopay, or when you want each payment checked before it goes | Low; known payee, known site, follows the autonomy policy |
| **2. Capped card for the assistant** | A merchant-locked virtual card funded from Amex Rewards Checking (e.g. Privacy.com), or an Amex Additional Card with a spending limit. The browser agent pays on the biller's site and the secret broker types in the card number. See [SECURE-CREDENTIALS.md §7](SECURE-CREDENTIALS.md). | Bills without autopay, one-off payments, renewals (DMV, tolls) | **Hard cap enforced by the card, not the AI** |
| **2b. Apple Pay** | The assistant prepares the checkout, you finish with Face ID | Online purchases where you'd use Apple Pay anyway | You confirm every one |
| **3. Bank-to-biller APIs** | Debt-payment APIs (e.g. Method Financial) or ACH via Plaid Transfer | Credit cards, loans; more of a business setup | Medium; needs approval flow |
| **4. Prepared for me** | Assistant fills everything in, I tap pay in my bank app | Anything else | None |

**Recommendation:** Start with tier 1 + tier 4 (phase 1). Add tier 1b once saved logins work (phase 2–3), then tier 2 for sites with no payment method on file. Treat tier 3 as optional.

The monthly bill workflow:

1. Detect the bill (email, portal, or recurring transaction pattern).
2. Match to a known payee and compare to history.
3. Autopay on and amount normal → log it, mention it in the digest.
4. Not on autopay → pay per the autonomy level for that payee.
5. Confirm it cleared (transaction appears) → update the timeline.

### 5.1 Paying people (Venmo and similar)

**The constraint:** Venmo retired its own developer and payouts APIs for new developers. Zelle and Apple Cash have no consumer APIs. Cash App has no public API for sending money between people. So "pay Meinecke from *your* Venmo" can't be a fully hands-off API call.

| Option | How it works | Your effort | Verdict |
|---|---|---|---|
| **A. Prefilled payment** (recommended) | The assistant sends a link that opens Venmo with Meinecke's handle, $30 and "Haircut" already filled in. You check it and tap **Pay**. Cash App (`cash.app/$tag/30`) and PayPal (`paypal.me/name/30`) links work the same way. | One or two taps | Uses Venmo's official payment links. Money comes from your own Venmo. You see exactly what's being sent. |
| **B. PayPal Payouts to a Venmo handle** | PayPal's Payouts API can send to a Venmo handle (US only). The money comes from a **PayPal Business** balance you top up. | None after setup | Truly hands-free, and the balance you top up is a hard cap. Downsides: needs a PayPal Business account, payments show as coming from the business, there are fees, and it mixes personal and business money. Check current eligibility with PayPal first. |
| **C. Automate the Venmo app or website** | A browser or phone-control agent logs into your Venmo and taps through | None | **Not recommended.** Likely breaks Venmo's terms, trips fraud checks and 2FA, and can get your account frozen. |
| **D. Apple Cash** | No API. The assistant tells you what to send. You send it in Messages with Face ID. | A few taps | For people you pay with Apple Cash |
| **E. PayPal** | A `paypal.me/name/amount` link, prefilled like Venmo | One or two taps | Rare use, so the link is enough |

**Flow for "Pay Meinecke $30 for haircut" (option A):**

1. Work out who Meinecke is, his method (Venmo) and his verified handle from memory.
2. Check the policy: known handle, amount close to his usual $30.
3. Reply: *"Venmo $30 to @Meinecke-Barber, note 'Haircut'? [Open in Venmo]"*. One tap opens Venmo prefilled, a second tap pays.
4. Watch for Venmo's "You paid…" receipt email to confirm it went through, then log it to the timeline.
5. If no receipt shows up within an hour, remind me once.

Later, if you set up option B, a payment like this (verified handle, normal amount) can move to L3 and happen with no taps at all.

---

## 6. Phased roadmap

Estimates assume one developer working with AI coding help, part-time.

### Phase 0 — Foundations (2–3 weeks)
- Mac setup: "Assistant" macOS user + Apple ID, power settings, tunnel, backups, health check.
- Repo, database schema, audit log.
- Secret broker skeleton, encrypted saved browser sessions, masking of anything sent to the model, kill switch.
- iMessage chat + Claude orchestrator.
- Onboarding interview that fills the profile (people, vehicles, work, accounts, places).
- "What I know about you" page (view and edit facts).
- **Done when:** I can ask "What cars do I have and when is each registration due?" and get correct answers.

### Phase 1 — Awareness (3–4 weeks)
- Read access to all email accounts + calendar. Ingest bills, receipts and notices.
- WhatsApp as a second chat channel.
- Amex via Plaid (read-only, OAuth), 24 months of history. Detect recurring bills and subscriptions, and confirm the payee list with you.
- Index your iMessage history (read-only) for context.
- VIN decode, recall checks, renewal tracking.
- Daily brief + reminders. Bill-due alerts (tier 1 and 4 payments).
- **Done when:** a month passes with no missed bill or renewal, and I didn't have to tell it about any of them.

### Phase 2 — Saved logins and places (4–5 weeks)
- "Add my Duke Energy account": Secure Entry from Apple Passwords, test login, saved in the assistant vault.
- Broker logs in to bill sites by itself: saved browser sessions, text-message codes via the Mac, email and authenticator codes.
- Reads balances, due dates and statements from each bill site (read-only for now).
- Location ingest. Place clustering (home, work, frequent stops).
- Location-triggered reminders. Mileage log per vehicle.
- Routine model (typical week) used for scheduling suggestions.
- **Done when:** I've added my bill sites once, and the assistant tells me every balance and due date without asking me for anything. "Remind me when I leave work" works too.

### Phase 3 — Actions with approval (4–6 weeks)
- Approval queue + push notifications in the PWA.
- Paying on bill sites with the card or account on file, using the saved logins.
- Capped card for the assistant + browser agent for paying bills and renewals, with every charge reconciled.
- Paying people: "Pay Meinecke $30" → prefilled Venmo, Cash App or PayPal link, confirmed from the receipt email (§5.1 option A).
- Calendar writes, email drafts and sends (with approval).
- Booking appointments (service shops, etc.) via APIs or browser.
- **Done when:** I approve a bill payment and a car service booking from my phone in one tap each, and "Pay Meinecke $30 for haircut" takes me two taps.

### Phase 4 — Earned autonomy (ongoing)
- Per-action autonomy promotion (L2 → L3 → L4) with limits.
- Weekly digest of everything done.
- Optional: hands-free payments to people via PayPal Payouts (§5.1 option B).
- Voice interface. Simple outbound phone calls.
- **Done when:** most routine admin happens without me, and the weekly digest is the main thing I read.

**Rough total to a strong v1 (phases 0–3): ~3–4 months part-time.**

---

## 7. Running costs (rough, monthly)

| Item | Estimate |
|---|---|
| LLM usage (chat + background jobs) | $20–100, depending on volume and model mix |
| Hosting | $0 on your Mac · ~$5 for offsite backups · one-time ~$60–120 for a battery backup (UPS) |
| Plaid | Free tier may cover a few accounts. Liabilities may cost extra; check your plan. |
| Connected-car API | Free tier to low tens of dollars, depending on provider and vehicle count |
| iMessage | $0 (bridge on your Mac) |
| WhatsApp Business API | Likely ~$0 at personal volume (Meta gives a monthly free allowance; pricing changes in 2026, so check) |
| **Total** | **~$30–150/month**, mostly AI usage |

Verify pricing for each provider before committing; these change often.

---

## 8. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Wrong or fraudulent payment | Known-payee list, card-level caps, anomaly checks, approvals, kill switch |
| Paying the wrong person (lookalike handles, "my Venmo changed" scams) | Handles trusted only from past successful payments or my own input; changed handles always need my approval; I always see the handle before I tap Pay |
| Prompt injection (malicious email or web page) | Untrusted-content tagging; money tools require verified payees; no auto-action on email instructions |
| Data breach (this system holds everything) | Single tenant, encryption, vault for secrets, MFA, minimal retention of raw data, no secrets in model context. The highest-value logins (Amex, Apple ID, main email) are never stored at all. |
| Credential leaks or misuse | Secret broker with site and task binding, Secure Entry, masking, audit of every use. See [SECURE-CREDENTIALS.md](SECURE-CREDENTIALS.md). |
| Wrong facts in memory | Provenance + confidence on every fact, confirmation for low-confidence facts, editable memory |
| Browser automation breaks or violates terms | APIs first; browser only for portals that allow it; screenshots + failure alerts; never script bank logins |
| Over-reliance (silent failures) | Weekly digest, "things I couldn't do" alerts, health checks on every integration |
| Assistant sends a message as you that you didn't intend | Sending as you (email, iMessage) needs approval until you promote it, per contact; it never sends as you in WhatsApp or Instagram |
| Integration sprawl | Each integration is its own MCP module with tests; add only when a real task needs it |

---

## 9. Build vs. buy

| Option | Effort | Fit |
|---|---|---|
| **Off-the-shelf assistant + connectors** (Claude with memory + Gmail/Calendar connectors, a budgeting app) | Days | Good for context and Q&A, weak on proactive work and payments |
| **Custom build on Claude Agent SDK** (this document) | ~3–4 months part-time | Full control, deep personal context, safe action-taking |
| **Hybrid** — start with off-the-shelf for Q&A while building phases 0–1 | — | **Recommended**: get value now, build the parts nothing off the shelf does well (memory model, bills, vehicles, policy engine) |

---

## 10. Open questions (need your input)

1. ~~**Phone / Mac:**~~ iPhone + a Mac that stays on. **Which Mac and macOS version?** And is it OK to create a separate Apple ID for the assistant?
2. ~~**Main interface:**~~ iMessage, then WhatsApp. **Email providers:** which services are your accounts on (Gmail, Outlook, iCloud, other), and which are personal vs. business? **Calendar:** iCloud, Google or Outlook?
3. **Personal vs. business:** Should it cover your work and business (email, vehicles, bills), or keep strictly personal with business later?
4. **Who else:** Just you, or family members too (shared vehicles, household bills)?
5. **Money comfort level:** Is "one-tap approve every payment" acceptable at first? What's a per-payment amount you'd be fine with it paying on its own later?
6. ~~**Banks:**~~ Amex. **Billers:** which utilities, insurance companies and lenders do you pay? ~~**Password manager:**~~ Apple Passwords. **Assistant vault:** Mac Keychain (free) or 1Password? **Text Message Forwarding** to the Mac OK? **Assistant's card:** a virtual card service or an Amex Additional Card? (See [SECURE-CREDENTIALS.md](SECURE-CREDENTIALS.md) §12.)
7. **Vehicles:** Makes and model years? (Determines connected-car API coverage.)
8. ~~**Hosting:**~~ Your Mac. **Plaid:** which plan does the dashboard show, and is American Express listed as available?
9. ~~**Paying people:**~~ Venmo, Apple Cash, PayPal. Is "two taps to confirm" fine, or do you want the hands-free PayPal Business setup?
10. **Build:** Are you building this yourself (with Claude Code), hiring, or a mix?

---

## Appendix A — Core data model (first cut)

```
Person      (id, name, aliases, relationship, role, contact_methods,
             default_payment_handle_id, notes)
PaymentHandle (id, person_id, app [venmo|zelle|cashapp|paypal|applecash],
             handle, verified, verification_source, first_paid_at,
             last_paid_at, typical_amount, typical_note)
Vehicle     (id, vin, year, make, model, trim, plate, state, owner_id,
             registration_due, inspection_due, insurance_policy_id,
             loan_account_id, odometer, odometer_as_of)
Account     (id, institution, type, mask, aggregator_ref, owner_id)
Payee       (id, name, category, verified, verification_source,
             payment_methods, portal_url, autopay_enabled)
Bill        (id, payee_id, amount, due_date, status, source, paid_txn_id)
Place       (id, label, lat, lng, radius, category, visit_stats)
Routine     (id, description, pattern, places, confidence)
Document    (id, type, entity_ref, expires_at, file_ref, extracted_fields)
Preference  (id, domain, key, value, source, confirmed)
Fact        (id, subject_ref, predicate, value, source, confidence,
             confirmed, valid_from, valid_to)
Action      (id, type, autonomy_level, status, request, result,
             approved_by, approved_at, audit_refs)
```

## Appendix B — Example proactive jobs

| Job | Schedule | Output |
|---|---|---|
| Daily brief | 6:30am | Calendar, bills due, reminders, weather on routes |
| Bill scan | Daily | New bills, anomalies, upcoming due dates |
| Renewal check | Weekly | Registration, insurance, licenses, domains, IDs expiring within 60 days |
| Recall check | Monthly | Open recalls per VIN |
| Subscription review | Monthly | New, increased, or unused subscriptions |
| Weekly digest | Sunday evening | Everything done, everything pending, anything it couldn't do |
