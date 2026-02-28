# Assumption Verification — Example Output

> This is an example of what Step 2 (verify) produces. Your actual output will reflect your codebase.

> **Verified:** Feb 28, 2026
> **Input:** `docs/assumptions.md`
> **Repo health:** Well-maintained. Clean commit history, consistent naming, partial test coverage. Trust level: moderate-high.

## Verdict Summary

| Flow | Assumption | Status | Action Needed |
|------|-----------|--------|---------------|
| 1 | User registration validates email format | **Confirmed** | None |
| 2 | Password reset tokens expire after 1 hour | **Incorrect (code bug)** | Tokens never expire — add TTL check |
| 3 | Admin delete removes user record | **Incorrect (assumption wrong)** | Soft-delete only — sets `deletedAt` |
| 4 | Rate limiting on login endpoint | **Confirmed** | None |
| 5 | Session invalidation on password change | **Incorrect (code bug)** | Old sessions remain valid |
| 6 | [UNCERTAIN] OAuth linking with existing account | **Partially correct** | Links by email but case-sensitive |
| 7 | Order total calculated server-side | **Partially correct** | Subtotal yes, discount trusts client |

**Bugs found: 2** (Flow 2 token expiry, Flow 5 session invalidation)
**Assumptions corrected: 2** (Flow 3 soft-delete, Flow 7 pricing)
**High-risk items investigated: 3** (Flows 5, 6, 7)

---

## Flow-by-Flow Verification

### Flow 1: User Registration — CONFIRMED

**Assumption:** Registration validates email format before creating the account.

**Verification:** Confirmed. `registerUser()` in `app/actions/auth.ts:34` uses a Zod schema with `.email()` validation. Invalid emails return a 400 before any database write.

**Action:** None.

---

### Flow 2: Password Reset — INCORRECT (CODE BUG)

**Assumption:** [ASSUMPTION] "Password reset tokens expire after 1 hour based on the `TOKEN_TTL` constant I found."

**Verification:** `TOKEN_TTL = 3600` is defined in `lib/constants.ts:14` but never referenced in the reset flow. `verifyResetToken()` in `lib/auth/reset.ts:47` checks token existence and validity but does NOT check `createdAt` against any TTL.

**Impact:** Password reset tokens are valid forever. A leaked or intercepted token from months ago still works. Security vulnerability.

**Fix options:**
1. Add `createdAt` check in `verifyResetToken()`: `if (Date.now() - token.createdAt > TOKEN_TTL * 1000) return null`
2. Add database-level TTL index on the tokens collection for automatic cleanup

**Priority:** P1 — security issue, simple fix.

---

### Flow 3: User Deletion — ASSUMPTION WRONG

**Assumption:** "Admin delete removes the user record from the database."

**Actual behavior:** `deleteUser()` in `app/actions/admin.ts:89` performs a soft-delete:
- Sets `deletedAt: new Date()`
- Anonymizes email to `deleted-{id}@removed.local`
- Nullifies personal fields (phone, avatar)
- Preserves the document for referential integrity

`getUserById()` filters soft-deleted users via `&& !defined(deletedAt)` in GROQ.

**Action:** Update assumptions doc — deletion is soft-delete with PII anonymization, not hard delete. This is intentional, not a bug.

---

### Flow 5: Session Invalidation — INCORRECT (CODE BUG)

**Assumption:** [UNCERTAIN] "I think changing your password invalidates existing sessions, but I'm not sure how."

**Verification:** It does NOT. `changePassword()` in `app/actions/account.ts:112` updates the password hash but does not call `invalidateSessions()` or modify the JWT signing key. Existing JWTs remain valid until their natural expiry (12 days).

**Impact:** If a user changes their password because they suspect compromise, the attacker's existing session continues working for up to 12 days.

**Fix options:**
1. Add a `passwordChangedAt` field to the user document. Check it in the JWT verification callback — reject tokens issued before the password change.
2. Increment a `securityVersion` counter on password change and include it in the JWT. Reject mismatches.

**Priority:** P1 — security issue, requires auth infrastructure change.

---

### Flow 6: OAuth Account Linking — PARTIALLY CORRECT

**Assumption:** [UNCERTAIN] "OAuth sign-in links to existing accounts by email, but I'm not sure about the matching logic."

**Verification:** Partially correct. `linkAccount()` in `lib/auth/adapter.ts:67` queries `*[_type == "user" && email == $email]`. This is case-sensitive GROQ equality.

- Google OAuth provides lowercase email (`john@gmail.com`)
- If the user registered with `John@gmail.com`, the link fails silently
- User ends up with two accounts

**Impact:** User can't access their existing data after OAuth sign-in. Contacts support.

**Fix:** Use `lower(email) == lower($email)` in the GROQ query. Also normalize email to lowercase at registration.

**Priority:** P2 — data integrity issue, simple GROQ fix.

---

## New Bugs Found

### Bug A: Password Reset Tokens Never Expire (Flow 2)
**Severity:** High
**Impact:** Leaked reset tokens work forever. Security vulnerability.
**Fix:** Check `createdAt` against `TOKEN_TTL` in `verifyResetToken()`
**Priority:** P1

### Bug B: Sessions Not Invalidated on Password Change (Flow 5)
**Severity:** High
**Impact:** Compromised sessions survive password changes for up to 12 days.
**Fix:** Add `passwordChangedAt` or `securityVersion` field, check in JWT callback
**Priority:** P1

## Corrections to Assumptions Doc

1. **Flow 3:** Replace "Admin delete removes user record" with: Soft-delete with PII anonymization. Document persists with `deletedAt` timestamp. Filtered from queries.

2. **Flow 7:** Replace "Order total calculated server-side" with: Subtotal IS recalculated server-side from package prices. The gap is the promo discount — it trusts the client-provided promo configuration.

## Prerequisites Discovered

| Prerequisite | Status | Needed Before |
|-------------|--------|---------------|
| Token TTL enforcement | Not done | Any security audit |
| Session invalidation on password change | Not done | Production launch |
| Case-insensitive email matching | Not done | OAuth integration |
