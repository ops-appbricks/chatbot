# Launch Report

## TL;DR
Authorization is generally enforced for chat/documents/messages/votes/history. However, the file upload endpoint allows writing blobs with public access using only session truthiness (and appears to ignore the provided blob token/constraints). The chat streaming endpoint permits access to any chat id without a visibility check when loading messages, and some endpoints return inconsistent/incorrect status codes for auth failures. There is also a potential auth-token redirect bypass surface via un

## Verdict
Launch ready: No
Security level: high

## Detailed Report
## Authentication / Authorization Launch Security Check (ops-appbricks/chatbot @ main)

### What looks good
- **Chat ownership checks** exist in the main chat endpoint: `app/(chat)/api/chat/route.ts` verifies `chat.userId !== session.user.id` and returns `forbidden:chat`.
- **Document ownership checks** exist in `app/(chat)/api/document/route.ts` for GET/POST/DELETE.
- **History and voting** endpoints check session user identity and chat ownership before returning/mutating data.
- **Messages in private chats** are protected in `app/(chat)/api/messages/route.ts` (private visibility + session mismatch => 403).

### Critical authorization-related issues
1) **Chat streaming messages loading lacks a private-visibility guard** in `app/(chat)/api/chat/route.ts`.
   - The endpoint uses `getChatById({id})` and enforces ownership only when the chat already exists.
   - But it then uses `getMessagesByChatId({id})` without checking `chat.visibility` (and without an allowlist for public chats). This may allow reading message histories via the chat generation endpoint even when a chat is public, or allow bypass patterns if visibility semantics differ across the app.

2) **File upload endpoint potentially creates public blobs without tight authorization controls**.
   - `app/(chat)/api/files/upload/route.ts` writes to Vercel Blob with `access: "public"` and only checks `if (!session)`.
   - There’s no authorization binding to a specific chat/document/artifact, and no visibility-based constraints.
   - If a guest user is used to generate or view chats, these public URLs could become widely accessible.

### Authentication / status code inconsistencies
- `app/(chat)/api/document/route.ts` returns `not_found:document` on missing auth in POST (should be unauthorized/forbidden). This can be

## AI Coding Agent Notes
## Agent action plan (concrete)
1) **Fix chat visibility authorization**: Update `app/(chat)/api/chat/route.ts` so that when loading `messagesFromDb` it enforces `chat.visibility` rules consistent with `app/(chat)/api/messages/route.ts`.
2) **Harden file upload authorization**: Update `app/(chat)/api/files/upload/route.ts` to avoid `access: "public"` (or enforce tighter auth + scoping). Also ensure the blob keying includes a namespace and cannot collide across users.
3) **Correct endpoint auth failure semantics**: Update `app/(chat)/api/document/route.ts` POST to return `unauthorized:document` (or `forbidden`) rather than `not_found:document`.

## References (where to patch)
- Chat generation/authorization: `app/(chat)/api/chat/route.ts`
- Document API auth handling: `app/(chat)/api/document/route.ts`
- Upload policy: `app/(chat)/api/files/upload/route.ts`

## Suggested verification after patches
- Attempt to call chat streaming generation with a **non-owned private chat id**: should always fail.
- Attempt to call messages GET for private chats already handled: should remain 403.
- Attempt to upload a file as guest/regular and verify resulting blob access behavior (public vs private) aligns with threat model.
- Confirm status codes for unauthenticated document POST are consistent and don’t leak resource existence.

## Fixable Findings
- ERROR: Chat streaming endpoint should enforce chat visibility/ownership consistently when loading messages
  - Location: app/(chat)/api/chat/route.ts:1-120
  - In `app/(chat)/api/chat/route.ts`, when a chat exists the endpoint checks ownership (`chat.userId !== session.user.id`) before loading messages via `getMessagesByChatId`. But the endpoint’s behavior is still inconsistent with `app/(chat)/api/messages/route.ts`, which explicitly enforces `chat.visibility === "private"` for non-owners.

As written, the chat generation endpoint only allows owners for existing chats, but it does not consult visibility at all (and it loads messages for any chat it considers accessible after the ownership check). To prevent authorization mismatches and future regressions (e.g., if ownership checks are altered or “public” chats are expected to be accessible), add a visibility-based guard aligned with the messages endpoint and the app’s share semantics.

Fix: fetch chat first, then enforce:
- if `chat.visibility === "private"` require `chat.userId === session.user.id`
- if `chat.visibility === "public"` allow based on the app’s share rules (or always require ownership if that is the intended model).

Exact location: `app/(chat)/api/chat/route.ts` around the `if (chat) { ... messagesFromDb = await getMessagesByChatId({ id }); }` block.
- ERROR: File upload stores files as publicly accessible blobs without authorization scoping
  - Location: app/(chat)/api/files/upload/route.ts:1-120
  - `app/(chat)/api/files/upload/route.ts` uploads user-provided files to Vercel Blob with `access: "public"`. The endpoint only checks `if (!session)` and does not tie the upload to a chat/document/artifact or restrict access to authorized viewers.

This can expose uploaded content broadly (including guest uploads) via the returned public URL.

Fix options (local code change):
- Change blob `access` from `"public"` to a non-public mode (if supported in this deployment) and/or
- Prefix blob keys with user id and add authorization checks wherever those URLs are consumed.

Exact location: `app/(chat)/api/files/upload/route.ts` at the `put(..., { access: "public" })` call.
- WARNING: Document POST returns not_found when unauthenticated
  - Location: app/(chat)/api/document/route.ts:1-110
  - In `app/(chat)/api/document/route.ts`, `POST` returns `new ChatbotError("not_found:document")` when `!session?.user`. This is inconsistent with other endpoints (and with `GET`/`DELETE` which return `unauthorized`). It can also leak different behavior patterns to an attacker.

Fix: return `unauthorized:document` (or `forbidden:document`) for missing auth.

Exact location: `app/(chat)/api/document/route.ts` in the `POST` handler immediately after `const session = await auth();`.