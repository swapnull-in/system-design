# Mobile 06 — Security & Privacy on Device

> **Sources:** Android Keystore / iOS Keychain & Data Protection docs · OWASP MASVS (the mobile checklist standard) · [Core Course 13 — Security](../Core%20Course/13-security.md) for the backend half.
> **Role in the track:** the EM/legal lens made technical. Every case study has one section that is really this note — [15](15-stock-trading-app.md)'s trading, [26](26-spotify-music.md)'s DRM, [20](20-analytics-sdk.md)'s consent, [22](22-push-notification-system.md)'s token-PII.

## What you'll master

- The **mobile threat model** — four adversaries, each defeating a different control.
- The **secure-storage hierarchy**: hardware keystore → encrypted storage → "not on the device at all."
- **Network trust**: TLS defaults, when pinning earns its operational cost, and the pinned-cert outage story.
- **The APK is public** — the one sentence that reorganizes where secrets and logic live.
- **Privacy as architecture**: data minimization, consent gating, and the deletion path — the parts regulators actually audit.

## 1. The threat model — four adversaries

| Adversary | Can do | Primary control |
|---|---|---|
| **Device thief / borrower** | Read unlocked apps, extractable storage | Platform encryption-at-rest + biometric/passcode gates on sensitive screens ([15](15-stock-trading-app.md)) |
| **Network attacker** (hostile Wi-Fi, MITM) | Intercept/modify traffic | TLS everywhere + platform trust store; pinning only where warranted (§3) |
| **Reverse engineer** | Decompile the APK/IPA, extract anything shipped in it | Ship no secrets; server-side authorization; obfuscation as speed-bump only |
| **The curious app itself** (your bugs, your SDKs) | Over-collect, leak via logs/analytics/third-party SDKs | Data minimization, consent gates ([20 §3](20-analytics-sdk.md)), SDK vetting, log hygiene |

The Staff framing: *name which adversary each control addresses.* "We encrypt the DB" defeats the thief, does nothing against the reverse engineer; "we obfuscate" inconveniences the reverse engineer and defeats no one. Controls without named adversaries are theater.

## 2. Secrets & storage — the hierarchy

