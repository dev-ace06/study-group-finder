# Study Group Finder - Complete Application Documentation

## 1. Application Overview

Study Group Finder is a Flask web application that helps students:

- register and log in
- manage a profile (subjects, skill level, availability)
- create and browse study groups
- join public groups or join private groups via invite link
- manage group members and roles (admin, moderator, member)
- use premium-only features (group chat, analytics, advanced creator filter, private groups)
- view a dashboard with joined groups, recommendations, usage, and upcoming sessions
- view plans and upgrade via Stripe checkout with payment verification

The app is server-rendered with Jinja templates and uses SQLAlchemy models with SQLite locally (and PostgreSQL-compatible configuration for deployment).

## 2. Tech Stack

Backend:

- Flask 3.0.3
- Flask-Login 0.6.3
- Flask-SQLAlchemy 3.1.1
- Flask-Migrate 4.0.7
- python-dotenv 1.0.1

Frontend:

- Jinja2 templates
- Bootstrap 5.3.3 (CDN)
- Bootstrap Icons 1.11.3 (CDN)
- Google Font: Space Grotesk
- Custom CSS and JS

Database:

- SQLite for local development (`instance/app.db`)
- PostgreSQL-compatible URI handling in production (`postgres://` rewritten to `postgresql://`)

Deployment:

- gunicorn 22.0.0
- render.yaml blueprint included

## 3. Project Structure

```text
study-group-finder/
  app.py
  APPLICATION_DOCUMENTATION.md
  README.md
  config.py
  extensions.py
  requirements.txt
  render.yaml
  fix_db.py
  fix_invite.py
  inspect_db.py
  database/
    __init__.py
    db.py
    schema.sql
  instance/
  migrations/
    alembic.ini
    env.py
    README
    script.py.mako
    versions/
      4cc4e3c3acc0_add_roles_invite_sessions_constraints.py
  models/
    __init__.py
    user.py
    group.py
    membership.py
    session.py
    subscription.py
    chat_message.py
  routes/
    __init__.py
    auth_routes.py
    group_routes.py
    user_routes.py
    chat_routes.py
    subscription_routes.py
  services/
    __init__.py
    group_service.py
    chat_service.py
    subscription_service.py
  utils/
    __init__.py
    validators.py
    matcher.py
    decorators.py
  static/
    style.css
    script.js
  templates/
    base.html
    index.html
    login.html
    register.html
    dashboard.html
    create_group.html
    browse_groups.html
    profile.html
    group_chat.html
    group_analytics.html
    group_manage.html
    subscription_plans.html
```

## 4. Configuration

Defined in `config.py`:

- `BaseConfig`:
  - `SECRET_KEY` from env or fallback
  - `SQLALCHEMY_DATABASE_URI` from `DATABASE_URL` or local SQLite path
  - `SQLALCHEMY_TRACK_MODIFICATIONS = False`
  - Stripe/billing settings:
    - `STRIPE_SECRET_KEY`
    - `STRIPE_PUBLISHABLE_KEY`
    - `STRIPE_PRICE_MONTHLY_ID`
    - `STRIPE_PRICE_YEARLY_ID`
    - `PREMIUM_MONTHLY_PRICE`
    - `PREMIUM_YEARLY_PRICE`
- `DevelopmentConfig`:
  - `DEBUG = True`
- `ProductionConfig`:
  - `DEBUG = False`
  - secure session/remember cookies enabled
- `CONFIG_MAP` supports `development|dev|production|prod`

Database URL handling:

- If provider gives `postgres://...`, it is rewritten to `postgresql://...` for SQLAlchemy compatibility.

## 5. App Initialization and Wiring

Defined in `app.py`:

1. Builds Flask app with `instance_relative_config=True`
2. Selects config from `FLASK_CONFIG` (default: development)
3. Initializes extensions via `init_extensions(app)`:
   - SQLAlchemy (`db`)
   - Flask-Login (`login_manager`)
   - Flask-Migrate (`migrate`)
4. Registers blueprints:
   - `auth_bp`
   - `chat_bp`
   - `group_bp`
   - `subscription_bp`
   - `user_bp`
5. Injects global template context for plan status and usage snapshot
6. Exposes home route `/`
7. Defines CLI commands:
   - `flask init-db`
   - `flask seed-db`
