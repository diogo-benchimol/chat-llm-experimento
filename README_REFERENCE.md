# ChatLLM - implementação de referência

Esta cópia implementa **instruções personalizadas por usuário** sobre a base do
ChatLLM.

## O que foi implementado

- Persistência em SQLite por usuário (`user_custom_instructions`).
- `GET /api/custom-instructions`.
- `PUT /api/custom-instructions`.
- Autenticação obrigatória nos endpoints.
- Isolamento pelo `current_user.id`.
- Fallback para o prompt padrão quando não existe configuração personalizada.
- Uso do system prompt personalizado em `/api/chat`.
- Uso do system prompt personalizado em `/api/chat/stream`.
- Interface React para visualizar, editar, salvar e restaurar o padrão.
- Persistência após logout/login.

## Por que uma tabela nova?

A aplicação usa `Base.metadata.create_all(bind=engine)`. Isso cria tabelas que
ainda não existem, mas não adiciona uma coluna a uma tabela SQLite já existente.
Por isso a implementação usa uma tabela 1:1 separada, em vez de adicionar uma
coluna diretamente em `users`.

## Executar

### Windows

```bat
python -m venv .venv
.venv\Scripts\activate
pip install -r backend\requirements.txt
copy .env.example .env
python -m uvicorn backend.main:app --reload
```

### Linux / macOS

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r backend/requirements.txt
cp .env.example .env
python -m uvicorn backend.main:app --reload
```

Abra `http://127.0.0.1:8000`.

## Teste manual

1. Cadastre o usuário A.
2. Abra **Instruções**.
3. Salve: `Responda sempre começando com "DIÓGENES:"`.
4. Envie uma mensagem.
5. Faça logout e login novamente.
6. Confira que as instruções continuam salvas.
7. Crie outro usuário e confirme que ele não vê as instruções do usuário A.
8. Use **Restaurar padrão** para apagar a personalização.

## Arquivos centrais alterados

- `backend/models.py`
- `backend/schemas/custom_instructions.py`
- `backend/routers/custom_instructions.py`
- `backend/services/openrouter.py`
- `backend/routers/chat.py`
- `backend/main.py`
- `frontend/src/api.js`
- `frontend/src/App.jsx`
- `frontend/index.html`