- **Keys live in hardware** (Keystore/StrongBox, Secure Enclave): generated there, non-exportable, operations happen inside. App-layer crypto keys wrap through them; biometric-bound keys give you "auth to use, not auth to see."
- **Data encrypts at rest by platform default** — full-disk/file-based encryption covers the baseline; add app-layer encryption (SQLCipher-style, or encrypted values in a plain DB) for the *sensitive subset* — tokens, financial data, messages — because platform encryption doesn't protect against the unlocked-device adversary.
- **The ladder's top rung is absence**: the most secure storage for a card number is the PSP's vault, not your device ([24](24-hotel-reservation.md)/[29](29-ecommerce-checkout.md)'s build-vs-buy) — for every sensitive datum, first ask *"why is this on the device at all?"*
- **Tokens, concretely:** short-lived access token in memory, refresh token in Keystore-wrapped storage, rotate on refresh, revoke server-side on logout/device-loss — and the session table server-side is the real control ([22 §3](22-push-notification-system.md)'s unregister-before-logout ordering applies to auth too).
- **The APK is public.** Anything shipped in it — API keys, endpoints, feature logic, "hidden" admin flags — is readable by anyone with 20 minutes. Consequences: client API keys are *identifiers not secrets* (real authz is server-side per-user), premium gating is server-enforced, and obfuscation is a speed bump you don't budget security on.

## 3. Network trust — pinning is a trade, not a default

- **Defaults first:** TLS everywhere, platform trust store, no cleartext (network security config / ATS enforce it), modern cipher policy from the OS. This is table stakes and one sentence in the room.
- **Certificate pinning** defends against compromised/rogue CAs — real but rare. Its cost is operational and severe: a botched rotation **bricks the app's network with no server-side fix, for the full tail of shipped versions** (the [07](07-release-engineering.md) no-rollback physics applied to certs). Decision rule: pin for banking/trading-grade threat models, with pin sets (current + backup), enforced rotation runbooks, and a remote kill switch for the pin itself ([21](21-ab-feature-flag-sdk.md)'s config lane); skip it for consumer apps where platform trust + transparency logs are proportionate.

## 4. Privacy as architecture (the EM's half)

- **Minimization is a design input, not a review finding:** every field collected needs a named purpose; analytics run allowlists not "log everything" ([20 §5](20-analytics-sdk.md)'s taxonomy registry); location gets the coarsest grain the feature survives ([14 §6](14-ride-hailing.md)'s background-location policy).
- **Consent gates the pipeline at the source** — events don't enter the queue before consent state allows ([20 §3](20-analytics-sdk.md)); regional regimes (GDPR/DPDP/CCPA) become *configuration of the same gate*, not forked builds.
- **The deletion path is a feature with an SLA:** account deletion reaches device caches, server stores, backups, and third-party SDKs — app stores now require it, and "we can't delete from the analytics vendor" is a contract you signed, not an excuse. Design the data inventory *first*; deletion falls out of knowing where data lives.
- **Third-party SDKs are your data processors:** every SDK ships with your app's permissions and your privacy label. Vet what they collect, pin their versions, and own the answer — "the ad SDK did it" is your incident.
- **On-device processing as a privacy feature:** ML inference, content scanning, smart replies computed locally turn a privacy review from "we upload everything" into "nothing leaves" — increasingly the differentiating answer ([Core Course 17](../Core%20Course/17-ai-systems.md) has the server side).
- **The platform privacy-compliance surface (2024+, mandatory — name it or read as pre-2024):** iOS **App Tracking Transparency** — you must request permission before tracking across apps / using the **IDFA**, and most users decline, so a design that *depends* on cross-app attribution is already broken; iOS **Privacy Manifests + required-reason APIs** — you (and every SDK you bundle) must declare data collection and justify use of certain APIs, or the App Store rejects the build; **Play Data Safety** — the Android equivalent declaration. The design consequence: **your third-party SDKs' privacy manifests are now your compile-time problem** (an SDK that doesn't declare gets your whole app rejected), which sharpens the "SDKs are data processors" point above from a policy into a build gate.

> **Staff-level insight:** privacy questions in the EM round are *risk-pricing* questions. The strong shape: name the data classes you hold, the adversary per control, the regulatory regimes as config, and the one-line posture — *"minimize at collection, encrypt the sensitive subset, buy the vaults, and make deletion a tested code path."*

## When NOT

- **Don't roll crypto** — platform primitives and vetted libraries only; custom crypto is how "encrypted" becomes a euphemism.
- **Root/jailbreak detection** is risk *signal*, not a control (defeatable by definition) — use it to step up verification for money apps, never as the security model.
- **Don't pin certs** without a rotation runbook and a remote un-pin path — the outage you cause outweighs the CA you feared.

---

## Interview Cheat-Sheet

**30-second version:** four adversaries (thief / network / reverse-engineer / your own SDKs), a named control each · keys in hardware keystore, sensitive subset app-layer-encrypted, best storage is absence (buy the vault) · TLS by default, pinning only with a rotation runbook + remote un-pin · the APK is public: client keys are identifiers, authz is server-side · privacy: minimize at collection, consent gates the pipeline, deletion is a tested path, SDKs are your processors.

| Probe | Answer |
|---|---|
| "Where do you store the auth token?" | Access token in memory, refresh token Keystore-wrapped; rotation on refresh, server-side revocation is the real control |
| "Do you pin certificates?" | Threat-model-priced: yes for banking-grade with pin sets + runbook + remote un-pin; no for consumer apps — a botched pin is a self-inflicted, unfixable outage across every shipped version |
| "Can users find your API key in the APK?" | Yes — which is why it's an identifier, not a secret; anything worth protecting is authorized server-side per user |
| "GDPR deletion request — walk me through it" | Data inventory → device caches + server + backups + SDK processors, on an SLA, verified by test — deletion is a code path, not a ticket |

**Say-verbatim:**
- *"Every control gets a named adversary — a control without one is theater."*
- *"The APK is public. I ship identifiers, not secrets."*
- *"The most secure storage is absence — first question: why is this on the device at all?"*
- *"Minimize at collection, encrypt the sensitive subset, buy the vaults, and make deletion a tested code path."*
