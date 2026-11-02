# Task Manager — Architecture & Decision Log

A mini-Trello / Notion-style task management app built with the MEAN stack.
This document records every architectural decision made during planning (Phases 1–6), so the implementation stays consistent and each choice can be explained.

---

## 1. Product Overview

**Model:** Team task manager with role-based access control (RBAC).

- **Admin** creates boards, columns, tags and tasks, and assigns tasks to users.
- **User** only sees tasks assigned to them. They can move tasks between columns, toggle sub-tasks and view the task activity. They cannot create tasks for themselves.

**Core features**
1. JWT authentication (access + refresh tokens) with authorization by role.
2. Advanced tasks: title, description, priority, due date (date + time), tags with colors, sub-task checklist, activity log.
3. Angular Kanban board with drag & drop, search, filters, and a stats dashboard.
4. Production-ready REST API: pagination, schema validation, centralized error handling, security hardening.

---

## 2. Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Database | MongoDB (Mongoose) | Atlas free tier for deployment |
| Backend | Node.js + Express, **JavaScript (ES Modules)** | TypeScript considered for a later project |
| Validation | **Zod** | Unknown fields are stripped, which also prevents mass assignment |
| Auth | JWT (access + refresh), bcrypt (cost 12) | See section 6 |
| Frontend | Angular (standalone components) | Signals + services for state, RxJS for HTTP |
| UI | Angular Material (custom theme) + custom SCSS for cards | |
| Drag & Drop | Angular CDK DragDrop | Optimistic update with rollback |
| Charts | Chart.js via ng2-charts | |
| Hosting | MongoDB Atlas, Render (API), Vercel/Netlify (SPA) | Deploy early (after M1) |

---

## 3. Repository & Folder Structure

Single monorepo with two folders.

```
task-manager/
├── README.md
├── ARCHITECTURE.md
├── backend/
│   ├── src/
│   │   ├── config/            # db.js, env.js (env validated with Zod)
│   │   ├── models/            # *.model.js
│   │   ├── controllers/       # thin: parse request, call service, send response
│   │   ├── services/          # business logic + authorization rules
│   │   ├── routes/
│   │   ├── validators/        # Zod schemas
│   │   ├── middlewares/       # auth, role, validate, error handler
│   │   ├── utils/             # ApiError, asyncHandler, token helpers
│   │   ├── app.js             # Express setup (no listen)
│   │   └── server.js          # entry point (listen)
│   ├── tests/
│   ├── .env.example
│   └── package.json
└── frontend/
    └── src/app/
        ├── core/              # services, interceptors, guards, models
        ├── shared/            # reusable components, pipes, directives
        ├── features/          # auth, board, tasks, dashboard, admin
        └── app.routes.ts
```

**Patterns**
- Layered backend: `routes -> controller -> service -> model`.
- Controllers stay thin; business logic and authorization live in services.
- `app.js` is separate from `server.js` so tests can import the app without starting a server.
- Central error handler plus `asyncHandler` instead of try/catch in every controller.
- File naming: `task.controller.js`, `task.service.js`, ... (keeps a later move to a feature-based layout easy).

---

## 4. Data Model (MongoDB)

### User
`name`, `email` (unique, lowercase), `password` (bcrypt hash, `select: false`), `role` (`admin` | `user`, default `user`), `avatar`, `isActive`, `lastLoginAt`, timestamps.
Indexes: `email` (unique), `role`.

### RefreshToken
`user` (ref, indexed), `tokenHash` (hash only, never the raw token), `expiresAt` (**TTL index**), `revokedAt`, `userAgent`, `ip`.

### Board
`title`, `description`, `createdBy` (admin), `columns[] { _id, name, order, color }` (custom; default To Do / In Progress / Done), `isArchived`, `members[]` (empty for now, reserved for future collaboration).

### Task
`title`, `description`, `board` (ref), `columnId` (id of a column inside the board, acts as the status), `createdBy` (admin), `assignee` (ref, required, indexed), `priority` (`low` | `medium` | `high` | `urgent`), `dueDate` (date + time), `position` (Number, order inside a column), `tags[]` (refs), `subtasks[] { _id, title, isDone, order }` (embedded), `completedAt`, `isArchived`, `archivedAt`, timestamps.

