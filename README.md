# ATO Scholarship Portal — proof of concept

**[Open the live demo](https://matasvai.github.io/ato-scholarship-demo/)** · **[Open the chair view](https://matasvai.github.io/ato-scholarship-demo/?view=chair#queue)**

The GitHub Pages version runs in your browser using a service worker and IndexedDB. Its API is explicitly simulated; records stay in that browser and are not shared with other visitors. The Node version below still provides the real local HTTP backend.

An API-driven scholarship submission and review demo for Alpha Tau Omega, Kappa Eta. The visual direction follows [ato.org](https://ato.org/): navy, gold, condensed headings, and simple rectangular controls.

**Fictional data only. This is not a live chapter system.** Microsoft / Google sign-in is simulated. Canvas returns local sample fixtures. No university account, Canvas server, Google Drive, Microsoft Graph, or Google Sheet is connected.

## Run locally

Requires **Node.js 24 or later** (uses built-in `node:sqlite`). There are no package dependencies to install.

```sh
npm start
```

Open <http://127.0.0.1:4173>. Open <http://127.0.0.1:4173/?view=chair#queue> to begin in the Scholarship Chair view.

```sh
npm test
```

To choose a different port:

```sh
ATO_DEMO_PORT=4174 npm start
```

The server binds to loopback only. The hosted GitHub Pages demo uses a browser simulation. Recipients can also clone this repository and run the real local backend.

## Try the workflow

1. Choose **Alex Brooks · Member** using **View as**.
2. Create a submission, attach fictional sample evidence, and submit it. Its status is pending and its estimated points do not count yet.
3. Choose **Scholarship chair**. Open the claim, inspect its sample evidence, then approve it or deny it with a reason.
4. Return to Alex to see the updated total and review note.
5. Choose **Import from Canvas**, load the sample assignments, select one, choose major/minor/lab, and import it for review.
6. Open **API activity** to inspect the real HTTP requests and responses. In a member view, run the access check: a request for another member's record returns 404.
7. In the chair view, export the submission register as CSV.

The demo date is fixed at **September 28, 2026** to keep the supplied Fall 2026 policy examples usable. Actual request and review timestamps use the machine's current time. Reset restores this browser session's fictional records only.

## What is real and what is simulated

| Capability | Status |
| --- | --- |
| HTTP JSON API | Working local backend |
| Durable submission records | Local SQLite, scoped to a random 24-hour demo session |
| Submission validation and point calculation | Server-side |
| Member record filtering and chair permissions | Server-side, within the selected demo persona |
| Approval / denial, history, and derived totals | Working |
| CSV export | Working |
| API activity inspector | Actual browser fetch requests and responses; no cookies logged |
| Canvas import | Working import workflow against **fictional local fixtures** |
| Microsoft / Google identity | Explanatory previews, **not connected** |
| File evidence | Fictional generated record, **no real upload support** |
| Cloud storage / Sheets / Graph | **Not connected** |

Anyone using the demo can select the chair persona. A demo session cookie is not proof of identity. **Do not enter real grades, personal information, credentials, or academic records.** The app is not ready to protect real student data.

## API layout

All paths are same-origin. Start a demo session before accessing other endpoints. Mutating requests require `Content-Type: application/json` and `X-ATO-Demo: 1`. Browser requests include `X-ATO-Expected-User` to catch another tab changing the selected persona. That header is a stale-tab guard, not authentication.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| POST | `/api/demo/session` | Start a session or select `alex`, `jordan`, or `chair` |
| GET | `/api/me` | Current demo persona |
| GET | `/api/rules` | Activity rules and checkpoints |
| GET | `/api/points` | Current member's derived totals |
| GET | `/api/submissions` | Member's own records, or chair's queue |
| POST | `/api/submissions` | Validate and create a pending claim |
| GET | `/api/submissions/:id` | Authorized detail lookup |
| GET | `/api/submissions/:id/evidence` | Authorized sample evidence lookup |
| POST | `/api/submissions/:id/review` | Chair decision; duplicate reviews return 409 |
| GET | `/api/members` | Chair-only progress totals |
| GET | `/api/integrations/canvas/assignments` | Member's fictional Canvas-shaped assignments |
| POST | `/api/integrations/canvas/import` | Atomically import selected sample assignments as pending claims |
| GET | `/api/export` | Chair-only CSV register |
| POST | `/api/demo/reset` | Restore this session's sample data |

A sample claim body:

```json
{
  "activity": "major",
  "title": "Calculus II · Quiz 4",
  "course": "MTH 2002",
  "date": "2026-09-28",
  "grade": 95,
  "evidence": "sample",
  "confirm": true,
  "note": "Fictional example"
}
```

A review body:

```json
{
  "decision": "approved",
  "points": 5,
  "note": "Sample grade verified."
}
```

## Storage and authorization boundaries

- Data is saved in `data/demo.sqlite`, which is ignored by Git.
- A random HttpOnly, SameSite cookie identifies an isolated demo dataset. Sessions expire after 24 hours; expired sessions are cleaned up when a new session starts.
- Member lookup endpoints filter by the server session's persona. Foreign records return 404; chair-only actions return 403 for members.
- Only approved claims contribute to point totals. Totals are computed from records, never maintained in a second editable sheet.
- Reviews transition a pending record once. Denied claims retain their history; a corrected submission may be sent separately.
- Canvas course and assignment IDs prevent repeated imports. Denied sample imports can be corrected and resubmitted; their original history stays in the register.
- State read, validation, and persistence occur synchronously after request parsing in a single Node process. This demo is not designed for multi-process writes or production scaling.
- Session cookies lack `Secure` only because this demonstration runs on local HTTP. Production needs HTTPS, secure cookies, real identity validation, role administration, rate limits, retention rules, and operational monitoring.

## Connecting university accounts later

**Microsoft:** use Microsoft Entra ID / OpenID Connect, normally a tenant-specific authorization code flow with PKCE. Validate the token with a supported library, its issuer/audience/expiry, and the approved university tenant. Match the verified identity to a chapter roster; store provider-scoped stable identifiers (such as tenant ID plus object ID), not an email typed into a form. University tenant access alone is not chapter membership. The institution may need to approve the app registration or consent.

**Google:** verify the signed ID token server-side, including issuer, audience, expiry, and the university hosted-domain claim. Use Google's stable `sub` identifier. A domain-matching email string by itself is insufficient. Chapter membership and the chair role still require explicit server-side authorization.

Neither provider's sign-in automatically grants access to Canvas, Drive, Sheets, or Microsoft Graph. Those are separate scoped integrations. Never assign the chair role from a user-supplied field or replace backend authorization with hidden interface controls.

Official references:

- [Microsoft OpenID Connect](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols-oidc)
- [Restrict an Entra application to selected users](https://learn.microsoft.com/en-us/entra/identity-platform/howto-restrict-your-app-to-a-set-of-users)
- [Google server-side ID-token verification](https://developers.google.com/identity/gsi/web/guides/verify-google-id-token)

## Connecting real Canvas later

A multi-user Canvas integration requires an institution-issued OAuth developer key. Do not ask chapter members to paste personal Canvas access tokens. Keep the OAuth client secret, access tokens, and refresh tokens on the backend; validate OAuth state, use narrow read-only scopes, and import only the consenting member's data.

Useful endpoints:

- `GET /api/v1/courses?enrollment_type=student`
- `GET /api/v1/courses/:course_id/assignments?include[]=submission`
- `GET /api/v1/courses/:course_id/assignment_groups` when investigating course weighting

Follow Canvas's `Link` header pagination. Distinguish missing grades from a numeric zero. For point-graded assignments with positive `points_possible`, a displayed percentage can be calculated as `100 * submission.score / points_possible`. Do not infer percentages from letter grades or treat student what-if scores as official marks. Respect grade visibility and excused work.

A group's course weight is **not** the individual assignment's course weight. Drop rules, grading periods, exclusions, and extra credit further complicate this. Preserve member classification and chair review rather than auto-awarding points. Import does not submit work to Canvas or change any Canvas grades.

Official references:

- [Canvas OAuth requirements](https://developerdocs.instructure.com/services/canvas/oauth2/file.oauth)
- [OAuth endpoints and scopes](https://developerdocs.instructure.com/services/canvas/oauth2/file.oauth_endpoints)
- [Assignments](https://developerdocs.instructure.com/services/canvas/resources/assignments)
- [Submissions](https://developerdocs.instructure.com/services/canvas/resources/submissions)
- [Pagination](https://developerdocs.instructure.com/services/canvas/basics/file.pagination)

## Policy assumptions requiring chapter review

Rules were adapted from the user-supplied *ATO Scholarship Plan 2026*, updated August 23, 2026. The original document is intentionally not included in this public repository.

- The plan conflicts on Tier 3 / Tier 4 GPA boundaries. This demo uses preassigned tiers.
- Fractional points are prohibited but multiplier rounding is unspecified. The chair must enter a whole-number award and explain deviations from the estimate.
- Weeks are Monday–Sunday in this demo. Pending study claims reserve capacity; denied claims release it. The shared study-hour cap covers explicit study categories, not professor office hours or tutoring.
- A grade-posting date starts the sample Canvas claim window. The policy does not settle which assignment date controls this window.
- Exact 5% course weight is not assigned automatically to major or minor.
- Checkpoint targets are interpreted as cumulative and are not multiplied by credit load.
- The final submission closing date remains unresolved.
- Required study-night attendance is separate from points and is not tracked here. No sanctions are automatically imposed.
- The current plan accepts Google Forms or physical evidence. A live portal should be approved as a submission channel before replacing that process.
- Academic evidence is limited to the member and Scholarship Chair in this design. Other chapter officers do not receive evidence access by default.

## Project structure

```text
dist/                   Browser interface (plain HTML, CSS, JavaScript)
server/index.mjs        HTTP routes, local sessions, SQLite storage
server/domain.mjs       Point rules, validation, fictional members and submissions
server/canvas-sample.mjs Fictional Canvas-shaped records
tests/api.test.mjs       Isolated HTTP integration tests
data/                   Generated local database; never committed
```

The interface also registers a read-only WebMCP tool when supported by the browser. It reads the current demo persona's records through the same API.

This is an unofficial chapter workflow prototype, not an official Alpha Tau Omega, Florida Tech, Canvas, Google, or Microsoft product.
