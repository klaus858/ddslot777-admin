# ddslot777-admin

DD Slot admin panel built with Vue 3, Element Plus, and Vite.

Live admin domain:

- https://ddslot777888.com

Demo login:

- Username: admin
- Password: admin123

Backend API:

- Base URL: https://ddslot777-api.vercel.app
- Health check: `GET /api/health`
- Contract check: `GET /api/contract`
- Contract version: `admin-v1`

Data flow rules:

- Admin data must come from the Go API.
- If the API is unavailable or returns an incompatible response shape, the admin panel shows an API error instead of silent mock data.
- Deposit confirmation calls `POST /api/admin/deposits/confirm`.
- Withdrawal approval/rejection calls `POST /api/admin/withdrawals/approve` or `POST /api/admin/withdrawals/reject`.
- Balance edits call `POST /api/admin/users/balance`.
- After every write action, the admin panel reloads summary, users, deposits, withdrawals, and audit logs from the API so the UI and backend state stay aligned.

Next production step: connect the Go API to a real database so members, balances, deposits, activities, and audit logs persist after deployment restarts.
