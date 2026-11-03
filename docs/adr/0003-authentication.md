# 0003 — Authentication

Status: Accepted
Date: 2026-07-19

## Context

Target users (shop owners) are already comfortable with phone + OTP flows
from banking/UPI apps, and password-based login has a higher real-world
support burden (forgotten passwords) for this audience than OTP delivery
issues.

## Decision

Phone number + OTP as the primary login method, JWT access tokens with
rotated refresh tokens for session management, email/password and Google
OAuth as secondary options for staff/admin users. See
[docs/authentication.md](../authentication.md).

## Consequences

- Requires reliable SMS/WhatsApp OTP delivery infrastructure from day one.
- Refresh token rotation adds implementation complexity but allows
  per-device revocation, important for shared-device retail environments.
- A user can belong to multiple organizations; session must carry an
  active-organization selection.