Indexes:
- `{ board, columnId, position }` — Kanban loading
- `{ assignee, isArchived, dueDate }` — "my tasks" and overdue
- text index on `title` + `description` — search

### Tag
`name` (unique), `color` (hex, validated), `createdBy` (admin). Shared across the workspace.

### ActivityLog (separate collection, key events only, no TTL)
`task`, `board`, `actor`, `action` (`created`, `status_changed`, `priority_changed`, `due_date_changed`, `reassigned`, `subtask_toggled`, `archived`, `restored`), `changes { field, from, to }`, `createdAt`.
Index: `{ task, createdAt: -1 }`.

### Modeling decisions and why
| Decision | Why |
|---|---|
| Sub-tasks embedded | Small, always read with the task |
| Activity log separate | Grows without bound, paginated, not needed for the board view |
| Tags as a collection with refs | Renaming or recoloring a tag updates everywhere; easy filters and stats |
| `columnId` instead of a status name | Renaming a column does not break tasks |
| Numeric `position` | A drag updates one document only |
| Hashed refresh tokens in their own collection | Multi-device support, TTL cleanup, revoke and rotation |
| Soft delete (`isArchived`) | Safe against mistakes, enables archive/restore |

---

## 5. REST API (`/api/v1`)

**Response envelope:** `{ success, data, meta }`. Errors: `{ success: false, message, errors[] }`.
**Pagination:** page-based (`?page=&limit=`), `meta` carries `page`, `limit`, `total`, `totalPages`. The Kanban endpoint loads all active tasks of a board in a single request.

### Auth
`POST /auth/register` (always creates a `user`), `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me`.

### Users (admin)
`GET /users` (search + pagination), `PATCH /users/:id` (role, isActive).

### Boards & Columns
`GET /boards`, `GET /boards/:id` (admin: all, user: boards where they have tasks), `POST /boards` (admin), `PATCH /boards/:id` (admin), `DELETE /boards/:id` (admin, archives).
`POST /boards/:id/columns`, `PATCH /boards/:id/columns/:colId`, `DELETE /boards/:id/columns/:colId` (admin). Deleting a column that still has tasks returns **409 Conflict**.

### Tasks
| Method | Path | Admin | User |
|---|---|---|---|
| GET | `/boards/:id/tasks` (Kanban) | all tasks | own only |
| GET | `/tasks` (filters + pagination) | yes | own only |
| GET | `/tasks/:id` | yes | own only |
| POST | `/tasks` | yes | no |
| PATCH | `/tasks/:id` | yes | no |
| PATCH | `/tasks/:id/assign` | yes | no |
| PATCH | `/tasks/:id/move` (`{ columnId, position }`) | yes | yes (own) |
| DELETE | `/tasks/:id` (archive), POST `/tasks/:id/restore` | yes | no |
| POST/DELETE | `/tasks/:id/subtasks` | yes | no |
| PATCH | `/tasks/:id/subtasks/:subId` | yes | toggle only |
| GET | `/tasks/:id/activity` | yes | yes (own) |

Filters on `GET /tasks`: `q`, `board`, `columnId`, `priority`, `tag`, `assignee` (admin only), `dueFrom`, `dueTo`, `overdue`, `sort`, `page`, `limit`.

### Tags & Stats
`GET /tags` (all), `POST/PATCH/DELETE /tags` (admin). `GET /stats/overview` (admin: global, user: own).

### Authorization rule
Authorization is enforced in the **service layer**, not only in the UI. A user requesting someone else's task gets **404** (not 403) so the task's existence is not leaked. For users, every task query is scoped with `assignee = currentUser`.

---

## 6. Authentication & Security

**Flow**
1. Login: verify password, issue an access token (JSON body) and a refresh token (httpOnly cookie); store only the **hash** of the refresh token.
2. Each request sends `Authorization: Bearer <access>`; middleware sets `req.user`.
3. On `401`, the Angular interceptor calls `POST /auth/refresh` (cookie sent automatically), receives a new pair, and retries the request.
4. **Rotation + reuse detection:** every refresh revokes the old token and issues a new one; reusing a revoked token revokes all of that user's tokens.
5. Logout: revoke the token and clear the cookie.
6. On app start, `APP_INITIALIZER` calls refresh so a page reload keeps the session.

