# GUIA COPY/PASTE — implementação completa

Este arquivo contém o conteúdo completo dos arquivos alterados/novos.
A ideia é poder substituir o arquivo inteiro por cada bloco correspondente.

> Observação: o projeto completo já está nesta mesma pasta; você não precisa copiar
> manualmente se for usar o ZIP pronto.


## ETAPA 1 — Banco


### `backend/models.py`

```python
from __future__ import annotations

from datetime import datetime, timezone

from sqlalchemy import DateTime, ForeignKey, Integer, String, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship

from backend.database import Base


def utc_now_naive() -> datetime:
    return datetime.now(timezone.utc).replace(tzinfo=None)


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True, nullable=False)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=utc_now_naive)


class UserCustomInstructions(Base):
    """
    One row per user.

    This is intentionally a separate table instead of a new column on `users`.
    Base.metadata.create_all() creates missing tables, but does not ALTER an
    already-existing SQLite table to add a new column.
    """

    __tablename__ = "user_custom_instructions"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    user_id: Mapped[int] = mapped_column(
        Integer,
        ForeignKey("users.id"),
        unique=True,
        index=True,
        nullable=False,
    )
    content: Mapped[str] = mapped_column(Text, nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=utc_now_naive)
    updated_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=utc_now_naive,
        onupdate=utc_now_naive,
    )

    user = relationship("User", backref="custom_instructions_record")


class TokenBlacklist(Base):
    __tablename__ = "token_blacklist"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    jti: Mapped[str] = mapped_column(String(255), unique=True, index=True, nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=utc_now_naive)


class ChatSession(Base):
    __tablename__ = "chat_sessions"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    user_id: Mapped[int] = mapped_column(
        Integer,
        ForeignKey("users.id"),
        index=True,
        nullable=False,
    )
    title: Mapped[str] = mapped_column(String(255), default="Novo chat")
    created_at: Mapped[datetime] = mapped_column(DateTime, default=utc_now_naive)
    updated_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=utc_now_naive,
        onupdate=utc_now_naive,
    )

    user = relationship("User", backref="sessions")


class ChatMessage(Base):
    __tablename__ = "chat_messages"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    session_id: Mapped[int] = mapped_column(
        Integer,
        ForeignKey("chat_sessions.id"),
        index=True,
        nullable=False,
    )
    user_id: Mapped[int] = mapped_column(
        Integer,
        ForeignKey("users.id"),
        index=True,
        nullable=False,
    )
    role: Mapped[str] = mapped_column(String(20), index=True)
    content: Mapped[str] = mapped_column(Text)
    model: Mapped[str] = mapped_column(String(120), default="google/gemma-4-31b-it")
    created_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=utc_now_naive,
        index=True,
    )

    session = relationship("ChatSession", backref="messages")
```


### Git ao fim da etapa

```bash
git add backend/models.py
git commit -m "feat: add custom instructions persistence"
git push
```


## ETAPA 2 — Schemas, endpoint e registro do router


### `backend/schemas/custom_instructions.py`

```python
from __future__ import annotations

from pydantic import BaseModel, Field


class CustomInstructionsUpdate(BaseModel):
    content: str = Field(default="", max_length=12000)


class CustomInstructionsOut(BaseModel):
    content: str
    is_custom: bool
```


### `backend/routers/custom_instructions.py`

```python
from __future__ import annotations

from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from backend.auth import get_current_user
from backend.database import get_db
from backend.models import User, UserCustomInstructions
from backend.schemas.custom_instructions import (
    CustomInstructionsOut,
    CustomInstructionsUpdate,
)
from backend.services.openrouter import DEFAULT_SYSTEM_PROMPT

router = APIRouter(
    prefix="/api/custom-instructions",
    tags=["custom-instructions"],
    dependencies=[Depends(get_current_user)],
)


def _get_record(db: Session, user_id: int) -> UserCustomInstructions | None:
    return (
        db.query(UserCustomInstructions)
        .filter(UserCustomInstructions.user_id == user_id)
        .first()
    )


@router.get("", response_model=CustomInstructionsOut)
def get_custom_instructions(
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
):
    record = _get_record(db, current_user.id)

    if record and record.content.strip():
        return CustomInstructionsOut(content=record.content, is_custom=True)

    return CustomInstructionsOut(
        content=DEFAULT_SYSTEM_PROMPT,
        is_custom=False,
    )


@router.put("", response_model=CustomInstructionsOut)
def save_custom_instructions(
    payload: CustomInstructionsUpdate,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
):
    content = payload.content.strip()
    record = _get_record(db, current_user.id)

    # Empty content means "restore default".
    if not content:
        if record:
            db.delete(record)
            db.commit()

        return CustomInstructionsOut(
            content=DEFAULT_SYSTEM_PROMPT,
            is_custom=False,
        )

    if record:
        record.content = content
    else:
        record = UserCustomInstructions(
            user_id=current_user.id,
            content=content,
        )
        db.add(record)

    db.commit()
    db.refresh(record)

    return CustomInstructionsOut(content=record.content, is_custom=True)
```


### `backend/main.py`

