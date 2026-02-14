# FastAPI Anchor

[![PyPI version](https://img.shields.io/pypi/v/fastapi-anchor.svg)](https://pypi.org/project/fastapi-anchor/)
[![Python 3.10+](https://img.shields.io/pypi/pyversions/fastapi-anchor.svg)](https://pypi.org/project/fastapi-anchor/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub](https://img.shields.io/github/stars/andreahlert/fastapi-anchor?style=social)](https://github.com/andreahlert/fastapi-anchor)

Session-based auth for FastAPI with HttpOnly cookies and optional PostgreSQL storage. Logout invalidates the session on the server so it stops working immediately.

Inspired by [Lucia](https://lucia-auth.com). Built on [fastapi-sessions](https://github.com/jordanisaacs/fastapi-sessions) by Jordan Isaacs.

---

## Features

| Feature | Description |
|--------|-------------|
| **Cookie-based** | Session ID in a signed HttpOnly cookie. No token in JS, better XSS story. |
| **Backend storage** | In-memory (dev) or PostgreSQL (prod). Session lives in your DB. |
| **Real logout** | Delete the session row; the next request with that cookie gets 401. |
| **Pluggable** | Same frontend/backend/verifier model as fastapi-sessions. Mix and match. |

## Installation

```bash
pip install fastapi-anchor
```

With PostgreSQL support:

```bash
pip install fastapi-anchor[postgres]
```

## Quick start (in-memory)

```python
from uuid import UUID, uuid4
from fastapi import Depends, FastAPI, HTTPException, Response
from fastapi_anchor.backends.implementations import InMemoryBackend
from fastapi_anchor.frontends.implementations import SessionCookie, CookieParameters
from fastapi_anchor.session_verifier import SessionVerifier
from pydantic import BaseModel

class SessionData(BaseModel):
    user_id: str

cookie_params = CookieParameters(max_age=14*24*3600, httponly=True, secure=False)
cookie = SessionCookie(
    cookie_name="session",
    identifier="auth",
    auto_error=True,
    secret_key="your-secret",
    cookie_params=cookie_params,
)
backend = InMemoryBackend[UUID, SessionData]()

class AuthVerifier(SessionVerifier[UUID, SessionData]):
    def __init__(self):
        self._identifier = "auth"
        self._backend = backend
        self._auto_error = True
        self._auth_http_exception = HTTPException(status_code=401, detail="Invalid session")

    @property
    def identifier(self): return self._identifier
    @property
    def backend(self): return self._backend
    @property
    def auto_error(self): return self._auto_error
    @property
    def auth_http_exception(self): return self._auth_http_exception
    def verify_session(self, model: SessionData): return True

verifier = AuthVerifier()
app = FastAPI()

@app.post("/login")
async def login(user_id: str, response: Response):
    session_id = uuid4()
    await backend.create(session_id, SessionData(user_id=user_id))
    cookie.attach_to_response(response, session_id)
    return {"ok": True}

@app.get("/me", dependencies=[Depends(cookie)])
async def me(session_data: SessionData = Depends(verifier)):
    return session_data

@app.post("/logout")
async def logout(response: Response, session_id: UUID = Depends(cookie)):
    await backend.delete(session_id)
    cookie.delete_from_response(response)
    return {"ok": True}
```

## PostgreSQL backend (real logout in production)

Create the table with `anchor_sessions_table`, then use `PostgresBackend`:

```python
from sqlalchemy import MetaData
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from fastapi_anchor.backends.implementations import (
    PostgresBackend,
    PostgresSessionData,
    anchor_sessions_table,
)

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
async_session_factory = async_sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)
metadata = MetaData()
sessions_table = anchor_sessions_table(metadata)
# Create tables once: await engine.run_sync(metadata.create_all)

backend = PostgresBackend(async_session_factory, table=sessions_table)
```

Session payload for Postgres: `user_id`, `expires_at`, `created_at` (use `PostgresSessionData` or your own Pydantic model with those fields). On logout, call `await backend.delete(session_id)` and clear the cookie; the session is removed from the DB.

## Links

- **PyPI:** [pypi.org/project/fastapi-anchor](https://pypi.org/project/fastapi-anchor/)
- **GitHub:** [github.com/andreahlert/fastapi-anchor](https://github.com/andreahlert/fastapi-anchor)

## License

MIT. Original [fastapi-sessions](https://github.com/jordanisaacs/fastapi-sessions) by Jordan Isaacs (MIT).
