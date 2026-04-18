# Study Group Finder App

A beginner-friendly full-stack Flask web app to help students discover and join study groups.

## Features

- User authentication (register, login, logout)
- User profile management (subjects, skill level, availability)
- Create study groups (subject, description, schedule, max members)
- Browse and filter groups by subject and schedule
- Join and leave groups
- Simple matching suggestions on dashboard
- Dashboard with joined groups and upcoming sessions
- Premium upgrade via Stripe Checkout (activated only after verified payment)
- SQLite database for local development

## Tech Stack

- Backend: Flask + Flask-SQLAlchemy + Flask-Login
- Frontend: Flask templates (Jinja2) + basic CSS
- Database: SQLite

## Folder Structure

```
study-group-finder/
├── app.py
├── config.py
├── requirements.txt
├── README.md
├── templates/
├── static/
├── models/
├── routes/
├── database/
├── utils/
└── instance/
```

## Run Locally

1. Open a terminal in the project folder.
2. Create and activate a virtual environment.

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

3. Install dependencies.

```powershell
pip install -r requirements.txt
```

4. Run the app.

```powershell
python app.py
```

5. Open `http://127.0.0.1:5000` in your browser.

## Seed Sample Data

Use Flask CLI commands to initialize and seed the database:

```powershell
$env:FLASK_APP = "app.py"
flask init-db
flask seed-db
```

### Sample Login Accounts

After seeding:

- `ava@example.com` / `password123`
- `liam@example.com` / `password123`
- `noah@example.com` / `password123`

## Notes

- To use a custom secret key, set `SECRET_KEY` in environment variables.
- To use a custom database path, set `DATABASE_URL`.

## Deploy on Render

This project includes a Render blueprint file at `render.yaml`.

### Option 1: Deploy with Blueprint (Recommended)

1. Push this repository to GitHub.
2. In Render, click **New +** -> **Blueprint**.
3. Connect your GitHub repository.
4. Render will detect `render.yaml` and create:
	 - one web service (`study-group-finder`)
	 - one PostgreSQL database (`study-group-finder-db`)

### Option 2: Manual Service Setup

Use these settings for the web service:

- Build command: `pip install -r requirements.txt`
- Start command: `gunicorn app:app`
- Environment variables:
	- `FLASK_CONFIG=production`
	- `SECRET_KEY=<your-random-secret>`
	- `DATABASE_URL=<your-render-postgres-connection-string>`
	- `STRIPE_SECRET_KEY=<your-stripe-secret-key>`
	- `STRIPE_PUBLISHABLE_KEY=<your-stripe-publishable-key>`
	- `STRIPE_PRICE_MONTHLY_ID=<your-stripe-monthly-price-id>`
	- `STRIPE_PRICE_YEARLY_ID=<your-stripe-yearly-price-id>`
	- `PREMIUM_MONTHLY_PRICE=$9.99` (display label)
	- `PREMIUM_YEARLY_PRICE=$89.99` (display label)

### After First Deploy

Open the Render web service shell and run:

```bash
flask db upgrade
```

Optional sample data:

```bash
flask seed-db
```
