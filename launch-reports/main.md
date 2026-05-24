# Launch Report

## TL;DR
Launch check (Payments, Rate limits): The codebase implements IP-based rate limiting for chat generation and message entitlements. However, file upload is not rate limited, and there are several local authorization/rate-limit robustness gaps (notably around chat/message ownership checks and IP detection inputs). Payments hardening is not visible in the supplied files beyond handling an AI Gateway credit-card error message (no local payment verification/webhook logic found).

## Verdict
Launch ready: No
Security level: medium

## Detailed Report
## Scope
- Criteria covered: **Payments** and **Rate limits**.
- Repository: `ops-appbricks/chatbot` (target `main`, head `3a5418180337246bce13f896f8bb71db3faafb5d`).

## Findings summary
### Payments
- The only payment-related behavior visible is graceful handling of a specific **Vercel AI Gateway “credit card required”** error during chat streaming.
- No other payment-specific endpoints (webhooks, subscription verification, entitlement gating based on payment provider, etc.) are visible in the supplied files.

**Implication:** Payment enforcement appears to rely on the AI Gateway error path rather than local verification/hardening. This is acceptable as a start, but for “Payments” criteria we typically expect more robust local checks.

### Rate limits
- **Chat generation** is rate-limited using `checkIpRateLimit(ipAddress(request))` plus **entitlement-by-user-type** (`maxMessagesPerHour`).
- Rate limiting is only applied in the **chat POST** endpoint; other costly endpoints (notably **file upload**) do not appear to use the same limiter.
- IP rate limiting depends on `ipAddress(request)` returning a value. If this is `undefined`, the limiter is skipped.

**Implication:** For a secure launch, rate-limiting should cover more expensive surfaces (file uploads, tool flows if they hit their own endpoints, etc.) and should be robust against missing/undefined IP inputs.

## Key security gaps affecting launch readiness
1. **Missing rate limiting for `/api/files/upload`** (file upload can be abused for bandwidth/storage).
2. **Potential robustness issues in rate-limit enforcement** if IP detection fails (limiter is bypassed when IP is `undefined`).
3. **Chat message retrieval endpoint leaks “private” vs. “public” visibility semantics** (it returns different shapes for missing/“

## AI Coding Agent Notes
### Launch agent notes (what to do next)
1. Add rate limiting to **`app/(chat)/api/files/upload/route.ts`** using the existing Redis limiter (`lib/ratelimit.ts`) or equivalent.
2. Ensure IP-based limiter always gets a stable key. If `ipAddress(request)` may be undefined in your hosting setup, update chat/IP rate limiting to derive a fallback (e.g., trusted proxy headers only).
3. Re-check ownership/authorization consistency across chat/message/document endpoints. Even though some checks exist, the message GET endpoint returns non-error payload when chat isn’t found.

### File references
- Rate limiting for chat generation: `app/(chat)/api/chat/route.ts` (uses `checkIpRateLimit` + entitlement check)
- Rate limiting implementation: `lib/ratelimit.ts`
- Missing limiter in upload: `app/(chat)/api/files/upload/route.ts`

## Fixable Findings
- WARNING: File upload endpoint lacks rate limiting (abuse risk)
  - Location: app/(chat)/api/files/upload/route.ts:1-120
  - `POST /api/files/upload` validates file size/type and checks authentication, but does not apply the existing IP rate limiter (`checkIpRateLimit`) or any upload-specific limiting. Attackers can repeatedly upload files (within the 5MB cap) causing bandwidth/storage/CPU costs.
- WARNING: IP rate limiter is bypassed when IP cannot be determined
  - Location: app/(chat)/api/chat/route.ts:140-260
  - `checkIpRateLimit(ipAddress(request))` skips limiting when the IP is `undefined`. Depending on hosting/proxy configuration, `ipAddress(request)` can fail, effectively disabling IP-based rate limiting for chat generation.