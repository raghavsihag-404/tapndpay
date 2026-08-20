# TapConfirm
### Phone-Based Step-Up Authentication for Contactless (Tap-to-Pay) Card Fraud

**One-liner:** When a tap-and-pay transaction looks risky, the cardholder's own phone asks *"Is this you?"* — and nothing gets charged until a fingerprint says yes.

---

## 1. The Problem

Contactless cards let people pay without a PIN or signature below a country-specific "Cardholder Verification Method" (CVM) limit — historically $50–$100 in the US, and up to £100 in the UK. That convenience is exactly what fraudsters exploit.

A technique known as **"Ghost Tap"** has surged roughly 35x since early 2025. Instead of physically cloning a card, attackers steal a card's NFC data through phishing or malware, then relay it in real time to a mule or device anywhere in the world — letting that person shop under the no-verification limit while the real card never leaves the owner's pocket. Because each transaction looks like an ordinary small in-store purchase with a valid NFC handshake, standard fraud detection (which watches for unusual amounts or locations) doesn't catch it — the cardholder usually only finds out when reviewing their statement.

To make matters harder, regulation is moving toward *larger, more flexible* contactless limits rather than smaller ones. In the UK, the FCA is letting banks set their own contactless caps from March 2026 instead of the fixed £100 ceiling. That means the amount of money a fraudster can take with **zero live verification** is likely to grow, not shrink.

**The gap:** the chip and the network are secure. What's missing is a live, real-time check that the person tapping the card is the person who actually owns it — for exactly the transactions that today skip PIN/OTP entirely.

---

## 2. The Idea

**TapConfirm** adds a second factor to *contactless* payments specifically — the ones currently allowed to clear with no PIN and no OTP. When a tap transaction is flagged as risky, the issuing bank sends a real-time push notification to the cardholder's registered phone. The phone shows the merchant, amount, and location, and asks for a fingerprint or Face ID confirmation. The payment stays pending until that confirmation arrives (or times out).

No PIN pad. No OTP to type at checkout. The second factor lives entirely on a device the fraudster doesn't have.

---

## 3. User Flow

