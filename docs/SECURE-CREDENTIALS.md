# Sensitive Credentials and Data — Design

**Status:** Draft v0.1 · **Date:** 2026-10-29 · Part of [SCOPE.md](SCOPE.md)

---

## 1. Goal

The assistant can log into sites, fill in forms, enter verification codes and pay for things **without the AI model ever seeing a password, card number or code**, and without you handing it the keys to your bank, email or Apple ID.

## 2. The core rule: the model handles references, never secrets

The AI decides *what* to do. A small, separate program (the **secret broker**, ordinary code with no AI in it) does the sensitive part.

```mermaid
sequenceDiagram
    participant M as AI model
    participant B as Secret broker (code)
    participant V as Assistant vault
    participant W as Browser (sandboxed)
    M->>B: fill_secret(field="password", ref="duke-energy/login")
    B->>B: Check: right website? active task? approval needed?
    B->>V: Fetch secret
    V-->>B: ••••••••
    B->>W: Type it into the field
    B-->>M: "Filled ✓" (never the value)
    Note over M,W: Screenshots and page text sent to the model<br/>have password, card and code fields blacked out
```

The model only ever sees names like `duke-energy/login` or `SSN on file (•••• 1234)`. Even if the model is tricked by a malicious email or web page, it has nothing to leak.

---

## 3. Four tiers: what gets stored, and where

| Tier | How the assistant gets access | Examples | What's stored |
|---|---|---|---|
| **A. Never needs your password** | You sign in once on the provider's own screen (OAuth). The assistant gets a token you can revoke anytime. | **Amex** card and checking (via Plaid; you sign in on Amex's screen with Face ID), Gmail / Outlook, Google Calendar | A revocable token only |
| **B. Stored, typed in by the broker** (optional) | Login saved in a separate assistant vault, used only on its own site. Only for sites you want handled hands-free (§4). | Utility portals, DMV, insurance, tolls, auto shops, booking sites | Encrypted in the assistant vault |
| **C. You enter it when it's needed** | The assistant asks, you enter it with Face ID through **Secure Entry** (§5) | Verification codes, logins you'd rather not store, one-off sites | Nothing (used once, then gone) |
| **D. Never given to the assistant** | — | **Amex login**, **Apple ID**, your main email passwords, **Venmo / PayPal logins**, password-manager master password, recovery codes | — |

Tier D is deliberate. Those accounts can reset everything else, so the assistant gets **read access via tokens** (tier A) or **prepares actions for you to finish** (Venmo links, Apple Pay), never their passwords.

---

## 4. Where secrets live

**Default plan: Apple Passwords only.** Your passwords stay in Apple Passwords and nothing new is needed. Apple Passwords has no API, so the assistant **can't read it on its own**. Every login goes through you:

1. The assistant needs to log in somewhere → texts you *"Need your Duke Energy login. [Enter securely]"*
2. You open Secure Entry (§5) with Face ID, tap the password field, choose **Passwords**, find Duke Energy, and Face ID again to fill it.
3. The broker logs in and **saves the session** (encrypted), so the site stays logged in for days to weeks. You're only asked again when the site logs it out.

This works because most of the assistant's access needs no password at all. Amex, email and calendar use tier A tokens, and email codes are read directly. Logins are only needed for occasional portals (DMV, utility sites, tolls), so expect **a few Face ID prompts a month**.

**The trade-off:** the assistant can't log in while you're asleep or busy. A 3 a.m. bill check that hits an expired session waits for you.

**Optional later: a small assistant vault.** If those prompts get annoying for particular sites, put just those logins in a separate 1Password vault (§4.1). The assistant can then log in without you, only on those sites. Revoking that vault's token cuts off all of them at once, as part of the kill switch.

### 4.1 Connecting a password manager

| Password manager | Can the assistant connect to it? | How |
|---|---|---|
| **Apple Passwords / iCloud Keychain** | ❌ No API for servers or agents. You fill logins in through Secure Entry with Face ID (the default plan above). | To go hands-free on some sites, move only those logins into an assistant vault with **Export Data to Another App** in the Passwords app (iOS 26 / macOS Tahoe). It's encrypted and needs Face ID, unlike the old unencrypted CSV export. Check that the receiving app supports it; if not, copy the handful of logins by hand. |
| **1Password** (recommended) | ✅ | **Service account** limited to the one "Assistant" vault, read-only, used by the broker through the 1Password SDK. Optionally, 1Password's **Secure Agentic Autofill** (early access, with Browserbase) fills logins into an agent's browser after you approve each request in the 1Password app, which is the same pattern as our broker. |
| **Bitwarden** | ✅ | **Secrets Manager** or its **Agent Access SDK** (credentials released on demand, with human approval) used by the broker. Bitwarden also ships an MCP server, but tools that hand the password back to the AI model break the core rule (§2), so use it only behind the broker. |

**Never:** give the assistant your master password or access to all vaults, leave CSV exports lying around, or scrape the Mac keychain with unofficial tools.

---

## 5. Secure Entry: you type it, the assistant never sees it

For anything the assistant doesn't have (tier C), it asks you through a private page instead of chat.

1. The assistant texts you: *"Duke Energy just sent a code to your phone. [Enter securely]"*
2. The link opens the **Secure Entry** page in the assistant's web app. **Face ID** (passkey) is required to open it.
3. The page says exactly what's being asked for and for which site: *"Duke Energy (duke-energy.com) — verification code"*.
4. Your iPhone fills it in for you:
   - **Text-message codes:** iOS shows the code from Messages above the keyboard. One tap.
   - **Passwords:** tap the field, choose **Passwords**, pick the site's login and Face ID to fill it. iOS won't suggest it on its own, because the Secure Entry page isn't on that site's domain.
   - **Passkey-only sites** can't be handled this way: a passkey only works on its own website, on your device. For those, you sign in yourself, or use the site's password option if it has one.
