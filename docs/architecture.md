# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

- Presentation tier: Keep the four capability components responsible for their own loading, success, empty, and error states; use the shared API client for relative `/api` requests and accessible status and alert text.
- Logic tier: Keep Express routes as thin request/response adapters and implement validation and business behavior in explicit service functions. Return deliberate 4xx responses for invalid input, malformed or missing references, and unknown users.
- Data tier: Express persistence through Mongoose schemas with required fields, references, and sensible defaults. Keep seed data idempotent and preserve unrelated records while reconciling seeded users' team memberships.
- Activity tracking: Validate required fields and well-formed existing user and team IDs before writing; compute points and the occurrence date on the server, then return the persisted activity from create and list operations.
- Team management: Require a name, retain the submitted name and mascot, and support a default empty members array and submitted member references.
- Leaderboard: Aggregate activity points by team, order totals descending, and assign one-based ranks in the service response.
- Workout suggestions: Find the requested user by email, use their fitness goal and recent activity to select and explain relevant workout suggestions, and return a 4xx for an unknown email.

## Test plan

- Dependency setup: `npm ci --prefix octofit-tracker/backend` and `npm ci --prefix octofit-tracker/frontend` (both completed before the baseline run).
- Baseline: `npm test --prefix octofit-tracker/backend` has 8 failing and 1 passing test. The failures identify missing required Mongoose validation for users, teams, activities, and workouts; unimplemented activity/team service behavior and validation responses; an empty leaderboard; and suggestions that do not match the user. The seed idempotency/preservation test passes.
- Baseline: `npm test --prefix octofit-tracker/frontend` has 15 failing tests. Three data/fetch rendering contracts and all loading, empty, and API-error state contracts fail because the capability screens remain placeholders.
- Data contract: `npm run test:data --prefix octofit-tracker/backend` checks model validation and idempotent seed behavior without deleting unrelated data; failures indicate a schema or seed regression.
- API contract: `npm run test:api --prefix octofit-tracker/backend` checks activity/team creation and listing, deliberate 4xx validation, leaderboard ordering, and user-specific workout suggestions; failures indicate route/service contract or business-logic defects.
- Frontend contract: `npm test --prefix octofit-tracker/frontend` checks API-backed rendering, the requested workout user, and accessible loading, empty, and error states; failures indicate presentation or API-client integration defects.
- Backend build: `npm run build --prefix octofit-tracker/backend` checks TypeScript types and backend compilation.
- Frontend build: `npm run build --prefix octofit-tracker/frontend` checks frontend TypeScript and production bundling.
