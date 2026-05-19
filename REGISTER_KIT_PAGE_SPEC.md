# EvoDNA /register-kit/ UI and behaviour spec

Version: v1.3
Date: 2026-05-19
Owner: medlitsolutions-max (Product & UX)

Pairs with:
- "EvoDNA Kit Registration v1.4 – CEO Briefing" (PowerPoint)
- "EvoDNA – Developer Build Instructions – Kit Registration v1.3" (PDF)
- "EvoDNA – Partner Setup Checklist – Kit Registration v1.3" (PDF)
- `README.md` and `PULL_REQUEST_PLAN.md` in this repo

---

## 1. Purpose

`/register-kit/` is a **WordPress-facing entry point** for customers who type the URL printed on the box or instruction card.

It is **not** the primary place where tube barcodes are registered.
Actual kit registration (tube barcode + recipient labels) happens inside the my-dna SPA via `POST v1kitregister` and the `KitPicker` component.

`/register-kit/` does three things only:

1. Verify the kit belongs to the person activating it (kit number + email).
2. Capture acceptance of Terms of Service and Privacy Policy for genetic testing.
3. Issue a **single-use, short-lived activation token** and email the customer a deep-link into my-dna, where they complete registration.

---

## 2. URL and routing

- Public URL: `https://evo-dna.com/register-kit/` (already live).
- Implementation: WordPress page template in the `evodna-mydna-portal` plugin, similar pattern to the existing thank-you redirect class.
- New file to add (suggested):
  - `mydna-portal/wordpress/evodna-mydna-portal/includes/class-evodna-register-kit-page.php`
- Bootstrap: require this file from `evodna-mydna-portal.php` in the same way as `class-evodna-thankyou-redirect.php`.

---

## 3. Page layout (UX)

Re-use the current visual style shown at `/register-kit/` (existing mobile screenshot in the owners binder). The structure is:

### 3.1 Hero

- Title: **Activate your EvoDNA kit**
- Supporting text:
  > "You're in the right place. We'll link your kit to your EvoDNA account and send you to your secure my-dna portal to finish registration."

Optional info banner at top of card:

> "Please provide your kit number and checkout email so we can verify your purchase."

### 3.2 Form fields

**Field 1 – Kit number**

- Label: `Kit number (from your box)`
- Input type: text
- Placeholder: `EVO-2026-04081 or KIT123456`
- Helper: `Printed on the outside of your kit box or instruction card.`

Notes:

- This is an **order-level or kit-token identifier**, not the tube barcode.
- It must be something we can resolve to the WooCommerce order and/or the DynamoDB KitsTable row without exposing tube barcodes in WordPress.

**Field 2 – Email address**

- Label: `Email address used at checkout`
- Input type: email
- Placeholder: `you@example.com`
- Helper: `We'll send a secure link to this email so you can finish registration in your my-dna portal.`

### 3.3 Terms and privacy

Two required checkboxes:

1. `[ ] I agree to the Terms of Service for genetic testing.`
   - Link target: existing Terms of Service URL.

2. `[ ] I have read the Privacy Policy and understand how my genetic data is stored and used.`
   - Link target: existing Privacy Policy URL.

Below checkboxes, keep the existing orange notice:

> "Important: Your genetic data will be securely stored and used only for generating your personalized reports."

### 3.4 Primary CTA

- Button label: `Send me my secure activation link`
- State: disabled until both fields are non-empty and both checkboxes are ticked.

**Success state (on this page):**

Show a confirmation message in the card:

> "Thank you. If the kit number and email match our records, we've emailed you a secure link to finish registering your kit in your EvoDNA portal.
> Please check your inbox and spam folder."

No further data is shown for privacy reasons.

---

## 4. Backend behaviour

`/register-kit/` **does not** call `v1kitregister` directly and never sees tube barcodes.

### 4.1 New backend endpoint: POST /v1/kit-activation-request

Implement a small customer-facing API endpoint in the existing customer API stack (same pattern as `v1kitregister` and `v1ordersync`).

- Method: `POST /v1/kit-activation-request`
- Auth: public endpoint with HMAC or ReCaptcha (to be decided); rate-limit by IP to avoid abuse.
- Request JSON:

```json
{
  "kitNumber": "EVO-2026-04081",
  "email": "you@example.com"
}
```

- Response JSON (always 200, see error handling):

```json
{
  "ok": true
}
```

### 4.2 Lambda behaviour (pseudocode)

High-level behaviour inside the new Lambda:

1. Normalise `kitNumber` and `email` (trim, lower-case email).
2. Look up the order row (Aurora / Woo mirror) by `orderId` or order metadata matching `kitNumber` and `email`.
3. If there is **no match**:
   - Log the attempt with reason `NO_MATCH`.
   - Still return `{ "ok": true }` to the client (do not leak whether the combination exists).
   - Do **not** send an email.
