# CLAUDE.md — Obsidian Ledger

This file is the source of truth for any AI agent working on this project.
Read it fully before writing a single line of code.

---

## Project Overview

**Obsidian Ledger** is a personal budget management app with a dark, glassmorphic
dashboard aesthetic. It is currently built as a Vite + Vanilla JS web SPA that will
also power a future Android app via the same REST API.

**Stack at a glance:**
- Frontend: Vite + Vanilla JS (SPA, hash-based routing)
- Backend: Node.js + Express.js
- Database: Supabase (PostgreSQL + Auth + RLS)
- Charts: Chart.js
- Fonts: Manrope (display) + Inter (body)
- Design: Dark glassmorphism, obsidian palette, no light mode

---

## Repository Structure

```
/
├── frontend/
│   ├── index.html
│   ├── .env                  # VITE_API_BASE_URL
│   ├── src/
│   │   ├── main.js           # SPA router
│   │   ├── auth.js           # Token storage helpers
│   │   ├── api.js            # Central fetch wrapper (replaces data.js)
│   │   ├── screens/
│   │   │   ├── login.js
│   │   │   ├── budgetHome.js
│   │   │   ├── transactions.js
│   │   │   ├── analysis.js
│   │   │   └── trends.js
│   │   └── components/
│   │       ├── navbar.js
│   │       ├── alertBanner.js
│   │       └── budgetCard.js
│   └── style.css
│
└── backend/
    ├── .env                  # Supabase credentials + PORT
    ├── index.js              # Express entry point
    ├── routes/
    ├── controllers/
    ├── middleware/
    └── services/
```

---

## Design System (Non-Negotiable)

### Color Palette (CSS Variables)
```css
--color-canvas:       #0e0e0e   /* page background */
--color-container:    #131313   /* low-level cards */
--color-surface:      #1a1a1a   /* elevated cards */
--color-accent:       #94aaff   /* primary accent */
--color-accent-dim:   #809bff   /* secondary accent */
--color-text-primary: #e8e8f0
--color-text-muted:   #8888a0
--color-border:       rgba(148, 170, 255, 0.15)  /* ghost border */
```

### Typography
- **Manrope** — all display text, headings, large numbers
- **Inter** — body text, labels, table content
- Scale: `display-lg`, `headline-md`, `title-md`, `body-md`, `label-sm`

### Glassmorphism Rules
- Backdrop blur: `24px`
- Card border: `1px solid rgba(148, 170, 255, 0.15)` (ghost border, no solid lines)
- Ambient shadow: `0 0 40px rgba(148, 170, 255, 0.06)`
- Border radius: `12px` (cards), `16px` (modals), `8px` (buttons/inputs)

### No-Line Rule
Never use solid divider lines between sections. Use spacing and layered
background depths to separate content instead.

---

## Authentication

- Supabase Auth handles JWT issuance and refresh tokens
- Tokens are stored in `localStorage` under the key `ob_token`
- Every authenticated API request must include `Authorization: Bearer <token>`
- On any `401` response, clear the token and redirect to `#login`
- `auth.js` exports: `saveToken`, `getToken`, `clearToken`, `isLoggedIn`

---

## API Contract

**Base URL:** `http://localhost:3000/api/v1` (configured via `VITE_API_BASE_URL`)

**All responses follow this envelope:**
```json
{ "success": true, "data": { ... } }
{ "success": false, "error": "Human-readable message" }
```

**All routes require `Authorization: Bearer <token>` except `/auth/*`.**

### Auth
| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| POST | `/auth/register` | `{ email, password }` | Create account |
| POST | `/auth/login` | `{ email, password }` | Login, returns `access_token` |
| POST | `/auth/logout` | — | Logout |

### Budgets
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/budgets?month=YYYY-MM-01` | Get budget + categories for a month |
| POST | `/budgets` | Create a monthly budget |
| PUT | `/budgets/:id` | Update total or currency symbol |
| DELETE | `/budgets/:id` | Delete a budget |

### Budget Categories
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/budgets/:budgetId/categories` | Add a category |
| PUT | `/categories/:id` | Update a category |
| DELETE | `/categories/:id` | Delete a category |

### Transactions
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/transactions?range=week\|month\|3month\|6month` | List transactions |
| GET | `/transactions/summary?month=YYYY-MM-01` | Totals per category + remaining budget |
| POST | `/transactions` | Add a transaction manually |
| PUT | `/transactions/:id` | Edit a transaction |
| DELETE | `/transactions/:id` | Delete a transaction |
| GET | `/transactions/export?range=...&format=pdf\|csv` | Download export |
| POST | `/transactions/ingest/google-pay` | Ingest a parsed Google Pay receipt |

### Trends
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/trends?range=week\|month\|3month\|6month` | Aggregated spending over time |

### Alerts
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/alerts?month=YYYY-MM-01` | In-app alert objects for the month |

**Alert object shape:**
```json
{
  "type": "warning" | "exceeded" | "total_exceeded",
  "category_name": "Food",
  "allocated": 800,
  "spent": 720,
  "percentage": 90
}
```
Alert thresholds: `warning` at ≥ 80%, `exceeded` when over 100%, `total_exceeded` for overall budget.

---

## Database Schema (Supabase)

All tables use `user_id uuid FK → auth.users(id)`.
All tables have Row Level Security (RLS) enabled — users see only their own rows.

### `budgets`
| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| user_id | uuid | FK |
| month | date | First day of month (e.g. 2025-04-01) |
| total_amount | numeric | |
| currency_symbol | text | Default `₪` |
| created_at | timestamp | |

### `budget_categories`
| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| budget_id | uuid | FK → budgets |
| user_id | uuid | FK |
| name | text | e.g. "Food", "Utilities" |
| allocated_amount | numeric | |
| created_at | timestamp | |

### `transactions`
| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| user_id | uuid | FK |
| category_id | uuid | FK → budget_categories (nullable) |
| business_name | text | e.g. "Amazon", "McDonald's" |
| amount | numeric | |
| date | date | |
| is_recurring | boolean | Default false |
| recurrence_frequency | text | "weekly", "monthly", "yearly" (nullable) |
| source | text | "manual", "google_pay", "google_wallet" |
| created_at | timestamp | |

---

## Frontend Screens

### Routing (`main.js`)
Hash-based SPA router. Valid routes:
- `#login` → `screens/login.js`
- `#budget-home` → `screens/budgetHome.js`
- `#transactions` → `screens/transactions.js`
- `#analysis` → `screens/analysis.js`
- `#trends` → `screens/trends.js`

