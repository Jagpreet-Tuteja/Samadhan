# Authentication

Status: design stage.

## Model

- JWT access tokens (short-lived) + rotated refresh tokens.
- Login via phone number + OTP as the primary method (matches how target
  business owners already verify identity — WhatsApp/SMS OTP), with
  email + password as a secondary option.
- Google OAuth as an additional login option for staff/admin users.
- A single user can belong to multiple organizations (e.g. an accountant
  managing several shops); the active organization is selected per
  session and every API call is scoped to it.

## Session handling

- Refresh tokens are rotated on use and revocable per-device, so a lost
  phone can be logged out remotely from account settings.
- Sensitive actions (deleting a customer, changing GST number, exporting
  all data) require re-authentication even within an active session.

## Why phone+OTP first

Target users are shop owners who already trust phone-based OTP flows from
banking and UPI apps; password fatigue and forgotten passwords are a
larger real-world support burden than OTP delivery reliability for this
audience.

See [adr/0003-authentication.md](adr/0003-authentication.md).