```python
from __future__ import annotations

from pathlib import Path

from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import FileResponse, Response
from fastapi.staticfiles import StaticFiles
from starlette.middleware.base import BaseHTTPMiddleware

from backend.database import Base, engine
from backend.routers.auth import router as auth_router
from backend.routers.chat import router as chat_router
from backend.routers.custom_instructions import router as custom_instructions_router
from backend.routers.sessions import router as sessions_router


Base.metadata.create_all(bind=engine)

app = FastAPI(title="ChatLLM Experiment API")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=False,
    allow_methods=["*"],
    allow_headers=["*"],
)


class NoCacheMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response: Response = await call_next(request)
        response.headers["Cache-Control"] = "no-cache, no-store, must-revalidate"
        response.headers["Pragma"] = "no-cache"
        response.headers["Expires"] = "0"
        return response


app.add_middleware(NoCacheMiddleware)

app.include_router(auth_router)
app.include_router(chat_router)
app.include_router(sessions_router)
app.include_router(custom_instructions_router)

NO_CACHE_HEADERS = {
    "Cache-Control": "no-cache, no-store, must-revalidate",
    "Pragma": "no-cache",
    "Expires": "0",
}

ROOT_DIR = Path(__file__).resolve().parent.parent
FRONTEND_DIR = ROOT_DIR / "frontend"

if FRONTEND_DIR.exists():
    app.mount(
        "/frontend",
        StaticFiles(directory=FRONTEND_DIR),
        name="frontend",
    )


@app.get("/health")
def health_check() -> dict[str, str]:
    return {"status": "ok"}


@app.get("/")
def root() -> FileResponse:
    index_path = FRONTEND_DIR / "index.html"
    if not index_path.exists():
        raise HTTPException(
            status_code=404,
            detail="frontend/index.html nao encontrado",
        )
    return FileResponse(index_path, headers=NO_CACHE_HEADERS)
```


### Git ao fim da etapa

```bash
git add backend/schemas/custom_instructions.py backend/routers/custom_instructions.py backend/main.py
git commit -m "feat: add custom instructions API"
git push
```


## ETAPA 3 — OpenRouter


### `backend/services/openrouter.py`

```python
from __future__ import annotations

import json

import httpx

from backend.config import OPENROUTER_API_KEY, OPENROUTER_API_URL, OPENROUTER_MODEL_DEFAULT


class OpenRouterConfigError(RuntimeError):
    pass


_SYSTEM_PROMPT = (
    "Keep your answers short and concise. "
    "When writing mathematical expressions, use LaTeX notation: "
    r"\( ... \) for inline math and $$ ... $$ for display/block math. "
    "When writing currency values (e.g. dollar amounts), always escape the dollar sign as the HTML entity &#36; "
    "(e.g. write &#36;5.00 instead of $5.00) so it is never confused with a LaTeX delimiter."
)

DEFAULT_SYSTEM_PROMPT = _SYSTEM_PROMPT


def _resolve_system_prompt(system_prompt: str | None) -> str:
    if isinstance(system_prompt, str) and system_prompt.strip():
        return system_prompt.strip()
    return _SYSTEM_PROMPT


def _build_messages(
    *,
    user_message: str,
    history: list[dict],
    system_prompt: str | None = None,
) -> list[dict]:
    messages: list[dict] = [
        {"role": "system", "content": _resolve_system_prompt(system_prompt)}
    ]

    for item in history:
        role = item.get("role")
        content = item.get("content")
        if role in {"user", "assistant"} and isinstance(content, str) and content.strip():
            messages.append({"role": role, "content": content.strip()})

    messages.append({"role": "user", "content": user_message.strip()})
    return messages


def _build_headers() -> dict[str, str]:
    return {
        "Authorization": f"Bearer {OPENROUTER_API_KEY}",
        "Content-Type": "application/json",
        "HTTP-Referer": "http://localhost",
        "X-Title": "ChatLLM Experiment",
    }


async def generate_reply(
    *,
    user_message: str,
    history: list[dict],
    model: str | None = None,
    system_prompt: str | None = None,
) -> tuple[str, str]:
    if not OPENROUTER_API_KEY:
        raise OpenRouterConfigError(
            "OPENROUTER_API_KEY nao definido. Configure em .env ou environment variables."
        )

    resolved_model = model or OPENROUTER_MODEL_DEFAULT
    messages = _build_messages(
        user_message=user_message,
        history=history,
        system_prompt=system_prompt,
    )

    payload = {
        "model": resolved_model,
        "messages": messages,
    }

    async with httpx.AsyncClient(timeout=60.0) as client:
        response = await client.post(
            OPENROUTER_API_URL,
            json=payload,
            headers=_build_headers(),
        )

    if response.status_code >= 400:
        raise RuntimeError(
            f"OpenRouter retornou erro {response.status_code}: {response.text}"
        )

    data = response.json()
    content = data.get("choices", [{}])[0].get("message", {}).get("content", "")

    reply = content.strip()
    if not reply:
        raise RuntimeError("OpenRouter nao retornou conteudo de resposta.")

    return reply, resolved_model


async def stream_reply(
    *,
    user_message: str,
    history: list[dict],
    model: str | None = None,
    system_prompt: str | None = None,
):
    if not OPENROUTER_API_KEY:
        raise OpenRouterConfigError(
            "OPENROUTER_API_KEY nao definido. Configure em .env ou environment variables."
        )

    resolved_model = model or OPENROUTER_MODEL_DEFAULT
    payload = {
        "model": resolved_model,
        "messages": _build_messages(
            user_message=user_message,
            history=history,
            system_prompt=system_prompt,
        ),
        "stream": True,
    }

    async with httpx.AsyncClient(timeout=90.0) as client:
        async with client.stream(
            "POST",
            OPENROUTER_API_URL,
            json=payload,
            headers=_build_headers(),
        ) as response:
            if response.status_code >= 400:
                body = await response.aread()
                raise RuntimeError(
                    f"OpenRouter retornou erro {response.status_code}: "
                    f"{body.decode(errors='replace')}"
                )

            async for line in response.aiter_lines():
                if not line or not line.startswith("data:"):
                    continue

                data = line[len("data:") :].strip()
                if data == "[DONE]":
                    break

                try:
                    parsed = json.loads(data)
                except json.JSONDecodeError:
                    continue

                delta = (
                    parsed.get("choices", [{}])[0]
                    .get("delta", {})
                    .get("content")
                )
                if isinstance(delta, str) and delta:
                    yield delta
```