5. The value is **encrypted on your phone** so only the broker can read it. The server never stores it in readable form. The broker types it in and throws it away.
6. If you've set up the optional assistant vault (§4), a **"Save to assistant vault"** checkbox moves a login to tier B, so you won't be asked again.
7. Links are single-use and expire after 5 minutes.

**Never send passwords or codes in iMessage or WhatsApp.** Chat history is stored by the messaging provider and passes through the AI model. If you do it by accident, the assistant redacts it from its memory and tells you to change that password.

---

## 6. Verification codes (2FA)

| Where the code arrives | How it's handled | Your effort |
|---|---|---|
| **Text message** | Secure Entry. iOS suggests the code automatically. | Face ID + one tap |
| **Email** | The broker reads it directly from your inbox (simple pattern match) and types it. The AI model never sees it. | None |
| **Authenticator app** | For tier B sites you choose, the code's setup key goes in the assistant vault and the broker generates codes. | None |
| **Push approval** (e.g. Amex app) | The assistant waits while you approve on your phone | One tap |

**Not doing:** auto-forwarding all your text messages to the server. It would expose every code from every service, including your bank's.

---

## 7. Payment cards and Apple Pay

**Apple Pay** needs your Face ID on your own device, so it can't be automated and shouldn't be. When a site takes Apple Pay, the assistant gets everything ready and sends you the link to finish.

**A card the assistant can use itself** (for bills and renewals that don't take Apple Pay):

| Option | Hard spending cap? | Notes |
|---|---|---|
| **1. Virtual card service funded from Amex Rewards Checking** (e.g. Privacy.com) | ✅ Per card, locked to one merchant | Best protection: a stolen number only works at one merchant, up to its limit. **To verify:** whether the service accepts Amex Rewards Checking as a funding account. |
| **2. Amex Additional Card with a spending limit** | ✅ Monthly limit, as low as $200, set in the Amex app | Must be issued to a person (13 or older), and you're liable even above the limit. |
| **3. Your regular Amex number** | ❌ | Only with instant Amex alerts + broker policy limits. Not recommended for anything automatic. |

**Recommendation:** option 1 if the funding check works, otherwise option 2.

Either way:

- Card numbers live **only in the assistant vault** and are typed in by the broker. They are never stored in the database or shown to the AI model.
- **Every charge is reconciled.** Each charge on the assistant's card is matched against the assistant's own action log. Any charge it didn't make gets an immediate alert.

---

## 8. Broker guardrails (code, not prompts)

1. **Site binding:** each secret works only on the site(s) it was saved for. A lookalike domain (`duke-energy-billing.com`) is refused and triggers an alert. This is how password managers stop phishing, and it also stops prompt injection.
2. **Task binding:** a secret is released only for a task that you or a scheduled job started, never because a web page or email asked for it.
3. **Approval:** the first use of each stored secret, and every use of high-sensitivity data, needs Face ID approval.
4. **Masking:** password, card and code fields are blacked out in anything sent to the model. Outgoing text is scanned for anything that looks like a secret.
5. **Short-lived:** secrets are held in memory for seconds and never logged. Browser sessions are saved encrypted, so logins are rare in the first place.
6. **Audit:** every use of a secret is logged (which secret, which site, which task, when) and listed in the weekly digest.
7. **Kill switch:** one command revokes the vault token, clears saved browser sessions and pauses all actions.

---

## 9. Sensitive personal data (not passwords)

| Sensitivity | Examples | Storage | What the AI model sees |
|---|---|---|---|
| **Low** | Address, VIN, plate, vehicle details | Normal memory | Everything |
| **Medium** | Driver's license number, date of birth, insurance policy numbers | Encrypted field | Masked unless it's needed to answer you |
| **High** | SSN, passport number | Assistant vault only. The broker types it into forms after Face ID approval each time. | "SSN on file (•••• 1234)" |

---

## 10. Example: "Renew the registration on the truck"

1. The assistant opens the DMV renewal site in its sandboxed browser.
2. Login → the DMV session saved from last time is still valid, so no login is needed. If it has expired, you get *"[Enter securely]"* and fill the login from Apple Passwords with Face ID. The model sees "Filled ✓".
3. The DMV texts a code → you get *"[Enter securely]"* → Face ID → iOS fills in the code → done.
4. The form needs your license number → the broker types it (medium sensitivity, masked to the model).
5. Payment → you get *"Pay $86.50 to the DMV for the truck's registration? [Approve with Face ID]"* → the broker types the assistant's virtual card number.
6. Confirmation page screenshot saved to the audit log. Registration due date updated in memory. The virtual card charge is reconciled.

What you did: **two Face IDs** (three if the DMV session had expired). The AI model never saw a password, code, license number or card number.

---

## 11. Build plan

| Phase | Credential work |
|---|---|
| **0 — Foundations** | Broker skeleton, encrypted saved browser sessions, audit log, masking of model inputs and outputs, kill switch |
| **1 — Awareness** | Tier A connections: Amex via Plaid (OAuth), email accounts, calendar |
| **3 — Actions** | Optional assistant vault (1Password service account) for hands-free sites, Secure Entry page (passkeys, on-device encryption, code autofill), broker typing into the browser, site and task binding, virtual card setup and charge reconciliation |

## 12. Open questions

1. ~~**Password manager:**~~ Apple Passwords. Logins go through Secure Entry with Face ID. A small 1Password vault is optional later for sites you want handled hands-free (§4).
2. Would you set up a **virtual card service** (option 1) or an **Amex Additional Card** (option 2) for the assistant?
3. Do you have a **separate business Amex** or bank account that should stay out of scope (or be handled separately)?