On every route change:
1. Check `isLoggedIn()` — redirect to `#login` if false
2. Call the screen's `init()` function
3. Call `checkAlerts()` (skip on login screen)
4. Update the active tab in the bottom navbar

### Screen Pattern
Every screen module must export an async `init()` that:
1. Shows a skeleton loader immediately
2. Fetches data from the API
3. Renders the screen
4. Shows an error state with a retry button if the fetch fails

```js
export async function init() {
  showSkeleton();
  try {
    const data = await api.get('/...');
    render(data);
  } catch (err) {
    showErrorState(err.message, init); // pass init as retry callback
  }
}
```

### Shared Components

**`budgetCard.js`** — `renderBudgetCard(summary, containerEl)`
Used on both Transactions and Trends screens. Shows total budget, spent, remaining,
and a color-coded progress bar (green → amber at 70% → red at 90%+).

**`alertBanner.js`** — `checkAlerts()`
Fetches `/alerts` and injects dismissible banners at the top of the active screen.
Tracks shown alerts in a `Set` to avoid repeating within the same session.
Auto-dismisses after 6 seconds.

**`navbar.js`** — Bottom navigation bar with 4 tabs.
Highlights the active tab based on the current hash.

---

## Chart.js Configuration

Apply globally in `main.js` before any screen loads:

```js
import Chart from 'chart.js/auto';

Chart.defaults.color = '#8888a0';
Chart.defaults.font.family = 'Inter';
Chart.defaults.plugins.legend.labels.color = '#e8e8f0';
Chart.defaults.plugins.tooltip.backgroundColor = 'rgba(19,19,19,0.95)';
Chart.defaults.plugins.tooltip.titleColor = '#94aaff';
Chart.defaults.plugins.tooltip.bodyColor = '#e8e8f0';
Chart.defaults.plugins.tooltip.borderColor = 'rgba(148,170,255,0.15)';
Chart.defaults.plugins.tooltip.borderWidth = 1;
```

When updating chart data (e.g. on range toggle), always use:
```js
chart.data.datasets[0].data = newData;
chart.update();
```
Never destroy and recreate the chart instance on range changes.

---

## Google Pay Integration

Transactions with `source: "google_pay"` or `source: "google_wallet"` must render
with a Google Pay badge icon in the transactions table, displayed next to the
business name. The badge is informational only — no additional action on click.

The `/transactions/ingest/google-pay` endpoint accepts pre-parsed receipt data:
```json
{ "business_name": "McDonald's", "amount": 45.50, "date": "2025-04-06" }
```
The frontend is responsible for parsing the raw notification/email before calling
this endpoint. The backend only stores the structured data.

---

## Environment Variables

### Frontend (`frontend/.env`)
```
VITE_API_BASE_URL=http://localhost:3000/api/v1
```

### Backend (`backend/.env`)
```
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
PORT=3000
```

Never commit `.env` files. Both are in `.gitignore`.

---

## Backend Rules

- All routes must be prefixed `/api/v1/` for future Android compatibility
- Auth middleware reads `Authorization: Bearer <token>`, verifies via Supabase,
  and attaches the user object to `req.user`
- Use the Supabase JS client query builder — avoid raw SQL unless unavoidable
- Project structure: `/routes`, `/controllers`, `/middleware`, `/services`
- One service file per resource (budgets, transactions, alerts, trends)

---

## Currency

Single currency per user. The symbol is set on the budget (`currency_symbol` field,
default `₪`). The frontend reads this from the budget object and applies it globally
for that session. No conversion logic is needed.

---

## What NOT to Do

- Do not use `data.js` mock data anywhere — it has been replaced by `api.js`
- Do not hardcode any Supabase credentials in source files
- Do not use raw SQL unless the Supabase query builder cannot express the query
- Do not destroy and recreate Chart.js instances on data updates — use `chart.update()`
- Do not use solid divider lines in the UI — use spacing and depth instead
- Do not use any font other than Manrope or Inter
- Do not add light mode styles — this is a dark-only app
- Do not duplicate the Budget Remaining Card logic — use the shared `budgetCard.js` component
- Do not show the same in-app alert twice in the same session

---

## Definition of Done

A feature is complete when:
- [ ] All data comes from the real API (no mock data)
- [ ] JWT is attached to every authenticated request
- [ ] Unauthenticated users are always redirected to `#login`
- [ ] Loading skeletons appear while fetching
- [ ] Error states appear on failure with a working retry button
- [ ] Charts update without page reload when range toggles change
- [ ] Budget Remaining Card is consistent on both Transactions and Trends screens
- [ ] Alerts display correctly and do not repeat in the same session
- [ ] Google Pay badge appears on auto-imported transactions
- [ ] Export triggers a real file download
- [ ] `npm run build` completes with zero errors and zero console errors in the browser