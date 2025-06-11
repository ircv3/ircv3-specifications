---
title: Policy Consent Extension
layout: spec
work-in-progress: true
copyrights:
-
name: "ValwareIRC"
email: "valerie@valware.co.uk"
period: "2025"
---

## Notes for implementing work-in-progress version

This specification relies on the Standard Replies framework.

Software implementing this work-in-progress specification MUST NOT use the unprefixed `draft/policy` capability name. Instead, implementations SHOULD use the `draft/policy` capability to ensure interoperability with compatible implementations.

The final version of the specification will use an unprefixed capability name.

## 1. Motivation

This draft introduces a structured policy consent mechanism for IRC servers and clients. It enables clients to present a policy UI via the `draft/policy` capability and allows users to explicitly agree to the terms using the `/CONSENT` command. Consent MAY be required for specific actions (e.g. account registration) or for general usage, depending on server configuration.

## 2. Capability Advertisement

Servers supporting this specification MUST advertise the following IRCv3 capability with an optional value:

- `draft/policy` – Consent supported, but not required
- `draft/policy=consent-required` – Consent is mandatory to use the service

### 2.1 Optional Policy Consent

```
CAP LS
draft/policy
```

- Consent is **supported**, but **not required** to connect.
- The server MAY still restrict specific commands (e.g. `/REGISTER`) until consent is granted.

### 2.2 Required Policy Consent

```
CAP LS
draft/policy=consent-required
```

- Consent is **required** before the user may proceed.
- The user MUST explicitly agree to the policy via `/CONSENT`.

## 3. Server Behavior

### 3.1 On Connection

Servers MUST check if the user has previously provided consent.

- If `draft/policy=consent-required` is advertised and no consent exists, the user MUST be prompted before proceeding.
- If only `draft/policy` is advertised, prompting MAY be deferred until a restricted action is attempted.

#### With `draft/policy` (grouped sections):

```
:irc.example.com NOTE POLICY TOS * :Terms of Service
:irc.example.com NOTE POLICY TOS 1 :You agree to follow network etiquette.
:irc.example.com NOTE POLICY TOS 2 :You will not spam or flood.
:irc.example.com NOTE POLICY TOS * :End of Terms of Service.

:irc.example.com NOTE POLICY RULES * :Community Rules
:irc.example.com NOTE POLICY RULES 1 :Be respectful to others.
:irc.example.com NOTE POLICY RULES 2 :No harassing language.
:irc.example.com NOTE POLICY RULES * :End of Community Rules.

:irc.example.com NOTE POLICY PRIVACY * :Privacy & Data Use
:irc.example.com NOTE POLICY PRIVACY 1 :Minimal personal data is collected.
:irc.example.com NOTE POLICY PRIVACY 2 :Data is retained for 30 days.
:irc.example.com NOTE POLICY PRIVACY * :End of Privacy & Data Use.

:irc.example.com NOTE POLICY * :To accept, click “Accept” or type /CONSENT
```

#### Without `draft/policy` support:

```
:irc.example.com NOTICE <nick> :You must review and accept our policy to continue. Visit https://example.com/policy — type /CONSENT to accept.
```

### 3.2 Consent-Gated Actions

When `draft/policy` is present without `=consent-required`, servers MAY block certain commands until consent is obtained.

Example:

```
REGISTER valware hunter2 valware@example.com
:irc.example.com NOTICE valware :You must first accept our policy to register. Type /CONSENT or visit https://example.com/policy.
```

## 4. Client Behavior

Clients SHOULD behave as follows:

- If `draft/policy` is present:
- Prepare to show a policy consent flow when prompted.
- Optionally display a “Review and Accept Policy” button in the UI.
- Render **TOS**, **RULES**, and **PRIVACY** as separate sections or panels.

- If `draft/policy=consent-required` is present:
- Require user interaction with the policy before proceeding.
- Automatically send `/CONSENT` after user approval.

## 5. Commands

### 5.1 `CONSENT`

```
CONSENT
```

Indicates the user agrees to the server’s policy.

**Server MUST:**

- Persistently store the user's consent (by hostmask, account, etc.)
- Suppress further consent prompts
- Reply with:

