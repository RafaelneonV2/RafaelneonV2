# Sobre o Rafaelneon

> Soy Rafaelneon. Soy um manito que quero aprender a programar.
>
> Não é bio de landing page — é o mapa do que existe, para não perder tempo
> procurando o que já está pronto.

---

## Quem sou

- **Nome:** Rafaelneon
- **Idade/empresa:** projeto pessoal, sem equipe
- **Objetivo:** aprender a programar de verdade — não só copiar tutorial
- **Idioma de trabalho:** português (pt-Br), incluindo comentários de código
- **Estilo de trabalho:** devagar e verificável — build local, teste rodando, e só
  então commit. Nada de "deve funcionar".

---

## Sites principais

| Site | URL | O que é |
|------|-----|---------|
| Portal | https://www.rafaelneon.shop | Site institucional / vitrine |
| Streaming | https://watch.rafaelneon.shop | Player de vídeo (HLS) |
| Auth | https://auth.rafaelneon.shop | Autenticação centralizada |
| Exemplo | https://example.rafaelneon.shop | App de referência / playground |

Também no ar:

- https://neonbot.rafaelneon.shop — painel do bot de Discord
- https://backend.rafaelneon.shop — API (raiz em `/` responde 404 de propósito;
  os endpoints reais começam com `/api` ou `/health`)

**Verificado em 08/10/2026:** `www`, `watch`, `auth` e `neonbot` respondem 200.
`example.rafaelneon.shop` não resolve (000) — ainda não está publicado.

---

## NeonUI — o design system

O **NeonUI** é o recurso próprio mais reutilizado: um design system em
**preto, roxo e branco**, com fontes *Orbitron* (títulos) e *Rajdhani* (texto).

Ele vive em `apps/neonui/` e é a base visual de praticamente tudo.

### Como ele funciona

O `index.html` é a casca; a interface real é montada por `src/main.jsx`
(React) e compilada para `dist/`, que é **HTML + CSS + JS estático** — sem
servidor, sem SSR. É esse `dist/` que é servido no navegador e, também,
embutido dentro do APK.

### A ideia que guia o projeto

O APK Android é uma **casca burra**: `AndroidManifest` + `MainActivity` +
`WebView`, carregando `assets/public/` (o `dist/` do NeonUI). Trocar de
artefato Android para bundle web é o caminho padrão, não a exceção.

```
┌─────────────────────────────────────────┐
│  APK  (casca nativa)                    │
│  ├── AndroidManifest, MainActivity      │
│  ├── WebView                            │
│  └── assets/public/   ← o app real      │
│       ├── index.html                    │
│       └── assets/*.js  *.css            │
└─────────────────────────────────────────┘
```

O contrato completo está em `apps/neonui/CONCEITO.md`. Scripts de build:
`build-apk.sh`, `build-config.sh`, `make-template.sh`.

---

## Backend auth centralizado

Um backend único de autenticação, compartilhado por todos os apps, para que a
conta seja a mesma em qualquer lugar — um `session_id`, um cookie, uma sessão.

Duas implementações equivalentes, mantidas em paridade de API:

| Projeto | Stack | Papel |
|---------|-------|-------|
| `backend/` | Go + Fiber (porta **2100**) | Referência. É a implementação de verdade. |
| `backend-js/` | Node + Cloudflare Workers + D1 | Mesma API, rodando na borda. |

- Banco unificado em **SQLite** (`data/neonbackend.db`) na versão Go.
- Tabelas mapeadas: **Go 17** vs **JS 82** — o JS é a união de todos os apps
  que dividem a mesma backend.
- Camada `json_docs` em ambos: documentos JSON arbitrários guardados como
  linhas, no lugar de um arquivo por config. É o que permite levar tudo para o
  D1 sem reescrever os stores.
- Rotas públicas (OAuth, SSO) são registradas **antes** do grupo `mgmt` —
  no Fiber v2.52 o middleware de `Group("/api", …)` vaza para todas as rotas
  `/api/*` registradas depois. Isso já causou 401 em rota que deveria ser 200.

Comandos úteis:

```bash
# Go
GOPATH=/home/rafaelneon/go go build -o /tmp/neonbackend .
PORT=2100 /tmp/neonbackend

# JS
npm test
node src/cmd/docmigrate.mjs status|import|verify
npx wrangler deploy --dry-run
```

---

## Repositórios

Todos no owner **RafaelneonV2**, cada um com sua própria chave SSH de deploy
em `~/.ssh/` (alias no `~/.ssh/config`):

| Repositório | Remote | Chave |
|-------------|--------|-------|
| backend Go | `git@github-backend-go:RafaelneonV2/backend-go.git` | `id_ed25519_github_backend_go` |
| backend JS | `git@github-backend-js:RafaelneonV2/Backend-js.git` | `id_ed25519_github_backend_js` |

Deploy em produção roda por **Cloudflare Workers Builds**, conectado ao
GitHub pelo painel — não há workflow no repo, o push dispara sozinho.

---

## Mapa do filesystem

```
/home/rafaelneon/projects/
├── apps/            # frontends (React + Vite) e o NeonUI
├── backend/         # API em Go + Fiber  ← referência de paridade
├── backend-js/      # mesma API em Node/Workers + D1
├── data/            # banco SQLite unificado + dados
├── apks/            # artefatos Android
├── keystores/       # chaves de assinatura
└── cpa/, experiments/
```

Dentro de `apps/` vivem os apps: `neonbot`, `neonflix-v3`, `neonui`, `www`,
`home`, `dashboard`, `uptime`, `search`, `changelog`, `neon-auth`, `auth`.

---

## O que já existe (não refazer)

Antes de criar algo, checar se já não está pronto:

- **Login** — `authApi.login`, `googleLogin`, `discordLogin` em `src/api.js`
- **Session auth** — `session.id` em `localStorage`, header `Authorization`
- **Wrapper de API** — `src/lib/api-backend.js` (~100 exports), `src/lib/backend.js`
- **OAuth Discord** — `GET /api/discord/oauth/config` + `POST /api/discord/oauth/login`
  (o backend devolve só `client_id` e `redirect_uri`; a URL de authorize é montada
  no cliente, porque é pública e fixa)
- **Docstore** — `loadDoc`, `saveDoc`, `existsDoc`, `deleteDoc`, `importFile`, `importDir`
- **CLI de migração** — `docmigrate status|import|verify`, espelhando o `cmd/docmigrate` do Go

---

## Lição registrada

**Não assumir que algo não existe — verificar.**

Duas vezes isso custou uma decisão errada:

1. `git ls-remote` voltou vazio e eu concluí que o repo do GitHub não existia.
   Não: repo **vazio** é exit 0 sem refs; repo **inexistente** dá
   `Repository not found`. O `git.txt` afirmava que o repo não existia e
   estava errado.
2. Disse que havia ~25 call sites no JS lendo `../data/*.json`. Não havia
   nenhum — o JS nunca usou arquivos JSON; os stores já eram tabelas.

`sqlite3` não está instalado: usar `node:sqlite` para inspeção.