### Git ao fim da etapa

```bash
git add backend/services/openrouter.py
git commit -m "feat: support custom system prompt"
git push
```


## ETAPA 4 — Chat normal e streaming


### `backend/routers/chat.py`

```python
from __future__ import annotations

import json

from datetime import datetime, timezone

from fastapi import APIRouter, Depends, HTTPException
from fastapi.responses import StreamingResponse
from sqlalchemy.orm import Session

from backend.auth import get_current_user
from backend.config import OPENROUTER_MODEL_DEFAULT
from backend.database import get_db
from backend.models import (
    ChatMessage,
    ChatSession,
    User,
    UserCustomInstructions,
)
from backend.schemas.chat import ChatRequest, ChatResponse
from backend.services.openrouter import (
    OpenRouterConfigError,
    generate_reply,
    stream_reply,
)

router = APIRouter(dependencies=[Depends(get_current_user)])


def _ensure_session(
    session_id: int | None,
    current_user: User,
    db: Session,
) -> ChatSession:
    if session_id:
        session = (
            db.query(ChatSession)
            .filter(
                ChatSession.id == session_id,
                ChatSession.user_id == current_user.id,
            )
            .first()
        )
        if session:
            return session

    session = ChatSession(user_id=current_user.id, title="Novo chat")
    db.add(session)
    db.commit()
    db.refresh(session)
    return session


def _get_user_system_prompt(current_user: User, db: Session) -> str | None:
    record = (
        db.query(UserCustomInstructions)
        .filter(UserCustomInstructions.user_id == current_user.id)
        .first()
    )
    if record and record.content.strip():
        return record.content
    return None


async def _generate_title(
    user_message: str,
    db: Session,
    session: ChatSession,
) -> str | None:
    """
    Title generation deliberately uses the default system prompt.

    Custom instructions control the actual conversation. Keeping title
    generation separate prevents instructions such as "always answer in JSON"
    from producing unusable sidebar titles.
    """
    try:
        reply, _ = await generate_reply(
            user_message=(
                "Generate a very short title (max 6 words) for a chat that "
                "starts with this message. Return ONLY the title, no quotes "
                f'or extra text.\n\nMessage: "{user_message}"'
            ),
            history=[],
            model=None,
            system_prompt=None,
        )
        title = reply.strip().strip('"').strip("'").strip(".")[:60]
        if title:
            session.title = title
            session.updated_at = datetime.now(timezone.utc).replace(tzinfo=None)
            db.commit()
            return title
    except Exception:
        pass

    words = user_message.strip().split()
    fallback = " ".join(words[:6])
    if len(fallback) > 60:
        fallback = fallback[:60] + "..."
    if not fallback:
        fallback = "Novo chat"

    session.title = fallback
    session.updated_at = datetime.now(timezone.utc).replace(tzinfo=None)
    db.commit()
    return fallback


@router.post("/api/chat", response_model=ChatResponse)
async def chat(
    payload: ChatRequest,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
) -> ChatResponse:
    session = _ensure_session(payload.session_id, current_user, db)
    session_id = session.id
    current_user_id = current_user.id
    system_prompt = _get_user_system_prompt(current_user, db)

    try:
        reply, model_name = await generate_reply(
            user_message=payload.message,
            history=[item.model_dump() for item in payload.history],
            model=payload.model,
            system_prompt=system_prompt,
        )
    except OpenRouterConfigError as exc:
        raise HTTPException(status_code=503, detail=str(exc)) from exc
    except RuntimeError as exc:
        raise HTTPException(status_code=502, detail=str(exc)) from exc

    resolved_model = payload.model or model_name or OPENROUTER_MODEL_DEFAULT

    db.add(
        ChatMessage(
            session_id=session_id,
            user_id=current_user_id,
            role="user",
            content=payload.message,
            model=resolved_model,
        )
    )
    db.add(
        ChatMessage(
            session_id=session_id,
            user_id=current_user_id,
            role="assistant",
            content=reply,
            model=resolved_model,
        )
    )

    if session.title == "Novo chat" or not session.title:
        await _generate_title(payload.message, db, session)

    session.updated_at = datetime.now(timezone.utc).replace(tzinfo=None)
    db.commit()

    return ChatResponse(
        reply=reply,
        model=resolved_model,
        session_id=session_id,
        title=session.title,
    )


@router.post("/api/chat/stream")
async def chat_stream(
    payload: ChatRequest,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
) -> StreamingResponse:
    session = _ensure_session(payload.session_id, current_user, db)
    session_id = session.id
    current_user_id = current_user.id
    resolved_model = payload.model or OPENROUTER_MODEL_DEFAULT
    system_prompt = _get_user_system_prompt(current_user, db)

    is_first_message = session.title == "Novo chat" or not session.title

    db.add(
        ChatMessage(
            session_id=session_id,
            user_id=current_user_id,
            role="user",
            content=payload.message,
            model=resolved_model,
        )
    )

    if is_first_message:
        await _generate_title(payload.message, db, session)
        db.refresh(session)

    session.updated_at = datetime.now(timezone.utc).replace(tzinfo=None)
    db.commit()

    async def event_generator():
        full_reply = ""
        try:
            async for delta in stream_reply(
                user_message=payload.message,
                history=[item.model_dump() for item in payload.history],
                model=payload.model,
                system_prompt=system_prompt,
            ):
                full_reply += delta
                yield (
                    "data: "
                    + json.dumps({"delta": delta}, ensure_ascii=True)
                    + "\n\n"
                )
        except OpenRouterConfigError as exc:
            yield (
                "data: "
                + json.dumps({"error": str(exc)}, ensure_ascii=True)
                + "\n\n"
            )
            return
        except RuntimeError as exc:
            yield (
                "data: "
                + json.dumps({"error": str(exc)}, ensure_ascii=True)
                + "\n\n"
            )
            return

        if full_reply.strip():
            db.add(
                ChatMessage(
                    session_id=session_id,
                    user_id=current_user_id,
                    role="assistant",
                    content=full_reply,
                    model=resolved_model,
                )
            )
            db.commit()

        yield (
            "data: "
            + json.dumps(
                {
                    "done": True,
                    "session_id": session_id,
                    "title": session.title,
                },
                ensure_ascii=True,
            )
            + "\n\n"
        )

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
        },
    )
```


