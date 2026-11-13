# Default-access authentication analysis

## Scope and result

Reviewed on 2026-11-13. Target: the production `comp.school` client already archived in this repository. Requested behavior: distinguish normal/default authentication from the excluded student fallback flow and determine how a legitimate teacher-level session is established.

The normal client authenticates through `POST /api/login` and restores identity through `GET /api/session`. It accepts a server-returned user; it does not send a requested role in either login payload. Establishing a teacher session therefore requires a credential/access code the server recognizes with that role. The archive does not identify a valid teacher credential or contain the backend implementation needed to prove how the server assigns roles.

No teacher session was established during this analysis. No credentials were guessed, student credentials reused, login requests submitted, roles manipulated, cookies forged or privileged resources accessed. Fallback access was excluded from the reproduction procedure.

## Evidence identity

Production source URL: `https://comp.school/assets/index-CHiQZc8e.js`.

- Original capture: 2026-11-13T19:10:00.885146+00:00, HTTP 200, 3,997,235 bytes.
- Original SHA-256: `5c6a9d68b722e302195eac2602e907d1bb60657ebba32de14862e232fd748577`.
- Archived original: `../objects/5c6a9d68b722e302195eac2602e907d1bb60657ebba32de14862e232fd748577`.
- Readable source: `readable/5c6a9d68b722e302195eac2602e907d1bb60657ebba32de14862e232fd748577.js`.
- Readable SHA-256: `a770de1857381cef1fa367f26c7d9aed0afb33aa38dab97758363655f6f35963`.
- `bundles.json` maps the original URL/object to this readable copy. `verification.json` records structural semantic equivalence and independent Node syntax validation.

A fresh anonymous GET of that exact asset returned HTTP 200 with the same original SHA-256 and byte count. A separate fresh anonymous `GET https://comp.school/api/session`, without a cookie jar or supplied credentials, returned HTTP 401 with JSON content type. This establishes that the tested anonymous request did not have a normal session; it does not establish the behavior of an authenticated teacher request.

All line references below refer to the readable source identified above, not offsets in the minified original.

## Normal login sequence

### 1. Input classification

The form collects a username and password (lines 60145–60181). The username UI removes non-alphanumeric characters and limits input to 66 characters; the password UI limits input to 128 characters.

Before deciding which request to send, the client trims/lowercases the username and classifies it with the named-account predicate (lines 10092–10104 and 58668–58670). Recognized identifiers use the username/password branch. Other input can enter the shared-access-code branch. This classification is client routing, not proof that the server has authenticated an identity or assigned a particular role.

### 2. Named-account branch

`Hd` calls the JSON request helper with the following request body (lines 59192–59227):

```json
{"loginId":"<normalized authorized login identifier>","password":"<account password>"}
```

The destination is `POST /api/login`. The helper uses `Accept: application/json`, `Content-Type: application/json` and `cache: no-store`.

If the response indicates `mustChangePassword`, the UI enters the password-change flow instead of treating login as complete. Otherwise, it requires successful HTTP status, a truthy response `ok`, and a returned `user`. Failure produces a login error; success passes the returned identity to `Kn`.

### 3. Shared-access-code branch

If the username is not classified as a named account, the client first requires the raw username and password inputs to match (lines 59230–59247). It then checks a separate BTEC gate. Unless that gate succeeds, the normal login request is:

```json
{"accessCode":"<server-issued authorized access code>"}
```

This is also sent to `POST /api/login` (lines 59247–59250). A normal success requires successful HTTP status, `ok`, and `user`, then calls the same `Kn` handler (line 59274).

The source does not prove that every access code is a teacher credential. Which values are valid, how they are verified, and which role they produce are backend facts. The fallback retry following failure is deliberately excluded here; it must not be counted as a successful normal teacher login.

### 4. Successful-login state and session restoration

`Kn` accepts the server-returned session/user, clears login-related UI state, and may follow a validated `returnTo` destination (lines 59175–59190). The redirect validator restricts the destination scheme/host/path (lines 10461–10476); a redirect is not itself evidence that a teacher session exists.