1. Card is tapped at a POS terminal.
2. The issuer's fraud/risk engine scores the transaction in real time.
3. **If flagged**, instead of auto-approving, the issuer sends a push notification to the cardholder's phone via the bank's app.
4. The phone displays merchant name, amount, and location, with an "Approve" fingerprint prompt.
5. The cardholder authenticates locally (fingerprint / Face ID, processed inside the phone's secure enclave — biometric data never leaves the device).
6. A signed approval token is sent back to the issuer.
7. **Approve →** transaction clears. **Deny or timeout →** transaction is declined, and the card can be auto-frozen pending review.

---

## 4. Risk-Based Triggers (why not *every* tap?)

Prompting on every single tap would kill the "tap and go" experience. TapConfirm only steps up authentication when the risk engine flags something, such as:

- First time paying this merchant, or a new merchant category
- GPS location of the phone doesn't match the terminal's location
- Unusual transaction frequency (several taps in quick succession)
- Amount sits just under the no-verification limit, repeatedly
- New or unrecognized device/SIM associated with the card
- First international transaction on the card

---

## 5. System Architecture (high level)

```
POS Terminal
     │  (NFC tap)
     ▼
Acquirer → Card Network (Visa/Mastercard/RuPay rails)
     │
     ▼
Issuer Fraud & Risk Engine ── scores transaction
     │
     ├── Low risk → auto-approve (today's normal flow, unchanged)
     │
     └── Flagged → Push Notification Service (FCM/APNs)
                        │
                        ▼
                 Bank App on Registered Phone
                        │
                        ▼
               Secure Enclave Biometric Prompt
                        │
                        ▼
              Signed Approval/Denial Token
                        │
                        ▼
              Back to Issuer → Approve/Decline
                        │
                        ▼
                  Result to Terminal
```

---

## 6. My Additions (add-ons beyond the core idea)

**Smarter triggering**
- *Risk scoring, not blanket prompts* — described above; keeps 95%+ of genuine taps instant.
- *Merchant-category-based thresholds* — a grocery store tap and an electronics-store tap don't carry the same risk; tune trigger sensitivity per category.

**Graceful degradation**
- *Trusted zones / trusted merchant list* — user pre-approves recurring locations (home, gym, regular coffee shop) so they're not pinged every day.
- *Offline fallback* — if the phone is unreachable (dead battery, no signal, left at home during a workout), the user chooses in advance whether unreachable = fall back to a standard PIN prompt at the terminal, or default-decline for anything above a set risk score.
- *Wearable approval* — let a paired smartwatch approve when the phone itself isn't on hand.

**Security hardening**
- *Silent duress signal* — instead of a plain "deny," give the user a way to reject that also silently flags the bank's fraud team and auto-freezes the card, useful if the card was actually stolen and someone is trying repeatedly.
- *Location correlation as an automatic flag* — cross-check phone GPS vs. terminal location server-side and raise risk score automatically on mismatch, even before the user responds.
- *FIDO2/WebAuthn-style signing* — only a cryptographically signed attestation leaves the device, never raw biometric data, which also helps with PCI-DSS/GDPR-style compliance questions.

**Accessibility**
- *Trusted contact fallback* — for elderly or vulnerable cardholders, let a family member also receive (and optionally approve) the prompt, or offer a large-text/voice-guided approval flow.
- *Backup device / PIN fallback* — a lost or broken phone shouldn't strand the cardholder; allow a secondary registered device or an in-app-verified backup PIN.

**Demo-ready extras**
- *Fail-safe default* — if the prompt times out, default to **decline**, not approve. Judges will ask about this; a default-deny stance is the correct security posture.
- *Analytics dashboard* — a simple screen showing flagged vs. auto-approved transactions is an easy, visually strong addition for a demo.

---

## 7. The Honest Challenge (and how to address it)

The hardest technical constraint isn't the biometric — it's **time**. EMV contactless transactions are engineered to clear in roughly half a second to a couple of seconds so the "tap and walk away" experience holds up; a full push-notification → biometric → response round trip over a mobile network realistically takes several seconds longer than that.

This is exactly why TapConfirm should **not** try to gate every transaction — only the small percentage flagged by the risk engine. For those, the issuer can use a short authorization hold ("stand-in" processing, similar to how some issuers already hold or soft-decline unusual transactions today) to buy the extra seconds needed, rather than trying to force the whole payment rail to wait on every tap. This is conceptually the tap-to-pay equivalent of 3D Secure step-up authentication, which already exists for card-not-present (online) purchases — TapConfirm extends the same step-up idea to card-present contactless payments, which is the genuinely new part.

Being upfront about this trade-off (speed for the 95% vs. security for the flagged 5%) is worth stating explicitly in a hackathon pitch — it shows the team understands the real constraint rather than hand-waving past it.

---

## 8. Suggested Tech Stack for a Hackathon Build

| Layer | Suggestion |
|---|---|
| Biometric auth | Android `BiometricPrompt`, iOS `LocalAuthentication` (Face ID/Touch ID) |
| Signed assertion | FIDO2 / WebAuthn |
| Push notifications | Firebase Cloud Messaging (Android/web) or APNs (iOS) |
| Risk/rules engine | Lightweight Node.js or Python service — hardcoded rules for the demo, with a clear upgrade path to an ML anomaly model |
| Transaction state | Redis (fast, short-lived pending-transaction store) |
| Payment simulation | You won't have real card-network access — build a simple POS simulator (web page with a "Tap Card" button) and simulate an ISO 8583-style authorization message, or wire it through a sandbox like Stripe/Razorpay/UPI test mode |

---

## 9. MVP Build Order (for a ~24–36 hr hackathon)

1. POS simulator — a web page that fires a "transaction event" on button press
2. Risk engine — hardcode a demo rule (e.g., always flag the 2nd tap within 10 seconds, or any tap over a demo threshold)
3. Push notification to a second phone/browser tab acting as the cardholder's device
4. Biometric/WebAuthn approval screen
5. Approve/deny result relayed back to the POS simulator in real time
6. A simple dashboard/log showing auto-approved vs. flagged-and-confirmed vs. declined transactions — strong visual for judges

---

## 10. Why This Matters (impact framing for judges)

Ghost Tap-style relay fraud has grown roughly 35x since early 2025 and has already cost a Fortune 100 US financial institution several million dollars in a single quarter of documented incidents. It works precisely because contactless payments below the CVM limit currently have **no live verification step at all**. TapConfirm targets that exact gap without asking regulators to lower contactless limits or forcing every cardholder back to typing a PIN.

---

## 11. Future Roadmap

- Pilot with a single issuing bank on a subset of cards
- Move the rules engine to an adaptive ML risk model trained on real approve/decline/dispute outcomes
- Expand device-approval options (wearables, trusted-contact co-approval)
- Explore issuer-side "stand-in" authorization holds as a standard mechanism for any step-up flow, not just this one

---

## Sources
- Resecurity — [NFC Fraud Wave: Evolution of Ghost Tap on the Dark Web](https://www.resecurity.com/blog/article/nfc-fraud-wave-evolution-of-ghost-tap-on-the-dark-web)
- Lock.pub — [NFC Ghost Tap: The Contactless Payment Fraud Surging in 2025–2026](https://lock.pub/en/blog/nfc-ghost-tap-payment-fraud)
- American Military University — [The Risks of Contactless Payment Are High Despite Security](https://www.amu.apus.edu/area-of-study/business-administration-and-management/resources/the-risks-of-contactless-payment-are-high-despite-security/)
- Northey Point — [UK Contactless Payment Limits Are About to Get a Shake-Up](https://northeypoint.substack.com/p/uk-contactless-payment-limits-are)