8. Ensures tables exist with `db.create_all()` inside app context

## 6. Authentication and Session Handling

Defined in `extensions.py` and `models/user.py`:

- Login manager settings:
  - `login_view = "auth.login"`
  - `login_message_category = "warning"`
- `User` uses `UserMixin`
- Password security:
  - `set_password()` -> `generate_password_hash`
  - `check_password()` -> `check_password_hash`
- User loader:
  - converts session ID to `int`
  - fetches user by primary key

## 7. Data Model

### 7.1 User (`models/user.py`)

Fields:

- `id` (PK)
- `name` (required)
- `email` (unique, required)
- `password_hash` (required)
- `subjects` (text, default empty)
- `skill_level` (`Beginner|Intermediate|Advanced`, default `Beginner`)
- `availability` (text, default empty)
- `created_at` (UTC timestamp)

Constraints/Indexes:

- check: skill level in allowed set
- check: name not blank after trim
- index: email

Relationships:

- `memberships` -> `GroupMember`
- `created_groups` -> `StudyGroup`
- `subscriptions` -> `Subscription` (latest first)
- `chat_messages` -> `ChatMessage`

### 7.2 StudyGroup (`models/group.py`)

Fields:

- `id` (PK)
- `subject` (required)
- `description` (required)
- `schedule` (required)
- `max_members` (required, default 5)
- `is_private` (default `False`, indexed)
- `invite_token` (unique, generated via `secrets.token_urlsafe`)
- `creator_id` (FK -> `user.id`)
- `created_at` (UTC timestamp)

Constraints/Indexes:

- check: `max_members >= 2`
- check: subject not blank after trim
- index: subject

Relationships:

- `creator` -> `User`
- `members` -> `GroupMember`
- `sessions` -> `GroupSession`
- `chat_messages` -> `ChatMessage`

Computed property:

- `current_member_count`: `len(self.members)`

### 7.3 GroupMember (`models/membership.py`)

Fields:

- `user_id` (FK -> `user.id`, composite PK)
- `group_id` (FK -> `study_group.id`, composite PK)
- `role` (`admin|moderator|member`, default `member`)
- `joined_at` (UTC timestamp)

Constraints/Indexes:

- check: role in allowed set
- index: `(group_id, role)`

Relationships:

- `user` -> `User`
- `group` -> `StudyGroup`

### 7.4 GroupSession (`models/session.py`)

Fields:

- `id` (PK)
- `group_id` (FK -> `study_group.id`)
- `title` (required)
- `starts_at` (required, indexed)
- `duration_minutes` (required, default 60)
- `notes` (text, default empty)
- `created_at` (UTC timestamp)

Constraints/Indexes:

- check: `duration_minutes >= 15`
- index: `(group_id, starts_at)`

Relationship:

- `group` -> `StudyGroup`

### 7.5 Subscription (`models/subscription.py`)

Fields:

- `id` (PK)
- `user_id` (FK -> `user.id`)
- `plan_type` (`free|premium`, default `free`)
- `start_date` (required)
- `end_date` (required)
- `status` (`active|expired`, default `active`)
- `stripe_subscription_id` (nullable)
- `created_at` (UTC timestamp)

Constraints/Indexes:

- check: valid `plan_type`
- check: valid `status`
- check: `end_date > start_date`
- index: `(user_id, end_date)`

Relationship:

- `user` -> `User`

### 7.6 ChatMessage (`models/chat_message.py`)

Fields:

- `id` (PK)
- `group_id` (FK -> `study_group.id`)
- `user_id` (FK -> `user.id`)
- `content` (required)
- `created_at` (UTC timestamp)

Constraints/Indexes:

- check: content not blank after trim
- index: `(group_id, created_at)`

Relationships:

- `group` -> `StudyGroup`
- `user` -> `User`

## 8. Service Layer and Business Rules

### 8.1 Group service (`services/group_service.py`)

Responsibilities:

- create group (validations + free/premium policy)
- browse filtering
- join/leave rules
- ownership transfer on creator leave
- role updates and removals for admins
- upcoming sessions retrieval
- group analytics aggregation

Important rules:

- free users:
  - create limit: 3 groups
  - join limit: 5 groups