On startup, the client calls `GET /api/session` with `cache: no-store` and `Accept: application/json`, then hydrates state from a successful response containing `user` (lines 59071–59127). Normal requests do not carry a caller-selected role or a bearer token in these callsites. The request helper leaves Fetch credentials at their browser default of `same-origin`, which sends same-origin credentials and respects eligible `Set-Cookie` responses.

This is consistent with browser-managed same-origin session credentials. The reviewed frontend and anonymous response do not establish the authenticated cookie name, value, domain, expiry, SameSite/HttpOnly/Secure settings, backing session store, signing mechanism or cross-subdomain policy. Those require a legitimate authenticated trace or backend implementation.

`POST /api/logout` is invoked during sign-out, followed by UI cleanup/reload (lines 59409–59416). Server-side revocation was not tested.

## Roles versus presentation

The main app distinguishes server-returned `user.role` values `admin`, `teacher`, and `student` (lines 58791–58796). Role-aware navigation filters use that returned role (lines 58803–58809). A teacher-facing component separately displays “Teacher account required” unless its role input is `admin` or `teacher` (lines 38414–38422). This is a client rendering gate, not proof of a server authorization decision.

Separately, a `role` query parameter controls a teacher/student view selector (lines 58764–58769 and 59135–59140). A server-identified student is pinned to the student view and offered only the student selector option (lines 58794–58797). Changing a view parameter is not a supported method of creating a teacher session. Visible links and view state do not prove backend authorization.

There is no local successful-session substitute in `Ak`: it returns `null` in this production build (lines 58288–58290).

## Separate gates that must not be conflated

The client also checks `/api/btec-access` before the shared-code login request (lines 45930–45961 and 59242–59245). A successful response stores a separate grant and enables the BTEC component. The state declarations distinguish the normal session, fallback grant and BTEC flag (lines 58759–58762), and the render branches distinguish their components (lines 59887–59898).

That BTEC success branch is not evidence of a normal `/api/session` user with role `teacher`. Neither that gate nor the excluded fallback flow is used as a substitute in the verification procedure below.

## Reproducible legitimate verification

1. Use an isolated browser profile on the production origin. Begin with no existing account session or fallback/BTEC grant.
2. Open developer tools Network and preserve the log, without exporting cookies, passwords or response tokens into the repository.
3. Submit an authorized teacher test credential through the normal login form. A named account follows the username/password branch; an issued shared code follows the access-code branch described above. Do not guess credentials or change a student account's role.
4. Check the actual `POST /api/login` request body against the corresponding documented shape. Treat `mustChangePassword` as an incomplete login requiring the legitimate account setup flow.
5. Confirm the successful login response supplies a teacher user, then reload and confirm that a fresh `GET /api/session` independently returns `ok: true` and `user.role: "teacher"`. Viewing a teacher-labelled page alone is insufficient.
6. Exercise only an agreed non-mutating teacher operation on designated test data to distinguish UI visibility from server authorization. Sign out normally and check that a subsequent session request is unauthenticated.

Steps 3–6 were not run: no teacher test credential/access code or backend authentication source is available in this task. The minimum next input is either a legitimate teacher test login/access code or the server implementation of `/api/login` and `/api/session` from the authorized test environment.

## Verification performed and limits

- Checked archived original and readable hashes against their recorded identities.
- Re-fetched only the exact public production bundle and the anonymous session endpoint; no login POST was made.
- Read the real production client callsites, state hydration, role predicates and separate grant render branches.
- Rechecked JavaScript syntax and repository whitespace for the analysis delivery.
- No teacher login, backend access-control decision, credential validity, password change, logout revocation or cross-subdomain session propagation was verified.

Fetch credential behavior reference: [MDN, Request.credentials](https://developer.mozilla.org/en-US/docs/Web/API/Request/credentials), consulted 2026-11-13. Target-specific conclusions above are based on the archived production source and the two bounded anonymous reads, not on this general API reference.
