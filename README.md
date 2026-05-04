# Laundry Management System

> **A campus laundromat, but the part nobody sees: status tracking for students, mobile data entry for staff, dashboards for the operator. Built as a group project for one of the actual laundromats on our campus.**

The on-campus laundromat at our university ran on a paper ledger and shouted updates ("Number 24, ready!"). The owner couldn't tell you which day of the week was busiest, students didn't know when their clothes were done, and staff wrote ticket numbers on a whiteboard. We built the software they should have had.

This was a team project — I worked primarily on the server and the web client; the mobile staff client and most of the database design came from teammates. The fork relationship on this repo is from when we were branching off the original organization's copy.

## Architecture

```
┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
│  web_client      │       │  StaffClient     │       │  server          │
│  (SvelteKit)     │       │  (Expo / RN)     │       │  (Sanic, Python) │
│                  │       │                  │       │                  │
│ • student status │ ─────▶│ • data entry     │ ─────▶│ • JWT auth       │
│ • owner analytics│       │ • status updates │       │ • REST API       │
└──────────────────┘       └──────────────────┘       └────────┬─────────┘
                                                               │
                                                      ┌────────▼─────────┐
                                                      │  MariaDB         │
                                                      │  (in Docker)     │
                                                      └──────────────────┘
```

### Three audiences, three frontends

- **`web_client/` — for students and the owner.** Students log in, see "your laundry: washing / drying / ready / collected." Owner gets the same auth but lands on dashboards built with `@carbon/charts-svelte` — daily volume, peak hours, revenue, machine utilization. Built with SvelteKit + Tailwind + bits-ui.
- **`StaffClient/` — for the staff on the floor.** Mobile-first because nobody at the counter wants to navigate a desktop UI mid-shift. Expo/React Native. The flow is: scan a ticket → mark in/out of each stage → push status updates the student's web app picks up immediately.
- **`server/` — the API in the middle.** Sanic (async Python) over MariaDB. JWT auth so the staff client can persist a session offline-ish. CORS configured for the LAN deployment we expected.

### Data layer

`db_container_linux/` and `db_container_windows/` ship a Docker Compose with MariaDB and the schema bootstrap (`lms.sql`) so any teammate can spin up the DB locally without messing with their host installation. The two folders are because half the team was on Linux and half on Windows, and the Docker volume mount semantics differed enough to warrant separate compose files.

## Running locally

```bash
# 1. Database
cd db_container_linux   # or db_container_windows
docker compose up -d

# 2. Server
cd ../server
cp .env.example .env    # set JWT_SECRET + DB creds
pip install -e .        # uses pyproject.toml
sanic app:app --host 0.0.0.0 --port 8000

# 3. Web client
cd ../web_client
pnpm install && pnpm dev

# 4. Staff client
cd ../StaffClient
npm install
./update_ip.sh          # injects your dev machine's LAN IP into the API base URL
npx expo start
```

The staff client needs to be on the same Wi-Fi as the server during development — `update_ip.sh` writes the current LAN IP into the app's config so the phone can reach the API.

## What we shipped

- Full auth (token-based) end-to-end across all three frontends
- Student-facing live status updates
- Operator dashboard with daily/weekly/monthly revenue + volume views
- Staff workflow for marking laundry through wash → dry → ready → collected
- A Docker setup the next year's team could pick up cleanly

## What we didn't get to

- Push notifications when laundry is ready (we settled for in-app polling)
- Payment integration — the laundromat still handles that on UPI separately
- A proper migrations system; schema changes were SQL files merged manually

## Why I'm proud of it

Three people, three different stacks, one shared backend, working at the end of the semester for a real customer. The integration testing we did (literally walking to the laundromat with a phone and a laptop) caught more bugs than any unit test we wrote.