- private groups require premium and invite-based access
- cannot join full groups
- creator must transfer ownership before leaving own group
- creator cannot be demoted below admin or removed

### 8.2 Subscription service (`services/subscription_service.py`)

Responsibilities:

- resolve active subscription
- evaluate premium access
- usage snapshot generation
- Stripe checkout session creation for monthly/yearly plans
- payment verification from Stripe checkout session
- premium activation only after verified paid checkout

### 8.3 Chat service (`services/chat_service.py`)

Responsibilities:

- fetch ordered group messages (limit default 120)
- post message with validation

Message constraints:

- non-empty content
- maximum length 800 chars
- sender must be a member of the group

## 9. Utility Logic

### 9.1 Validators (`utils/validators.py`)

- `normalize_text(value)`: trims input
- `split_csv_field(value)`: parses comma-separated text into clean tokens

### 9.2 Matching (`utils/matcher.py`)

`suggest_groups_for_user(user, groups, joined_group_ids, limit=5, weights=None)`:

- excludes joined groups
- excludes private groups
- excludes full groups
- weighted score components:
  - exact subject match
  - subject keyword overlap
  - availability token overlap
  - open-seat ratio bonus
  - recent-group bonus (fades over 14 days)
- returns top results sorted by score, seats, recency

### 9.3 Premium gate decorator (`utils/decorators.py`)

- `premium_required`:
  - redirects unauthenticated users to login
  - redirects non-premium users to plans page with flash warning

## 10. Route Reference

### 10.1 Root route (`app.py`)

- `GET /`
  - public home page
  - shows latest public groups (up to 6)

### 10.2 Auth routes (`routes/auth_routes.py`)

- `GET, POST /register`
  - redirects authenticated user to dashboard
  - validates required fields
  - enforces unique email
  - stores hashed password

- `GET, POST /login`
  - redirects authenticated user to dashboard
  - validates credentials
  - starts user session

- `GET /logout`
  - requires auth
  - ends session

### 10.3 User routes (`routes/user_routes.py`)

- `GET /dashboard`
  - requires auth
  - shows joined groups, suggested groups, upcoming sessions
  - includes premium flag and usage snapshot

- `GET, POST /profile`
  - requires auth
  - updates profile fields
  - validates non-empty name

### 10.4 Group routes (`routes/group_routes.py`)

Blueprint prefix: `/groups`

- `GET, POST /groups/create`
  - requires auth
  - validates fields and max members
  - supports premium-only private groups
  - auto-adds creator as admin member

- `GET /groups/browse`
  - requires auth
  - filters:
    - `subject`
    - `time`
    - `creator` (premium only)
  - returns public groups list and membership context

- `POST /groups/<group_id>/join`
  - requires auth
  - joins public group when capacity and plan limits allow

- `GET /groups/invite/<invite_token>`
  - requires auth
  - joins target group by invite token (supports private groups)

- `POST /groups/<group_id>/leave`
  - requires auth
  - leaves group
  - creator must transfer ownership first

- `GET /groups/<group_id>/analytics`
  - requires auth + premium
  - user must be group member
  - shows aggregated engagement metrics

- `GET, POST /groups/<group_id>/manage`
  - requires auth
  - admin-only
  - role updates and member removal actions

### 10.5 Chat routes (`routes/chat_routes.py`)

Blueprint prefix: `/chat`

- `GET, POST /chat/<group_id>`
  - requires auth + premium
  - user must be group member
  - GET: list chat messages
  - POST: submit message

### 10.6 Subscription routes (`routes/subscription_routes.py`)

Blueprint prefix: `/subscription`

- `GET /subscription/plans`
  - requires auth
  - shows plan cards, usage, active subscription
  - shows configured monthly/yearly display pricing

- `POST /subscription/upgrade`
  - requires auth
  - creates Stripe Checkout session (`cycle=monthly|yearly`)
  - redirects user to Stripe hosted checkout page

- `GET /subscription/checkout/success`
  - requires auth
  - verifies checkout session belongs to current user
  - verifies Stripe payment status is `paid`
  - activates premium subscription

- `GET /subscription/checkout/cancel`
  - requires auth
  - handles canceled checkout and returns to plans page

## 11. Frontend Templates