### Git ao fim da etapa

```bash
git add backend/routers/chat.py
git commit -m "feat: use user instructions in chat"
git push
```


## ETAPA 5 — API do frontend


### `frontend/src/api.js`

```javascript
const API_BASE = window.location.origin;

function getAuthHeaders() {
  const token = localStorage.getItem("access_token");
  if (!token) return {};
  return { Authorization: `Bearer ${token}` };
}

async function apiFetch(url, options = {}) {
  const response = await fetch(`${API_BASE}${url}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      ...getAuthHeaders(),
      ...options.headers,
    },
  });

  if (!response.ok) {
    const body = await response.json().catch(() => ({}));
    throw new Error(body.detail || `Erro ${response.status}`);
  }

  return response;
}

async function sendMessageStream({
  message,
  history,
  session_id,
  onDelta,
  signal,
}) {
  const response = await fetch(`${API_BASE}/api/chat/stream`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      ...getAuthHeaders(),
    },
    body: JSON.stringify({ message, history, session_id }),
    signal,
  });

  if (!response.ok) {
    const body = await response.json().catch(() => ({}));
    const detail =
      body?.detail || "Erro ao enviar mensagem para o servidor.";
    throw new Error(detail);
  }

  if (!response.body) {
    throw new Error("Streaming nao suportado no ambiente atual.");
  }

  const reader = response.body.getReader();
  const decoder = new TextDecoder("utf-8");
  let buffer = "";

  while (true) {
    const { value, done } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const events = buffer.split("\n\n");
    buffer = events.pop() || "";

    for (const rawEvent of events) {
      const line = rawEvent
        .split("\n")
        .find((part) => part.startsWith("data:"));

      if (!line) continue;

      const payloadText = line.slice(5).trim();
      if (!payloadText) continue;

      let payload;
      try {
        payload = JSON.parse(payloadText);
      } catch {
        continue;
      }

      if (payload.error) {
        throw new Error(payload.error);
      }

      if (payload.delta) {
        onDelta(payload.delta);
      }

      if (payload.done) {
        return {
          session_id: payload.session_id,
          title: payload.title,
        };
      }
    }
  }
}

async function listSessions() {
  const resp = await apiFetch("/api/sessions");
  const data = await resp.json();
  return data.sessions;
}

async function createSession() {
  const resp = await apiFetch("/api/sessions", {
    method: "POST",
    body: JSON.stringify({}),
  });
  return await resp.json();
}

async function getSessionMessages(sessionId) {
  const resp = await apiFetch(`/api/sessions/${sessionId}/messages`);
  return await resp.json();
}

async function deleteSession(sessionId) {
  await apiFetch(`/api/sessions/${sessionId}`, {
    method: "DELETE",
  });
}

async function getCustomInstructions() {
  const resp = await apiFetch("/api/custom-instructions");
  return await resp.json();
}

async function saveCustomInstructions(content) {
  const resp = await apiFetch("/api/custom-instructions", {
    method: "PUT",
    body: JSON.stringify({ content }),
  });
  return await resp.json();
}
```


### Git ao fim da etapa

```bash
git add frontend/src/api.js
git commit -m "feat: add custom instructions frontend API"
git push
```


## ETAPA 6 — Interface React e CSS


### `frontend/src/App.jsx`

```javascript
const {
  useEffect,
  useMemo,
  useRef,
  useState,
  useCallback,
} = React;

function createMessageId() {
  return `${Date.now()}-${Math.random()
    .toString(36)
    .slice(2, 10)}`;
}

