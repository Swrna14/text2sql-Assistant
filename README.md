# Text-to-SQL Assistant

Ask questions about a database in plain English. An LLM (Google Gemini) writes the SQL, a **safety validator** makes sure it is a single read-only query on allowed tables, and the database returns the answer as a table and chart.

Built with **FastAPI, SQLAlchemy, Pydantic, JWT/RBAC and the Gemini API**.

## Features

- Natural language to SQL using Gemini (swappable; a mock provider works without an API key)
- **SQL safety layer**: SELECT-only, single statement, forbidden keywords blocked, table allowlist, automatic `LIMIT`
- **Defence in depth**: read-only transaction, row cap, and a recommendation to use a read-only DB user
- **JWT authentication** and **role-based access control**: `USER` cannot query the `customers` table, `ADMIN` can
- Prompt receives only the schema the caller's role may see
- Query history and audit log (users see their own, admins see all)
- Simple web UI with results table and an automatic bar chart
- Works with SQLite out of the box; MySQL and PostgreSQL via `DATABASE_URL`
- Swagger docs at `/docs`, pytest suite, Docker and GitHub Actions CI

## How it works

```
Question --> FastAPI (JWT + role) --> Gemini (schema for this role only)
         --> validate_sql() --> read-only execution (row cap) --> JSON --> table + chart
                  |
                  +-- blocked queries are logged and rejected with HTTP 400
```

| Layer | File |
|---|---|
| API, auth routes, query flow | `app/main.py` |
| JWT, password hashing, RBAC | `app/auth.py` |
| LLM client (Gemini / mock) | `app/llm.py` |
| SQL safety validator | `app/sql_validator.py` |
| Models and role-to-table map | `app/models.py` |
| Synthetic demo data | `app/seed.py` |
| Web UI | `frontend/index.html` |

## Quick start

```bash
git clone https://github.com/<your-username>/text2sql-assistant.git
cd text2sql-assistant
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env        # add your GEMINI_API_KEY (or set LLM_PROVIDER=mock)
uvicorn app.main:app --reload
```

Open http://localhost:8000 for the UI or http://localhost:8000/docs for Swagger.

Demo accounts (synthetic, local only): `admin / admin123` and `analyst / analyst123`.

With Docker: `docker compose up --build`.

## Example questions

- Top 5 products by revenue
- How many orders were cancelled each month?
- Which category sells the most units?
- Average order value by status
- Customers per country (admin only)

## API

| Method | Path | Description |
|---|---|---|
| POST | `/auth/register` | Create a `USER` account |
| POST | `/auth/login` | Get a JWT |
| GET | `/schema` | Tables visible to your role |
| POST | `/query` | `{"question": "..."}` returns SQL, columns and rows |
| GET | `/history` | Recent queries |

## Security notes

LLM output is treated as untrusted input. The validator blocks `DROP`, `DELETE`, `UPDATE`, `INSERT`, multiple statements, comment tricks, schema-qualified names and tables outside the caller's role (see `tests/test_sql_validator.py`). No validator is perfect, so for a real deployment also:

- connect with a database user that only has `SELECT` on the business tables
- set a strong `JWT_SECRET` and remove the demo accounts
- add rate limiting and query timeouts
- keep your `.env` out of Git (it is in `.gitignore`)

## Tests

```bash
pytest -q
```

Covers the validator (injection and role cases) and the API (auth, blocked SQL, RBAC, history) using the mock LLM.

## Limitations and roadmap

- The model can still write a wrong (but safe) query; showing the SQL helps users spot this
- Add an evaluation set of question/SQL pairs to measure accuracy across prompts and models
- Few-shot examples and column descriptions to improve accuracy
- Next.js frontend and a deployed demo
- Conversation memory for follow-up questions ("now only for 2026")

## License

MIT
