# OpenAI Agents SDK Code
This markdown file contains code relating to the OpenAI Agents SDK.

## 1. Postgres Session Implementation
```python
import json
from typing import Any

import asyncpg
from agents.memory import Session


class PostgreSQLSession(Session):
    """Custom PostgreSQL-based implementation of the Session protocol.

    https://openai.github.io/openai-agents-python/sessions/#custom-memory-implementations
    Stores each session’s items in two tables:
      - agent_sessions (session_id, created_at, updated_at)
      - agent_messages (id, session_id, message_data, created_at)
    """

    def __init__(self, connection_pool: asyncpg.Pool, session_id: str):
        """Initialize a PostgreSQLSession."""
        self._session_id = session_id
        self._pool = connection_pool

    async def add_items(self, items: list) -> None:
        """Store new items as JSONB payloads."""
        if not items:
            return

        async with self._pool.acquire() as conn, conn.transaction():
            await conn.execute(
                "INSERT INTO agent_sessions(session_id) VALUES($1) ON CONFLICT DO NOTHING",
                self._session_id
            )
            await conn.executemany(
                """
                INSERT INTO agent_messages(session_id, message_data)
                VALUES($1, $2::jsonb)
                """,
                [(self._session_id, json.dumps(item)) for item in items]
            )
            await conn.execute(
                "UPDATE agent_sessions SET updated_at = NOW() WHERE session_id = $1",
                self._session_id
            )

    async def get_items(self, limit: int | None = None) -> list:
        """Retrieve stored items, returning Python dicts."""
        sql = """
            SELECT message_data
              FROM agent_messages
             WHERE session_id = $1
             ORDER BY created_at ASC
        """
        args: list = [self._session_id]
        if limit is not None:
            sql += " LIMIT $2"
            args.append(limit)

        async with self._pool.acquire() as conn:
            rows = await conn.fetch(sql, *args)

        result: list = []
        for r in rows:
            v = r["message_data"]
            result.append(json.loads(v) if isinstance(v, str) else v)
        return result

    async def pop_item(self) -> Any | None:
        """Atomically delete and return the most recent item."""
        async with self._pool.acquire() as conn, conn.transaction():
            row = await conn.fetchrow(
                """
                DELETE FROM agent_messages
                 WHERE id = (
                   SELECT id
                     FROM agent_messages
                    WHERE session_id = $1
                    ORDER BY created_at DESC
                    LIMIT 1
                 )
                RETURNING message_data
                """,
                self._session_id
            )

        if not row:
            return None

        payload = row["message_data"]
        return json.loads(payload) if isinstance(payload, str) else payload

    async def clear_session(self) -> None:
        """Remove all items and the session row."""
        async with self._pool.acquire() as conn:
            await conn.execute(
                "DELETE FROM agent_sessions WHERE session_id = $1",
                self._session_id
            )
```