4. If there **is a match**:
   - Generate a short-lived JWT or random token (for example 32 bytes) with claims:
     - `sub` = Cognito user id (if already exists) or the email.
     - `orderId` = identified order.
     - `issuedAt`, `expiresIn` (for example 30 minutes).
   - Store a record of the token in DynamoDB or use a signed JWT with a secret in AWS Secrets Manager.
   - Compose an email "Finish registering your EvoDNA kit" with a link:

     `https://my-dna.evo-dna.com/?activationToken=...`

   - Send email via SES or via GoHighLevel using the existing inbound webhook URL for customer messages.
   - Return `{ "ok": true }`.

Error conditions (database down, email provider failure) should surface to logs and metrics but return a generic 500 to the WordPress caller so we can display a generic error banner.

### 4.3 WordPress – API integration

In `class-evodna-register-kit-page.php`:

- On form POST:
  - Validate required fields server-side.
  - Call `POST /v1/kit-activation-request` with `kitNumber` and `email`.
  - If 200 with body `{ "ok": true }`: show the confirmation message.
  - If non-200: show a generic error at top of card:

    > "Something went wrong on our side. Please try again in a few minutes or contact us at info@evo-dna.com."

### 4.4 Email UX ownership

All customer-facing emails sent from the `/register-kit/` flow
("Finish registering your EvoDNA kit" and any follow-up messages)
must use copy provided by **medlitsolutions-max**.

Developers must:

- Expose a template or configuration file for the email subject and body.
- Use `info@evo-dna.com` as the visible "from" or "reply-to" address (implementation depends on SES / GHL constraints).
- Avoid hard-coding email text inside Lambda functions.

---

## 5. my-dna SPA behaviour with activationToken

This reuses the kit registration flow defined for the my-dna dashboard (single-kit and family-pack, using `KitPicker.tsx`):

When the user clicks the emailed link:

1. The my-dna SPA reads `activationToken` from the URL.
2. Frontend calls `POST /v1/kit-activation-consume` (new endpoint) with the token.
3. Backend Lambda verifies the token:
   - If valid:
     - Ensure the user exists in Cognito, creating the account if needed.
     - Return whatever is required for the SPA to complete sign-in (for example a Cognito auth code or a presigned login URL).
     - Return the relevant `orderId` so the SPA can fetch kits via the existing `/v1/kits?orderId=...` endpoint.
   - If invalid or expired: return 401 so the SPA can show a "link expired" message with a link back to `/register-kit/`.

4. Once signed in, the SPA navigates to the kit registration screen (KitPicker):
   - Single-kit: one tile to register the tube barcode.
   - Family pack: multiple tiles (Adult 1 / Adult 2 / Child) with editable `recipientLabel` per tube.

No changes are required to `v1kitregister` or `KitPicker.tsx` beyond what is already planned in the v1.3 build.

---

## 6. Security and privacy constraints

- `/register-kit/` and `POST /v1/kit-activation-request` **must never expose tube barcodes, tube lots, sample IDs, or lab requisition IDs** in WordPress templates, JavaScript, or logs.
- Tokens must be single-use and short-lived; expired or used tokens are rejected at `kit-activation-consume`.
- Error messages to the user must not reveal whether a specific kit/email combination exists (prevents kit enumeration and guessing).
- All kit registration that touches tube barcodes must happen server-side in AWS via `POST v1kitregister` only, inside the my-dna SPA flow.

---

## 7. Acceptance criteria (dev)

1. `/register-kit/` renders with the fields and copy described in Section 3.
2. Submitting valid-looking data with both checkboxes ticked calls `POST /v1/kit-activation-request`.
3. On 200 with `{ "ok": true }`, the page shows the success message and does **not** echo any PII back to the browser.
4. When a known kit/email combination is used in a test environment:
   - An email is delivered with a working my-dna activation link.
   - Clicking the link signs the user into my-dna (or prompts them to set a password) and lands them on the kit registration screen for the correct order.
5. When an unknown kit/email combination is used:
   - The user still sees the generic success message, but no email is sent.
6. `/register-kit/` has no references to tube barcodes, sample IDs, or lab requisition IDs in HTML, JS, PHP, or application logs.

---

## References

- EvoDNA Kit Registration v1.4 – CEO Briefing (PowerPoint, "Kit-Registration-Plan.pptx").
- EvoDNA – Developer Build Instructions – Kit Registration v1.3 (PDF, "EvoDNA-Developer-Build-Instructions.pdf").
- EvoDNA – Partner Setup Checklist – Kit Registration v1.3 (PDF, "EvoDNA-Partner-Setup-Checklist.pdf").
- EvoDNA Kit Registration v1.3 developer source mirror – `README.md`, `PULL_REQUEST_PLAN.md`.
