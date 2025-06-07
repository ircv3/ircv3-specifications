---
title: Privacy Policy extension
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

Software implementing this work-in-progress specification MUST NOT use the unprefixed account-registration capability name. Instead, implementations SHOULD use the draft/privacy capability name to be interoperable with other software implementing a compatible work-in-progress version.
The final version of the specification will use an unprefixed capability name.

## 1. Motivation

This draft introduces a privacy consent workflow for IRC servers and clients. It allows clients to render a structured privacy policy UI via the `draft/privacy` capability. Users can explicitly consent to the privacy policy via the `/CONSENT` command. Consent MAY be required for specific actions (e.g. account registration) or for general usage, depending on server configuration.

## 2. ISUPPORT Token

Servers supporting this specification MUST advertise one of the following ISUPPORT tokens:

### 2.1 Optional Privacy Consent

```
005 PRIVACY
```

- Consent is **supported**, but **not required** to connect.
- The server MAY still require consent before specific actions (e.g. `/REGISTER`, `/LOGIN`, joining private channels, etc.).
- Clients can opt-in to show the privacy policy and allow proactive consent.

### 2.2 Required Privacy Consent

```
005 PRIVACY=consent-required
```

- Consent is **mandatory** before the user can interact with the network.
- Users must explicitly accept the policy via `/CONSENT` or will be prompted accordingly.

## 3. Capability Registration

Clients MAY negotiate the following IRCv3 capability:

```
draft/privacy
```

This enables structured delivery of privacy policy content using `NOTE PRIVACY` messages, suitable for modal rendering.

## 4. Server Behavior

### 4.1 On Connection

Upon user connection, the server MUST check whether the user has already consented to the privacy policy.

- If `PRIVACY=consent-required` is set and the user has **not yet consented**, the server MUST prompt them.
- If only `PRIVACY` is present, the server MAY wait until a restricted command (e.g. `/REGISTER`) is issued before prompting.

#### With `draft/privacy`:

```
:irc.example.com NOTE PRIVACY :Privacy information:
:irc.example.com NOTE PRIVACY :This service collects and processes personal data in accordance with our privacy policy.
:irc.example.com NOTE PRIVACY :You can read the full policy at: https://example.com/privacy-policy
:irc.example.com NOTE PRIVACY :By using this service, you consent to the collection and processing of your personal data as described in the policy.
:irc.example.com NOTE PRIVACY :If you do not agree with the terms, please disconnect from the service.
:irc.example.com NOTE PRIVACY :To accept the privacy policy, click Accept or type /CONSENT
```

#### Without `draft/privacy`:

```
:irc.example.com NOTICE <nick> :You must read and accept our privacy policy before continuing. View it at https://example.com/privacy-policy — type /CONSENT to accept.
```

### 4.2 Consent-Gated Commands

When `PRIVACY` is set (without `=consent-required`), the server MAY reject specific commands until the user consents.

Example behavior:

```irc
REGISTER valware hunter2 valware@example.com
:irc.example.com NOTICE valware :You must first consent to our privacy policy before registering. Type /CONSENT or visit https://example.com/privacy-policy.
```

## 5. Client Behavior

Clients SHOULD behave as follows:

- If `PRIVACY` is present:
  - Be prepared to show a privacy consent flow when requested.
  - Optionally show a “Review and Accept Privacy Policy” button in registration/login UIs.

- If `PRIVACY=consent-required` is present:
  - Require user interaction with the policy before proceeding with any network activity.
  - Send `/CONSENT` after acceptance.

## 6. Commands

### 6.1 `CONSENT`

```
CONSENT
```

Indicates the user agrees to the server’s privacy policy.

**Server MUST:**

- Persistently record the user's consent (by hostmask, IP, or account)
- Suppress future prompts
- Respond with:

```
:irc.example.com CONSENT SUCCESS * :Thank you. You may now use the service.
```

**Server MAY:**

- Gate further commands until consent is provided (see ISUPPORT behavior)
- Treat repeated `CONSENT` as idempotent, replying with:

```
:irc.example.com NOTE CONSENT ALREADY_CONSENTED * :You have already accepted the privacy policy.
```

### 6.2 `CONSENT REVOKE`

```
CONSENT REVOKE
```

Used by the user to explicitly withdraw their prior consent.

**Server MUST:**

- Remove or mark the consent as revoked
- Reply with:

```
:irc.example.com CONSENT SUCCESS * :Your consent has been revoked. You will be prompted again when required.
```

**Server MAY:**

- Immediately restrict user access to features requiring consent
- Disconnect the user if consent is mandatory at all times (`PRIVACY=consent-required`), replying with:

```
:irc.example.com CONSENT ERROR * :Consent is required to use this service. Disconnecting.
ERROR :Consent revoked
```

**Client MAY:**

- Expose a “Revoke Consent” button in settings
- Automatically re-prompt the user when needed

### 6.3 Example Message Types

- `CONSENT SUCCESS` — for completed operations
- `NOTE CONSENT` — for informational messages
- `FAIL CONSENT` — for denials or issues

## 7. Storage and Revocation

- Consent SHOULD be stored persistently.
- If the privacy policy changes substantially, the server MAY revoke previous consent and prompt again.
- Consent SHOULD be tied to user accounts when possible.

## 8. Security and Privacy Considerations

- Clients MUST NOT auto-send `/CONSENT` without user interaction.
- Servers MUST maintain an accessible and accurate privacy policy URL.
- Consent-gating MUST be clearly communicated to the user.

## 9. Example Flows

**A. With `PRIVACY=consent-required` and `draft/privacy`:**

```
:irc.example.com NOTE PRIVACY POLICY :Privacy information:
:irc.example.com NOTE PRIVACY POLICY :Etc
...
:irc.example.com NOTE PRIVACY POLICY_END :End of PRIVACY
[User clicks Accept]
CONSENT
:irc.example.com CONSENT SUCCESS valware :Thank you. You may now use the service.
```

**B. With keyless `PRIVACY`, consent required before registration:**

```
REGISTER valware weewoo123 valware@example.com
:irc.example.com NOTE REGISTER CONSENT_REQUIRED * :You must first consent to our privacy policy before registering.
:irc.example.com NOTE REGISTER CONSENT_REQUIRED :Type please check visit https://example.com/privacy-policy and type /CONSENT.
CONSENT
:irc.example.com CONSENT SUCCESS valware :Thank you. You may now register an account.
```

