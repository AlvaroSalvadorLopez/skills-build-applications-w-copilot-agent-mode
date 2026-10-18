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

Record your plan here before editing code.

- Presentation tier decision: Keep each capability in its existing React component and use the shared API client for relative `/api` requests. Every data view must expose accessible loading, empty, and error states.
- Logic tier decision: Keep route handlers small; validate request data and references at the API boundary, and put reusable business rules in the service layer. Return deliberate 4xx responses for invalid input or unknown records instead of leaking database cast errors.
- Data tier decision: Make Mongoose schemas enforce required fields, valid references, and sensible numeric constraints. Keep seed data representative and idempotent. Authentication is out of scope.
- Activity tracking approach: Validate that `userId` and `teamId` are well-formed IDs for existing records before creating an activity. Store the referenced user and team, submitted type and minutes, computed points, and a server-generated occurrence time.
- Team management approach: Require a non-empty name; preserve the submitted name and mascot and the member references on create and list.
- Leaderboard approach: Sum activity points per team, sort totals from highest to lowest, and assign 1-based ranks. Use team name as a stable tie-breaker.
- Workout suggestions approach: Find the user by email and use their fitness goal and recent activity to order relevant workouts. Include a short reason for the top suggestion and return a 4xx response when no user matches.

Implement bottom-up: data models, API behavior, then presentation. Within the API, finish activity and team writes before the leaderboard and personalized workout queries that depend on persisted data.

## Test plan

Run the focused checks in dependency order. A failing data check points to a schema or seed contract; a failing API check points to request validation or business behavior; a failing frontend check points to rendering, API integration, or accessible request states. Build failures indicate TypeScript or bundling issues rather than a contract assertion.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` - checks required model fields and seed behavior. Current baseline: the seed test passes, while the User, Team, Activity, and Workout schema contract tests fail.
- API command: `npm run test:api --prefix octofit-tracker/backend` - checks activity and team validation/writes, leaderboard totals and ordering, and user-specific workout suggestions. Current baseline: all four API contract tests fail; invalid activity/team requests return `501`, the leaderboard is empty, and Mona's first workout suggestion does not match her endurance activity.
- Frontend command: `npm test --prefix octofit-tracker/frontend` - checks data rendering, the user-specific workout request, and loading, empty, and accessible error states for all four views. Current baseline: all 15 presentation contract tests fail.
- Backend build command: `npm run build --prefix octofit-tracker/backend` - checks backend TypeScript types; not yet run for the current baseline.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` - checks frontend TypeScript and Vite production bundling; not yet run for the current baseline.