```
:irc.example.com CONSENT SUCCESS * :Thank you. You may now use the service.
```

**Server MAY:**

- Block commands until consent is received
- Treat multiple `CONSENT` calls as idempotent, replying:

```
:irc.example.com NOTE CONSENT ALREADY_CONSENTED * :You have already accepted the policy.
```

### 5.2 `CONSENT REVOKE` Command

```
CONSENT REVOKE
```

Explicitly withdraws prior consent.

**Server MUST:**

- Revoke or mark consent as withdrawn
- Reply:

```
:irc.example.com CONSENT SUCCESS * :Your consent has been revoked. You will be prompted again when required.
```

**Server MAY:**

- Restrict or disconnect the user if consent is mandatory:

```
:irc.example.com CONSENT ERROR * :Consent is required to use this service. Disconnecting.
ERROR :Consent revoked
```

**Clients MAY:**

- Provide a “Revoke Consent” button
- Automatically re-prompt as needed

### 5.3 Message Types

- `CONSENT SUCCESS` — for successful operations
- `NOTE CONSENT` — for consent-related messages
- `FAIL CONSENT` — for failures

### 5.4 `POLICY` Command
Requests the current structured policy information from the server.

Server MUST:

- Send a series of NOTE POLICY messages grouped by type (TOS, RULES, PRIVACY)

- Ensure message order allows clients to render sections properly

- End each group with a final * line indicating the section is complete

- If the policy has already been shown recently, MAY still resend it on request

Example Response:
```
:irc.example.com NOTE POLICY TOS * :Terms of Service
:irc.example.com NOTE POLICY TOS 1 :You agree to follow network etiquette.
:irc.example.com NOTE POLICY TOS 2 :No spam or flooding allowed.
:irc.example.com NOTE POLICY TOS :Other text.

:irc.example.com NOTE POLICY RULES * :Network Rules
:irc.example.com NOTE POLICY RULES 1 :Be kind to others.
:irc.example.com NOTE POLICY RULES 2 :No illegal activity.
:irc.example.com NOTE POLICY RULES :Other text.

:irc.example.com NOTE POLICY PRIVACY * :Privacy Policy
:irc.example.com NOTE POLICY PRIVACY 1 :We collect minimal connection data.
:irc.example.com NOTE POLICY PRIVACY 2 :Logs are retained for abuse prevention.
:irc.example.com NOTE POLICY PRIVACY :Other text.

:irc.example.com NOTE POLICY * :To accept the policy, type /CONSENT
```
Client MAY:

- Trigger /POLICY when the user presses a “Review Policy” button
- Automatically call /POLICY during login flows if draft/policy=consent-required

Notes:

- The /POLICY command has no parameters
- Servers SHOULD send all three sections: TOS, RULES, and PRIVACY, even if some are empty
- A delayed or partial policy response MAY lead to invalid or rejected consent

## 6. Storage and Revocation

- Consent SHOULD be stored persistently.
- If the policy changes significantly, servers MAY revoke prior consent.
- Consent SHOULD be tied to authenticated accounts where possible.

## 7. Security and Policy Considerations

- Clients MUST NOT auto-send `/CONSENT` without user input.
- Servers MUST provide an accessible and accurate policy URL.
- Consent requirements MUST be clearly communicated to the user.

## 8. Example Flows

### A. With `draft/policy=consent-required` and UI rendering:

```
:irc.example.com NOTE POLICY TOS * :Terms of Service...
...
:irc.example.com NOTE POLICY PRIVACY * :Privacy & Data Use...
CONSENT
:irc.example.com CONSENT SUCCESS valware :Thank you. You may now use the service.
```

### B. With `draft/policy` (optional), blocking `/REGISTER`:

```
REGISTER valware hunter2 valware@example.com
:irc.example.com NOTE REGISTER CONSENT_REQUIRED * :Consent is required to register.
:irc.example.com NOTE REGISTER CONSENT_REQUIRED :Please review the policy by typing /POLICY.
CONSENT
:irc.example.com CONSENT SUCCESS valware :Thank you. You may now register an account.
```
