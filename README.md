# Campus Complaint Management System

This repository contains the current `develop` state of the Campus Complaint Management System (CCMS), a React, Node.js, Express, and MongoDB application for submitting, tracking, and administering campus complaints.

## Fork Context

This repository is a fork of the original CCMS team project:

- Original repository: [HarshitPandey-2021/Campus-Complaint-Management-System](https://github.com/HarshitPandey-2021/Campus-Complaint-Management-System)
- Fork: [Shiva-Singh09/Campus-Complaint-Management-System](https://github.com/Shiva-Singh09/Campus-Complaint-Management-System)
- Verifiable base state: the fork's `develop` history currently ends at merge commit `f94fbf5`, which merged the `shakti-student-ui` work after the Shiva backend merge.

The original repository URL is supported by the merge history and the previous project documentation. The exact fork creation event is not separately recorded in the available Git metadata, so the base state is described only to the extent that it can be verified.

This fork is intended for an independent technical review of the original project, identification of engineering gaps, bug and architecture analysis, documentation of improvement opportunities, and subsequent independent improvements.

## Original Team

The following descriptions combine repository history, merge metadata, existing project attribution, and the code present at the released `develop` tip. They are neutral scope descriptions, not rankings.

### Harshit Pandey

- Major frontend and product implementation
- Admin dashboard UI and integration work
- Student portal scaffolding and project structure
- Documentation and integration/merge coordination
- Admin activity-log, complaints-table, and responsive UI work visible in history

### Shakti / Sishan24

- Frontend development
- Landing page work
- Login and signup UI
- Student-facing frontend changes

The Git author identity used for this work is `Sishan24`; the relationship to the name Shakti is based on the project's existing team attribution.

### Shiva Singh / Shiva-Singh09

- Core Node.js and Express backend
- REST API implementation
- MongoDB integration using the official MongoDB driver
- Database connection, indexes, and ObjectId handling
- JWT authentication, bcrypt password hashing, and role authorization
- Complaint-management business logic and complaint APIs
- Analytics APIs and admin-backend integration
- Admin complaint-management components and live frontend API communication
- Integration and merge contribution where recorded in Git history

### Somesh Pandey

The existing project attribution identifies Somesh with backend-related development, including password and forgot-password work and backend consolidation. The available history in this fork does not contain a matching `Somesh` author identity or a separately preserved branch, so those specific contributions cannot be independently verified from Git metadata in this copy.

## My Contribution to the Original Project

I was also a member of the original CCMS team. I owned the core backend, data and security layer, complaint-management APIs, and frontend-backend integration of the original project. The verified scope includes:

- Core Node.js/Express backend and REST APIs
- MongoDB persistence through the official `mongodb` driver, including connection setup and indexes
- ObjectId validation and handling
- JWT authentication, bcrypt password hashing, and role-based authorization
- Complaint-management business logic and analytics endpoints
- Admin API integration
- Replacement of admin mock-data paths with Axios-based live API calls
- The current admin complaint-table component system and small admin defect fixes
- Integration and merge work where recorded in the project history

This does not mean that I built the entire CCMS or the entire admin dashboard. The released `develop` state contains work from all contributors described above.

### Released Versus Branch-Only Work

The contribution statements above refer to work present in the current `develop` tree or supported by its merge history. No separate later modular-backend restructuring or backend README is represented as part of this released `develop` tip. Any such later work outside this tip is branch-only and is not treated as part of the released project in this audit.

## Current Architecture

The repository currently contains three frontend applications and a backend:

- `frontend/student-ui`: landing and entry experience
- `user-portal`: student complaint experience
- `admin`: administration dashboard, complaint management, activity UI, and analytics views
- `backend`: Express API, JWT middleware, controllers, routes, and MongoDB access

The intended request flow is:

```text
Client -> frontend application -> Express API -> authentication middleware
       -> controller/business logic -> MongoDB -> HTTP response
```

The backend uses the official MongoDB driver for persistence. The current tree also contains more than one backend entry path (`app.js` and `index.js`), which should be consolidated before production operation. Controllers currently combine request validation, business logic, database access, and response formatting.

## Independent Technical Review

Review performed on **2026-09-27** after forking the project. This section records the current implementation as observed in the repository; no application source code was changed for this review.

### Confirmed Issues

- Admin complaint routes do not enforce authentication or role authorization. The relevant middleware calls are commented out for listing complaints, reading a complaint, updating status, marking a complaint read, and analytics.
- User registration accepts the `role` supplied by the request body. Without a server-side restriction to permitted registration roles, a caller can request an administrative role.
- The admin service sends `page` and `limit`, but the backend complaint-list controller ignores them and returns the complete collection. Client-side filtering and sorting therefore operate on an unbounded response.
- The analytics endpoint performs several independent `countDocuments` queries and loads every resolved complaint into memory to calculate average resolution time. The admin analytics page also triggers several full complaint-list requests in parallel.
- The admin frontend hard-codes `http://localhost:4000/api/complaints`, while the project documentation describes a deployed backend URL. This makes the admin integration environment-specific unless the source is changed or proxied.
- The repository contains two different backend startup implementations with different route shapes and ports. The backend package has no development or start script, increasing the risk of running an unintended API surface.
- Complaint status updates accept arbitrary status strings and do not clear `resolvedAt` when a resolved complaint is moved back to another status. The update checks also use truthiness, so empty remarks or assignments cannot be intentionally cleared.
- The analytics error response includes the raw caught error object, which can expose implementation details to clients.
- The admin logs route and controller are present but contain no registered routes or exported behavior, so that backend capability is incomplete in the current tree.

### Improvement Opportunities

- Introduce a service layer so controllers focus on transport concerns and database operations are reusable and testable.
- Define consistent request schemas, permitted enum values, response shapes, and error codes across all endpoints.
- Make pagination, filtering, sorting, and projections explicit API contracts rather than client-side assumptions.
- Consolidate the backend entry point and provide documented development, test, and production commands.
- Add integration tests for authorization, ownership checks, complaint transitions, analytics, and error responses.
- Add structured logging, request correlation, monitoring, and a controlled environment configuration strategy.
- Restrict CORS to known application origins and add security headers in production.

### Future Improvements

- Add centralized rate limiting for authentication and abuse-prone endpoints.
- Add a deliberate notification delivery model if real-time updates are required.
- Add background processing for expensive reporting or notification work when usage justifies it.
- Add CI checks for linting, tests, API contracts, and deployment readiness.

## Bugs / Defects Identified

### Admin mutation and analytics endpoints are publicly reachable

**Observed behavior:** The admin complaint routes are registered without `verifyToken` and `authorizeRoles("admin")`.

**Root cause:** The protected route definitions are commented out in `backend/src/routes/complaints.js`.

**Impact:** Requests can reach administrative reads and mutations without the intended authentication and authorization boundary.

**Current status:** Identified and documented; source-code fix planned for a subsequent branch.

### Admin pagination contract is not implemented by the backend

**Observed behavior:** The admin client sends `page` and `limit`, but the controller returns all complaints and the client treats the response as an array.

**Root cause:** The backend controller has no pagination, skip/limit, or pagination metadata.

**Impact:** Response size and client memory/processing grow with the complaint collection.

**Current status:** Identified and documented; source-code fix planned for a subsequent branch.

### Registration permits a caller-selected role

**Observed behavior:** Registration copies `role` from the request body into the user document and JWT.

**Root cause:** There is no server-side policy restricting public registration to the student role.

**Impact:** A caller may obtain a role with administrative privileges if the route is reachable and the role is accepted by the rest of the system.

**Current status:** Identified and documented; source-code fix planned for a subsequent branch.

## Security Review

### Confirmed security findings

- JWT verification and role middleware exist and student complaint routes use them, but the current admin complaint routes bypass both checks.
- Complaint detail ownership is checked for the student path in the controller, but the admin path is currently unauthenticated at the route boundary.
- Passwords are hashed with bcrypt before persistence and are omitted from the login response.
- Public registration accepts a caller-provided role and should be restricted server-side.
- CORS is enabled for all origins with `cors()` and no visible allowlist.

### Security improvement opportunities

- Add rate limiting and account-abuse controls for login and registration.
- Validate all request bodies with explicit schemas and permitted values.
- Use security headers, restricted CORS, secret rotation, and production token storage/refresh policies.
- Avoid returning raw errors and ensure uploaded content is validated and stored safely if file upload is enabled.

## Performance & Scalability Review

### Confirmed performance problems

- Complaint listing is unpaginated despite pagination parameters in the client.
- Analytics uses multiple count queries plus an unbounded read of resolved complaints.
- The analytics page repeats full complaint-list requests for category, status, trend, average time, and fallback statistics.
- Client-side search, filtering, sorting, and aggregation require the full complaint dataset.

### Potential scalability improvements

- Preserve and extend the existing complaint indexes for the actual filter and sort patterns, then verify them with query plans.
- Replace repeated analytics reads with targeted aggregation pipelines or a materialized statistics strategy.
- Return only fields needed by each view and add server-side filtering and sorting.
- Cache short-lived dashboard and analytics results after correctness and invalidation rules are defined.

## Redis & Caching Review

Redis is identified as a future scalability and caching improvement and is **not implemented in this review branch**.

Potential uses include:

- Dashboard statistics with a short TTL
- Expensive analytics aggregation results with a short TTL
- Short-lived notification state or pub/sub if real-time notifications are introduced
- A centralized rate-limit store when the API runs on multiple instances

Redis is not required for the current JWT model. JWT validation is stateless; Redis would become useful only if token revocation, session coordination, distributed rate limiting, or shared short-lived state is required.

Cache invalidation should follow complaint events. A complaint created should invalidate dashboard and relevant analytics keys. A status update should invalidate status/count and analytics keys. TTLs and explicit invalidation should be chosen together so stale statistics are bounded and predictable.

## Improvement Roadmap

### Phase 1 - Correctness

- Restore admin authentication and authorization
- Restrict registration roles
- Add request validation and consistent status/error responses
- Implement the pagination contract
- Correct complaint transition and analytics edge cases

### Phase 2 - Architecture

- Consolidate backend entry points
- Separate routes, controllers, services, and data access
- Create reusable authentication and validation middleware
- Complete or remove unfinished admin-log integration

### Phase 3 - Performance

- Add server-side pagination, filtering, sorting, and projections
- Optimize analytics with aggregation and measured indexes
- Cache dashboard and analytics responses where justified
- Introduce Redis only for a demonstrated distributed-state or caching need

### Phase 4 - Security

- Add rate limiting and abuse controls
- Strengthen authorization coverage and ownership tests
- Add security headers, CORS policy, and input sanitization
- Define a production token and secret-management strategy

### Phase 5 - Production Readiness

- Add automated unit and integration tests
- Add structured logs, monitoring, and alerting
- Add CI/CD checks and deployment configuration
- Document environment management without exposing secrets

## Local Structure

```text
admin/                 React admin dashboard
backend/               Express and MongoDB API
frontend/student-ui/   Landing and entry frontend
user-portal/           Student portal
docs/                  Project documentation and screenshots
```

## Environment Variables

The backend expects environment variables such as `MONGODB_URI`, `DB_NAME`, `JWT_SECRET`, and optionally `JWT_EXPIRES_IN` and `SALT_ROUNDS`. Keep secrets in local environment configuration and never commit their values.

## Review Scope

This commit documents the initial independent audit only. It intentionally does not modify application source code, `package.json`, database code, configuration or environment files, other documentation, or Redis. Source-code fixes will be handled in subsequent branches after this review.
