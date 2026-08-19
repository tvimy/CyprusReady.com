# CyprusReady — Operations Runbook

Internal notes on how the funnel is wired today and what to check when something breaks. Not linked from the site.

## The funnel, end to end

1. **Payment** — Stripe Payment Link (`buy.stripe.com/cNiaEX9KR2SV5On9eabZe00`, linked from `index.html`). Success redirect should point to `/confirmation.html`.
   - *Check:* Stripe Dashboard → Payment Links → this link → confirm the "After payment" redirect URL is set to `https://cyprusready.it.com/confirmation.html`. This is dashboard config, not in this repo — verify it hasn't drifted.
2. **Survey** — `confirmation.html` links out to a Google Form (25 questions). Response goes to a linked Google Sheet.
3. **Report generation** — a Google Apps Script (bound to the Sheet/Form) calls the OpenAI API to generate the report and emails it to the customer. Script source lives in Google Apps Script, **not in this repo**.
4. **Delivery** — email sent directly from the Apps Script (or via a connected mail service).

## Known gaps as of this runbook's writing

- **No code-level link between payment and survey access.** Anyone with the Google Form URL could fill it out without paying — the only gating is that the link isn't advertised outside `confirmation.html`, plus a `robots.txt` disallow (not real access control). Worth checking the Form's settings for a lightweight guard (e.g. requiring a per-customer code from the Stripe receipt), or accepting the current soft-gate as good enough at this scale.
- **No visible retry or alerting if the Apps Script/OpenAI call fails.** If OpenAI errors, times out, or the script throws, there is currently no automatic notification — a customer could pay, fill the survey, and never get a report with nobody noticing.
  - *Until this is hardened (tracked separately, needs the Apps Script source):* periodically check the Apps Script **Executions** log (Apps Script editor → left sidebar → Executions) for failures, and check the Google Sheet for rows with no corresponding "sent" status if one exists.
  - Customers who hit this are pointed to `cyprusready@gmail.com` via the fallback note added to `confirmation.html`.
- **No monitoring/alerting on the Stripe side** for failed or abandoned checkouts — not necessarily worth building, but worth knowing it doesn't exist.

## Recovery for a specific customer complaint

1. Search the Google Sheet for their email/response.
2. If present but no report was generated: check Apps Script Executions for an error around that timestamp; re-run manually if the script supports it, or generate the report manually and email it.
3. If not present: check Stripe for the payment, confirm they didn't drop off before reaching the Form, and send them the Form link directly.
