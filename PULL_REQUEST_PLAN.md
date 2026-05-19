# EvoDNA Kit Registration — Pull Request Plan

Version: v1.3
Owner: medlitsolutions-max
Last updated: 2026-05-19

This file tracks the ordered PRs that deliver the v1.3 / v1.4 kit registration
flow across the EvoDNA admin console, customer API, my-dna SPA, and the
WordPress `evodna-mydna-portal` plugin.

---

## Task table

| ID  | Branch                          | Type   | Summary                                                                 | Key files |
|-----|---------------------------------|--------|-------------------------------------------------------------------------|-----------|
| 2.8 | `feat/register-kit-page`        | featwp | WordPress `/register-kit/` entrypoint and activation flow               | `docs/REGISTER_KIT_PAGE_SPEC.md`<br>`mydna-portal/wordpress/evodna-mydna-portal/includes/class-evodna-register-kit-page.php`<br>`mydna-portal/wordpress/evodna-mydna-portal/evodna-mydna-portal.php`<br>(new API handlers and infra files as needed for `v1/kit-activation-request` and `v1/kit-activation-consume`) |

---

## Task 2.8 — `/register-kit/` entrypoint and activation flow

### Title

`featwp: WordPress /register-kit/ entrypoint and activation flow (Task 2.8)`

### What

- Implement the public `/register-kit/` page in the `evodna-mydna-portal` plugin.
- Call new customer API endpoint `POST /v1/kit-activation-request` to request a
  my-dna activation link.
- Wire activation email sending using templates owned by `medlitsolutions-max`
  (see `docs/REGISTER_KIT_PAGE_SPEC.md`).
- Handle success and error states on the page, including support contact
  `info@evo-dna.com`.
- Add backend support for `POST /v1/kit-activation-consume` so my-dna can consume
  activation tokens and route users into the `KitPicker` flow.

### Why

- Align the public `/register-kit/` URL printed on boxes with the new kit
  registration architecture.
- Keep tube barcodes and lab identifiers out of WordPress while giving customers
  a simple way to activate kits.
- Centralise tube registration in `v1kitregister` inside the my-dna SPA.

### Files

- `docs/REGISTER_KIT_PAGE_SPEC.md`
- `mydna-portal/wordpress/evodna-mydna-portal/includes/class-evodna-register-kit-page.php` (NEW)
- `mydna-portal/wordpress/evodna-mydna-portal/evodna-mydna-portal.php` (bootstrap `require_once`)
- Customer API: new handlers / routes for `v1/kit-activation-request` and
  `v1/kit-activation-consume`.
- Any supporting tests / config for the above.

### How to verify

- Load `https://evo-dna.com/register-kit/` in staging:
  - Form shows kit number, email, two checkboxes, and the
    "Send me my secure activation link" button.
- Submit with both fields filled and both checkboxes ticked:
  - Page calls `POST /v1/kit-activation-request`.
  - On `200 { ok: true }` the page shows the success message and no PII is
    echoed back.
- Use a known kit/email combination in staging:
  - Receive an email with the EvoDNA activation hero (logo + runners image) and
    correct copy.
  - Clicking the link logs into my-dna (or prompts to set password) and lands
    on the kit registration screen for that order.
- Use an unknown kit/email combination:
  - See the generic success message, but no email is delivered.
- Trigger an artificial backend error:
  - Page shows "Something went wrong on our side. Please try again in a few
    minutes or contact us at `info@evo-dna.com`."

### Risk

- **Medium:**
  - Touches the public `/register-kit/` entrypoint and adds two new customer
    API routes.
  - New email flow must be tested carefully to avoid spam or token leakage.
- **Mitigations:**
  - Start behind a staging flag.
  - Use short-lived, single-use tokens and verify all error states before
    production enable.