- `base.html`: app shell, navbar, plan badge, flash stack
- `index.html`: marketing hero + recent public groups
- `login.html`: sign-in form
- `register.html`: sign-up form
- `dashboard.html`: joined groups, recommendations, upcoming sessions, quick actions
- `create_group.html`: group creation form with premium private toggle
- `browse_groups.html`: filtering, join/leave, invite copy, management links
- `group_manage.html`: admin member/role management table
- `group_analytics.html`: premium metrics and activity breakdown
- `group_chat.html`: premium in-group chat UI
- `profile.html`: editable profile
- `subscription_plans.html`: free vs premium plans and upgrade actions

## 12. Static Assets

- `static/style.css`
  - design tokens in `:root`
  - responsive dashboard/auth/group UI styling
  - premium and utility badges/cards

- `static/script.js`
  - staged reveal animation delays
  - loading-state submit buttons (`.js-loading-form`)
  - invite-link copy-to-clipboard handling
  - auto-scroll chat container to newest message

## 13. Database and Migrations

Runtime ORM table names:

- `user`
- `study_group`
- `group_member`
- `group_session`
- `subscription`
- `chat_message`

Migration setup:

- Alembic environment under `migrations/`
- Existing migration: `4cc4e3c3acc0_add_roles_invite_sessions_constraints.py`
  - adds member role
  - adds invite token
  - adds indexes for lookup patterns

Schema reference:

- `database/schema.sql` contains SQL Server-style DDL mirroring current domain entities and constraints.

## 14. Seed Data

CLI command: `flask seed-db`

Behavior:

- creates tables if needed
- skips if any user already exists
- inserts:
  - 3 users
  - 3 study groups
  - role-based memberships
  - 3 upcoming sessions
  - 1 active premium subscription
  - 1 chat message

Sample accounts:

- `ava@example.com` / `password123`
- `liam@example.com` / `password123`
- `noah@example.com` / `password123`

## 15. Commands and Local Development

From project root (`study-group-finder`):

1. Create venv

```powershell
python -m venv .venv
```

2. Activate venv

```powershell
.\.venv\Scripts\Activate.ps1
```

3. Install dependencies

```powershell
pip install -r requirements.txt
```

4. Run app

```powershell
python app.py
```

5. Flask CLI setup

```powershell
$env:FLASK_APP = "app.py"
```

6. Initialize/seed database

```powershell
flask init-db
flask seed-db
```

7. Run migrations (when using Alembic workflow)

```powershell
flask db upgrade
```

## 16. Utility and Maintenance Scripts

- `fix_db.py`: direct SQLite patch helper (legacy/manual)
- `fix_invite.py`: adds invite column manually (legacy/manual)
- `inspect_db.py`: quick table/column inspection helper

These scripts are pragmatic repair utilities, but migration commands are the preferred schema management path.

## 17. Known Behaviors and Constraints

- Most features require authenticated session.
- Chat and analytics are premium-gated.
- Free plan limits are enforced in service layer:
  - max 3 created groups
  - max 5 joined groups
- Private groups are invite-based and premium-controlled.
- Browse listing shows public groups only.
- Group creator ownership must be transferred before leaving a self-created group.
- Recommendation engine is deterministic weighted heuristics, not ML.
- Premium is activated only after Stripe-verified paid checkout.

## 18. Suggested Enhancements

- Add route/service unit tests and end-to-end UI tests.
- Add CSRF protection for forms if not already enabled by extension stack.
- Add pagination and sort controls for large browse sets.
- Add session creation/edit routes and UI for real scheduling workflows.
- Add Stripe webhooks to sync renewals, cancellations, and failed payments.
- Add API mode (JSON endpoints) for mobile/client consumption.

## 19. Quick Troubleshooting

- Styling appears broken:
  - verify CDN access for Bootstrap/Icons/Google Fonts
  - hard refresh browser cache

- Unexpected redirects to plans:
  - confirm premium-required features are being accessed intentionally
  - check active subscription status in DB

- Group join blocked:
  - check if group is full
  - check free plan limits
  - for private groups, use invite URL

- Migration mismatch/errors:
  - run `flask db upgrade`
  - verify `DATABASE_URL` points to expected database

- Seed skipped:
  - expected when at least one user already exists