**Token and cookie settings**
- Access token: 15 minutes, kept in memory only. Refresh token: 7 days.
- Cookie: `httpOnly`, `secure` in production, `sameSite` and domain settings driven by env (assume frontend and API on different domains), path limited to `/api/v1/auth`.

**First admin:** created automatically on server start from `.env`. Safeguards: idempotent (never duplicates or overwrites), server refuses to start if `ADMIN_EMAIL` / `ADMIN_PASSWORD` are missing or weak, password is never logged.

**Hardening checklist**
- `helmet` security headers
- CORS whitelist (frontend origin only, credentials enabled)
- Rate limiting, stricter on `/auth/login` and `/auth/register`
- `express-mongo-sanitize` (NoSQL injection)
- Body size limit
- Zod schemas whitelist allowed fields (mass assignment)
- Generic login error ("Email or password incorrect")
- `.env` out of git, `.env.example` committed
- Password policy: at least 8 characters, with a letter and a number

---

## 7. Frontend Architecture (Angular)

- Standalone components, lazy-loaded feature routes.
- State: Signals in services; RxJS for HTTP and interceptors.
- Interceptors: attach access token, handle refresh on 401, global error handling.
- Guards: `authGuard`, `guestGuard`, `adminGuard`. Guards improve UX only; real protection is in the backend.
- Drag & drop: Angular CDK, optimistic update, rollback and error message if the API call fails.

| Route | Page | Guard |
|---|---|---|
| `/login`, `/register` | Auth | guest |
| `/boards` | Boards list | auth |
| `/boards/:id` | Kanban | auth |
| `/tasks` | My tasks (filters + search) | auth |
| `/dashboard` | Stats | auth |
| `/admin/users` | User management | auth + admin |
| `/admin/tags` | Tag management | auth + admin |

---

## 8. Working Agreements

- **Build in vertical slices:** each milestone delivers backend and frontend for a feature, so a working app exists after every milestone.
- **Git:** feature branches, merge to `main` when working, Conventional Commits (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`).
- **Testing:** Postman for exploratory testing (collection exported to the repo). Automated tests (Jest + Supertest) for critical paths: auth with token rotation/reuse, and task authorization. *Final testing scope to be confirmed.*
- **Deploy early** after M1 to surface CORS and cookie issues sooner.
- **Per milestone protocol:** mini-plan, approval, code in small batches with explanation, run and verify, small challenge, commit.
- **Docs:** Swagger/OpenAPI for the API, README with screenshots and a drag & drop GIF.

---

## 9. Roadmap

| # | Milestone | Scope | Size | Definition of Done |
|---|---|---|---|---|
| M0 | Foundations | Repo, git, backend skeleton (app/server, env validation, DB, error handler, ApiError, asyncHandler), Angular skeleton, ESLint/Prettier | S | `/api/v1/health` responds, Angular app opens |
| M1 | Auth | User + RefreshToken, register/login/refresh/logout/me, admin bootstrap, security middlewares; Angular login/register, interceptor, guards, APP_INITIALIZER | L | Login works, reload keeps session, reuse detection works. **First deploy** |
| M2 | Boards & Columns | Board model, admin CRUD, column rules (409); Angular boards list + column management | M | Admin manages boards and columns |
| M3 | Core Tasks | Tag + Task models, CRUD, assign, sub-tasks; Angular Kanban (no drag) + task dialog | L | Admin creates and assigns; user sees only own tasks |
| M4 | Drag & Drop + Activity | `PATCH /move`, position logic, ActivityLog; CDK DragDrop with optimistic update | M | Moves persist; activity shows on the task |
| M5 | Search, Filters, Admin | `GET /tasks` filters + pagination; users and tags admin pages | M | Filters and admin pages work |
| M6 | Dashboard | `/stats/overview` + charts | S | Correct charts per role |
| M7 | Polish & Ship | Tests, Swagger, README, final deploy, LinkedIn post | M | Live link + README ready for the CV |

If time gets tight, M0–M4 already deliver a complete Kanban with auth and permissions; M5–M7 are upgrades.

---

## 10. Open Items

- Final testing scope (Postman only vs. Postman + automated critical-path tests).
- Exact cookie `sameSite` / `secure` values once the hosting domains are known.
- Optional later: board members and collaboration, column deletion with `moveTasksTo`, reminders and notifications, TypeScript migration.
