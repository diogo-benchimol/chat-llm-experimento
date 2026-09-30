# Implementação completa — mapa de alterações

Esta é uma referência do que mudou no projeto.

## Etapa 1 — persistência

Arquivo: `backend/models.py`

Foi adicionada a classe `UserCustomInstructions`, com:
- `user_id` único e FK para `users.id`;
- `content` em `Text`;
- timestamps.

### Commit sugerido

```bash
git add backend/models.py
git commit -m "feat: add custom instructions persistence"
git push
```

## Etapa 2 — schemas e API

Novos arquivos:
- `backend/schemas/custom_instructions.py`
- `backend/routers/custom_instructions.py`

Endpoints:
- `GET /api/custom-instructions`
- `PUT /api/custom-instructions`

`PUT` com string vazia remove o registro e restaura o prompt padrão.

`backend/main.py` registra o novo router.

### Commit sugerido

```bash
git add backend/schemas/custom_instructions.py backend/routers/custom_instructions.py backend/main.py
git commit -m "feat: add custom instructions API"
git push
```

## Etapa 3 — OpenRouter

Arquivo: `backend/services/openrouter.py`

`_build_messages`, `generate_reply` e `stream_reply` agora aceitam:
`system_prompt: str | None = None`.

Sem valor personalizado, `_SYSTEM_PROMPT` continua sendo usado.

### Commit sugerido

```bash
git add backend/services/openrouter.py
git commit -m "feat: support custom system prompt"
git push
```

## Etapa 4 — chat normal e streaming

Arquivo: `backend/routers/chat.py`

A função `_get_user_system_prompt` busca as instruções pelo `current_user.id`.
O valor é passado tanto para `generate_reply` quanto para `stream_reply`.

### Commit sugerido

```bash
git add backend/routers/chat.py
git commit -m "feat: use user instructions in chat"
git push
```

## Etapa 5 — frontend API

Arquivo: `frontend/src/api.js`

Foram adicionadas:
- `getCustomInstructions()`
- `saveCustomInstructions(content)`

### Commit sugerido

```bash
git add frontend/src/api.js
git commit -m "feat: add custom instructions frontend API"
git push
```

## Etapa 6 — React

Arquivo: `frontend/src/App.jsx`

Foi adicionada uma interface modal para:
- carregar o valor;
- editar;
- salvar;
- restaurar o padrão;
- mostrar erro/status.

Arquivo `frontend/index.html` contém os estilos do modal.

### Commit sugerido

```bash
git add frontend/src/App.jsx frontend/index.html
git commit -m "feat: add custom instructions UI"
git push
```

## Commit final

```bash
git status
git add .
git commit -m "chore: finalize custom instructions feature"
git push
```

Se `git status` mostrar `nothing to commit`, não há necessidade do commit final.