function App() {
  const [token, setToken] = useState(
    localStorage.getItem("access_token")
  );
  const [userEmail, setUserEmail] = useState(
    localStorage.getItem("user_email") || ""
  );

  const [sessions, setSessions] = useState([]);
  const [activeSessionId, setActiveSessionId] = useState(null);
  const [messages, setMessages] = useState([]);
  const [text, setText] = useState("");
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState("");

  const [instructionsOpen, setInstructionsOpen] = useState(false);
  const [instructions, setInstructions] = useState("");
  const [instructionsAreCustom, setInstructionsAreCustom] =
    useState(false);
  const [instructionsLoading, setInstructionsLoading] =
    useState(false);
  const [instructionsSaving, setInstructionsSaving] =
    useState(false);
  const [instructionsError, setInstructionsError] =
    useState("");
  const [instructionsSaved, setInstructionsSaved] =
    useState(false);

  const messagesRef = useRef(null);
  const abortControllerRef = useRef(null);

  const chatHistory = useMemo(
    () =>
      messages.filter(
        (msg) =>
          msg.role === "user" || msg.role === "assistant"
      ),
    [messages]
  );

  useEffect(() => {
    const el = messagesRef.current;
    if (el) {
      el.scrollTop = el.scrollHeight;
    }
  }, [messages]);

  useEffect(() => {
    return () => {
      abortControllerRef.current?.abort();
    };
  }, []);

  useEffect(() => {
    if (!instructionsOpen) return;

    const onKeyDown = (event) => {
      if (event.key === "Escape" && !instructionsSaving) {
        setInstructionsOpen(false);
      }
    };

    window.addEventListener("keydown", onKeyDown);
    return () =>
      window.removeEventListener("keydown", onKeyDown);
  }, [instructionsOpen, instructionsSaving]);

  const loadSessions = useCallback(async () => {
    try {
      const sessionList = await listSessions();
      setSessions(sessionList);

      if (sessionList.length > 0) {
        const sid = sessionList[0].id;
        setActiveSessionId(sid);

        const msgs = await getSessionMessages(sid);
        if (msgs.length > 0) {
          setMessages(
            msgs.map((message) => ({
              id: message.id,
              role: message.role,
              content: message.content,
            }))
          );
        } else {
          setMessages([
            {
              id: createMessageId(),
              role: "assistant",
              content:
                "Bem-vindo ao ChatLLM Lab. Como posso ajudar voce hoje?",
            },
          ]);
        }
      } else {
        const newSession = await createSession();
        setSessions([newSession]);
        setActiveSessionId(newSession.id);
        setMessages([
          {
            id: createMessageId(),
            role: "assistant",
            content:
              "Bem-vindo ao ChatLLM Lab. Como posso ajudar voce hoje?",
          },
        ]);
      }
    } catch {
      try {
        const newSession = await createSession();
        setSessions([newSession]);
        setActiveSessionId(newSession.id);
      } catch {}
    }
  }, []);

  useEffect(() => {
    if (token) {
      loadSessions();
    }
  }, [token, loadSessions]);

  const selectSession = useCallback(async (sessionId) => {
    setActiveSessionId(sessionId);

    try {
      const msgs = await getSessionMessages(sessionId);

      if (msgs.length === 0) {
        setMessages([
          {
            id: createMessageId(),
            role: "assistant",
            content:
              "Bem-vindo ao ChatLLM Lab. Como posso ajudar voce hoje?",
          },
        ]);
      } else {
        setMessages(
          msgs.map((message) => ({
            id: message.id,
            role: message.role,
            content: message.content,
          }))
        );
      }
    } catch {
      setMessages([
        {
          id: createMessageId(),
          role: "assistant",
          content:
            "Bem-vindo ao ChatLLM Lab. Como posso ajudar voce hoje?",
        },
      ]);
    }
  }, []);

  const onAuthSuccess = async (newToken, email) => {
    setToken(newToken);
    setUserEmail(email);
  };

  const refreshSessions = useCallback(async () => {
    try {
      setSessions(await listSessions());
    } catch {}
  }, []);

  const handleNewSession = async () => {
    try {
      const newSession = await createSession();
      setSessions((prev) => [newSession, ...prev]);
      selectSession(newSession.id);
    } catch {
      setError("Erro ao criar sessao");
    }
  };

  const handleDeleteSession = async (sessionId) => {
    try {
      await deleteSession(sessionId);
      const newList = sessions.filter(
        (session) => session.id !== sessionId
      );
      setSessions(newList);

      if (activeSessionId === sessionId) {
        if (newList.length > 0) {
          selectSession(newList[0].id);
        } else {
          handleNewSession();
        }
      }
    } catch {
      setError("Erro ao excluir sessao");
    }
  };

  const handleLogout = async () => {
    try {
      await fetch("/api/auth/logout", {
        method: "POST",
        headers: {
          Authorization: `Bearer ${token}`,
        },
      });
    } catch {}

    localStorage.removeItem("access_token");
    localStorage.removeItem("user_email");

    setToken(null);
    setUserEmail("");
    setSessions([]);
    setActiveSessionId(null);
    setMessages([]);
    setError("");
  };

  const openInstructions = async () => {
    setInstructionsOpen(true);
    setInstructionsLoading(true);
    setInstructionsError("");
    setInstructionsSaved(false);

    try {
      const data = await getCustomInstructions();
      setInstructions(data.content || "");
      setInstructionsAreCustom(Boolean(data.is_custom));
    } catch (err) {
      setInstructionsError(
        err.message || "Nao foi possivel carregar as instrucoes."
      );
    } finally {
      setInstructionsLoading(false);
    }
  };

  const handleSaveInstructions = async () => {
    setInstructionsSaving(true);
    setInstructionsError("");
    setInstructionsSaved(false);

    try {
      const data = await saveCustomInstructions(instructions);
      setInstructions(data.content || "");
      setInstructionsAreCustom(Boolean(data.is_custom));
      setInstructionsSaved(true);
    } catch (err) {
      setInstructionsError(
        err.message || "Nao foi possivel salvar as instrucoes."
      );
    } finally {
      setInstructionsSaving(false);
    }
  };

  const handleRestoreDefault = async () => {
    setInstructionsSaving(true);
    setInstructionsError("");
    setInstructionsSaved(false);

    try {
      const data = await saveCustomInstructions("");
      setInstructions(data.content || "");
      setInstructionsAreCustom(false);
      setInstructionsSaved(true);
    } catch (err) {
      setInstructionsError(
        err.message || "Nao foi possivel restaurar o padrao."
      );
    } finally {
      setInstructionsSaving(false);
    }
  };

  const onStop = () => {
    abortControllerRef.current?.abort();
    abortControllerRef.current = null;
    setBusy(false);
  };

  const onSubmit = async (event) => {
    event.preventDefault();

    const cleaned = text.trim();
    if (!cleaned || busy) return;

    setError("");

    const userMessage = {
      id: createMessageId(),
      role: "user",
      content: cleaned,
    };
    const assistantMessageId = createMessageId();

    setMessages((prev) => [
      ...prev,
      userMessage,
      {
        id: assistantMessageId,
        role: "assistant",
        content: "",
      },
    ]);

    setText("");
    setBusy(true);

    const abortController = new AbortController();
    abortControllerRef.current = abortController;

    try {
      const result = await sendMessageStream({
        message: cleaned,
        history: chatHistory,
        session_id: activeSessionId,
        signal: abortController.signal,
        onDelta: (delta) => {
          setMessages((prev) =>
            prev.map((msg) =>
              msg.id === assistantMessageId
                ? {
                    ...msg,
                    content: `${msg.content}${delta}`,
                  }
                : msg
            )
          );
        },
      });

      if (result?.session_id) {
        setActiveSessionId(result.session_id);

        if (result.title) {
          setSessions((prev) =>
            prev.map((session) =>
              session.id === result.session_id
                ? {
                    ...session,
                    title: result.title,
                  }
                : session
            )
          );
        }
      }

      refreshSessions();

      setMessages((prev) =>
        prev.map((msg) =>
          msg.id === assistantMessageId &&
          !msg.content.trim()
            ? {
                ...msg,
                content:
                  "Nao foi possivel obter resposta do modelo agora.",
              }
            : msg
        )
      );
    } catch (err) {
      const aborted = err?.name === "AbortError";

      if (!aborted) {
        setError(
          err.message ||
            "Falha inesperada ao gerar resposta."
        );

        setMessages((prev) =>
          prev.map((msg) =>
            msg.id === assistantMessageId
              ? {
                  ...msg,
                  content: msg.content.trim()
                    ? msg.content
                    : "Nao foi possivel obter resposta do modelo agora.",
                }
              : msg
          )
        );
      } else {
        setMessages((prev) =>
          prev.map((msg) =>
            msg.id === assistantMessageId &&
            !msg.content.trim()
              ? {
                  ...msg,
                  content: "Resposta interrompida.",
                }
              : msg
          )
        );
      }
    } finally {
      refreshSessions();
      abortControllerRef.current = null;
      setBusy(false);
    }
  };

  if (!token) {
    return <Auth onAuthSuccess={onAuthSuccess} />;
  }

  return (
    <>
      <div className="app-layout">
        <Sidebar
          sessions={sessions}
          activeSessionId={activeSessionId}
          onSelectSession={selectSession}
          onNewSession={handleNewSession}
          onDeleteSession={handleDeleteSession}
        />

        <main className="app-shell">
          <header className="app-header">
            <div className="brand">ChatLLM Lab</div>

            <div className="header-right">
              <button
                className="secondary-btn"
                onClick={openInstructions}
              >
                Instrucoes
              </button>

              <span className="user-email">
                {userEmail}
              </span>

              <button
                className="secondary-btn"
                onClick={handleLogout}
              >
                Sair
              </button>
            </div>
          </header>

          <section
            className="messages"
            aria-live="polite"
            ref={messagesRef}
          >
            <div className="messages-inner">
              {messages.map((msg) => (
                <article
                  key={msg.id}
                  className={`bubble ${msg.role}`}
                >
                  <MessageContent content={msg.content} />
                </article>
              ))}
            </div>
          </section>

          <Composer
            text={text}
            busy={busy}
            error={error}
            onChangeText={setText}
            onSubmit={onSubmit}
            onStop={onStop}
          />
        </main>
      </div>

      {instructionsOpen && (
        <div
          className="modal-backdrop"
          onMouseDown={(event) => {
            if (
              event.target === event.currentTarget &&
              !instructionsSaving
            ) {
              setInstructionsOpen(false);
            }
          }}
        >
          <section
            className="modal-card"
            role="dialog"
            aria-modal="true"
            aria-labelledby="instructions-title"
          >
            <div className="modal-header">
              <div>
                <h2 id="instructions-title">
                  Instrucoes personalizadas
                </h2>
                <p>
                  Este texto sera enviado como system prompt
                  nas suas conversas.
                </p>
              </div>

              <button
                className="icon-btn"
                onClick={() =>
                  setInstructionsOpen(false)
                }
                disabled={instructionsSaving}
                aria-label="Fechar"
              >
                ×
              </button>
            </div>

            {instructionsLoading ? (
              <div className="modal-loading">
                Carregando...
              </div>
            ) : (
              <>
                <div className="instructions-status">
                  {instructionsAreCustom
                    ? "Usando instrucoes personalizadas"
                    : "Usando prompt padrao"}
                </div>

                <textarea
                  className="instructions-textarea"
                  value={instructions}
                  onChange={(event) => {
                    setInstructions(event.target.value);
                    setInstructionsSaved(false);
                  }}
                  maxLength={12000}
                  disabled={instructionsSaving}
                  autoFocus
                />

                <div className="modal-meta">
                  <span>
                    {instructions.length}/12000
                  </span>

                  {instructionsSaved && (
                    <span className="success-text">
                      Salvo.
                    </span>
                  )}

                  {instructionsError && (
                    <span className="error-text">
                      {instructionsError}
                    </span>
                  )}
                </div>

                <div className="modal-actions">
                  <button
                    className="danger-light-btn"
                    onClick={handleRestoreDefault}
                    disabled={instructionsSaving}
                  >
                    Restaurar padrao
                  </button>

                  <div className="modal-actions-right">
                    <button
                      className="secondary-btn"
                      onClick={() =>
                        setInstructionsOpen(false)
                      }
                      disabled={instructionsSaving}
                    >
                      Fechar
                    </button>

                    <button
                      className="primary-btn"
                      onClick={handleSaveInstructions}
                      disabled={instructionsSaving}
                    >
                      {instructionsSaving
                        ? "Salvando..."
                        : "Salvar"}
                    </button>
                  </div>
                </div>
              </>
            )}
          </section>
        </div>
      )}
    </>
  );
}

const root = ReactDOM.createRoot(
  document.getElementById("root")
);
root.render(<App />);
```


### `frontend/index.html`

```html
<!doctype html>
<html lang="pt-BR">
  <head>
    <meta charset="UTF-8" />
    <meta
      name="viewport"
      content="width=device-width, initial-scale=1.0"
    />
    <meta
      http-equiv="Cache-Control"
      content="no-cache, no-store, must-revalidate"
    />
    <title>ChatLLM Lab</title>

    <link
      rel="stylesheet"
      href="https://cdn.jsdelivr.net/npm/katex@0.16.11/dist/katex.min.css"
    />
    <link
      rel="stylesheet"
      href="https://cdn.jsdelivr.net/npm/highlight.js@11.10.0/styles/github-dark.min.css"
    />

    <style>
      :root {
        --bg-page: #ffffff;
        --text: #0d0d0d;
        --muted: #777;
        --accent: #1a1a1a;
        --accent-hover: #404040;
        --user-bubble: #f4f4f4;
        --border: #e5e5e5;
        --sidebar-bg: #f7f7f8;
        --sidebar-width: 260px;
      }

      * { box-sizing: border-box; }

      html, body, #root {
        width: 100%;
        height: 100%;
        margin: 0;
      }

      body {
        overflow: hidden;
        background: var(--bg-page);
        color: var(--text);
        font-family:
          ui-sans-serif, -apple-system, system-ui,
          "Segoe UI", Helvetica, Arial, sans-serif;
      }

      button, input, textarea {
        font: inherit;
      }

      button { cursor: pointer; }

      .app-layout {
        width: 100%;
        height: 100%;
        display: flex;
      }

      .sidebar {
        width: var(--sidebar-width);
        height: 100%;
        display: flex;
        flex-direction: column;
        flex-shrink: 0;
        overflow: hidden;
        background: var(--sidebar-bg);
        border-right: 1px solid var(--border);
      }

      .sidebar-header {
        padding: 12px;
        border-bottom: 1px solid var(--border);
      }

      .new-chat-btn {
        width: 100%;
        padding: 10px;
        border: 1px solid var(--border);
        border-radius: 10px;
        background: white;
      }

      .sidebar-list {
        flex: 1;
        overflow-y: auto;
        padding: 8px;
      }

      .sidebar-item {
        padding: 10px 12px;
        margin-bottom: 2px;
        border-radius: 8px;
        display: flex;
        align-items: center;
        gap: 8px;
        cursor: pointer;
      }

      .sidebar-item:hover,
      .sidebar-item.active {
        background: #e9e9ea;
      }

      .sidebar-item-title {
        flex: 1;
        overflow: hidden;
        white-space: nowrap;
        text-overflow: ellipsis;
        font-size: 0.88rem;
      }

      .sidebar-item-del {
        border: 0;
        background: transparent;
        color: var(--muted);
        font-size: 1.1rem;
      }

      .app-shell {
        flex: 1;
        min-width: 0;
        display: flex;
        flex-direction: column;
      }

      .app-header {
        padding: 12px 20px;
        min-height: 58px;
        border-bottom: 1px solid var(--border);
        display: flex;
        align-items: center;
        justify-content: space-between;
      }

      .brand {
        font-weight: 700;
      }

      .header-right {
        display: flex;
        align-items: center;
        gap: 10px;
      }

      .user-email {
        font-size: 0.82rem;
        color: var(--muted);
      }

      .secondary-btn,
      .primary-btn,
      .danger-light-btn,
      .icon-btn {
        border-radius: 8px;
        padding: 7px 12px;
      }

      .secondary-btn {
        background: white;
        border: 1px solid var(--border);
      }

      .secondary-btn:hover {
        background: #f5f5f5;
      }

      .primary-btn {
        background: var(--accent);
        color: white;
        border: 1px solid var(--accent);
        font-weight: 600;
      }

      .primary-btn:hover {
        background: var(--accent-hover);
      }

      .danger-light-btn {
        background: #fff5f5;
        color: #a52727;
        border: 1px solid #f0caca;
      }

      button:disabled {
        opacity: 0.55;
        cursor: not-allowed;
      }

      .messages {
        flex: 1;
        min-height: 0;
        overflow-y: auto;
        padding: 28px 16px 16px;
      }

      .messages-inner {
        max-width: 760px;
        margin: 0 auto;
      }

      .bubble {
        line-height: 1.65;
        overflow-wrap: anywhere;
        padding: 16px 0;
      }

      .bubble.user {
        margin-left: auto;
        width: fit-content;
        max-width: 78%;
        padding: 12px 18px;
        border-radius: 18px;
        background: var(--user-bubble);
      }

      .bubble.assistant {
        width: 100%;
      }

      .bubble pre {
        overflow-x: auto;
        padding: 14px;
        border-radius: 10px;
        background: #1f2937;
        color: white;
      }

      .bubble :not(pre) > code {
        padding: 2px 5px;
        border-radius: 4px;
        background: #f4f4f4;
      }

      .composer-wrap {
        padding: 8px 16px 20px;
      }

      .composer {
        max-width: 760px;
        margin: 0 auto;
        display: flex;
        align-items: center;
        gap: 8px;
        padding: 8px 10px 8px 18px;
        border: 1px solid #d9d9d9;
        border-radius: 28px;
        background: white;
        box-shadow: 0 2px 8px rgba(0,0,0,.06);
      }

      .composer input {
        flex: 1;
        border: 0;
        outline: 0;
        padding: 7px 0;
        font-size: 1rem;
      }

      .composer button {
        width: 36px;
        height: 36px;
        border: 0;
        border-radius: 50%;
        background: var(--accent);
        color: white;
      }

      .note {
        max-width: 760px;
        margin: 0 auto 8px;
        font-size: 0.86rem;
      }

      .error, .error-text {
        color: #b42318;
      }

      .success-text {
        color: #18794e;
      }

      .auth-shell {
        width: 100%;
        height: 100%;
        display: flex;
        align-items: center;
        justify-content: center;
        padding: 24px;
      }

      .auth-card {
        width: 100%;
        max-width: 400px;
        padding: 36px 30px;
        border: 1px solid var(--border);
        border-radius: 16px;
        box-shadow: 0 6px 30px rgba(0,0,0,.06);
      }

      .auth-brand,
      .auth-title {
        text-align: center;
      }

      .auth-title {
        color: var(--muted);
        font-size: 1rem;
        font-weight: 400;
      }

      .auth-form {
        display: flex;
        flex-direction: column;
        gap: 12px;
      }

      .auth-form input {
        padding: 12px 14px;
        border: 1px solid #d9d9d9;
        border-radius: 10px;
      }

      .auth-form button {
        padding: 12px;
        border: 0;
        border-radius: 10px;
        background: var(--accent);
        color: white;
        font-weight: 600;
      }

      .auth-error {
        color: #b42318;
        font-size: .86rem;
      }

      .auth-toggle {
        text-align: center;
        color: var(--muted);
        font-size: .88rem;
      }

      .modal-backdrop {
        position: fixed;
        inset: 0;
        z-index: 20;
        background: rgba(0,0,0,.42);
        display: flex;
        align-items: center;
        justify-content: center;
        padding: 20px;
      }

      .modal-card {
        width: min(760px, 100%);
        max-height: min(720px, 90vh);
        overflow: auto;
        background: white;
        border-radius: 16px;
        box-shadow: 0 20px 80px rgba(0,0,0,.24);
        padding: 22px;
      }

      .modal-header {
        display: flex;
        align-items: flex-start;
        justify-content: space-between;
        gap: 16px;
      }

      .modal-header h2 {
        margin: 0 0 4px;
        font-size: 1.15rem;
      }

      .modal-header p {
        margin: 0;
        color: var(--muted);
        font-size: .9rem;
      }

      .icon-btn {
        padding: 2px 8px;
        border: 0;
        background: transparent;
        font-size: 1.6rem;
      }

      .instructions-status {
        margin-top: 18px;
        color: var(--muted);
        font-size: .84rem;
      }

      .instructions-textarea {
        width: 100%;
        min-height: 300px;
        margin-top: 10px;
        padding: 14px;
        resize: vertical;
        border: 1px solid #d9d9d9;
        border-radius: 10px;
        line-height: 1.5;
        outline: 0;
      }

      .instructions-textarea:focus {
        border-color: #999;
      }

      .modal-meta {
        min-height: 24px;
        margin-top: 6px;
        display: flex;
        flex-wrap: wrap;
        gap: 12px;
        color: var(--muted);
        font-size: .82rem;
      }

      .modal-actions {
        margin-top: 14px;
        display: flex;
        align-items: center;
        justify-content: space-between;
        gap: 12px;
      }

      .modal-actions-right {
        display: flex;
        gap: 8px;
      }

      .modal-loading {
        padding: 50px 0;
        text-align: center;
        color: var(--muted);
      }

      @media (max-width: 720px) {
        .sidebar { display: none; }
        .user-email { display: none; }
        .bubble.user { max-width: 90%; }
        .modal-actions {
          align-items: stretch;
          flex-direction: column;
        }
        .modal-actions-right {
          justify-content: flex-end;
        }
      }
    </style>
  </head>

  <body>
    <div id="root"></div>

    <script src="https://unpkg.com/react@18/umd/react.development.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
    <script src="https://unpkg.com/@babel/standalone@7.22.20/babel.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/dompurify@3.1.7/dist/purify.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.10.0/highlight.min.js"></script>
    <script defer src="https://cdn.jsdelivr.net/npm/katex@0.16.11/dist/katex.min.js"></script>
    <script defer src="https://cdn.jsdelivr.net/npm/katex@0.16.11/dist/contrib/auto-render.min.js"></script>

    <script type="text/babel" src="/frontend/src/api.js"></script>
    <script type="text/babel" src="/frontend/src/Auth.jsx"></script>
    <script type="text/babel" src="/frontend/src/MessageContent.jsx"></script>
    <script type="text/babel" src="/frontend/src/Composer.jsx"></script>
    <script type="text/babel" src="/frontend/src/Sidebar.jsx"></script>
    <script type="text/babel" src="/frontend/src/App.jsx"></script>
  </body>
</html>
```


### Git ao fim da etapa

```bash
git add frontend/src/App.jsx frontend/index.html
git commit -m "feat: add custom instructions UI"
git push
```


## Testes finais

```bash
python -m compileall -q backend tests
python -m pytest -q tests/test_custom_instructions_unit.py
```


## Estado do repositório

```bash
git status
git log --oneline -8
```
