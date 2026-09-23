# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

underneedle is an open source online collaborative LaTeX editor and compiler. Users collaborate in real time on `.tex` and `.bib` files, compile to PDF server-side, and authenticate via SAML SSO.

## Stack

- **Backend:** Node.js, Express, `ws` (WebSockets)
- **Database:** SQLite via `better-sqlite3` (synchronous driver)
- **Frontend:** Vanilla JS, no bundler, no framework; Monaco editor from CDN
- **Auth:** `passport` + `passport-saml`; sessions stored in SQLite via `better-sqlite3-session-store`
- **Compiler:** `latexmk` (system TeX Live, inside Docker)
- **Container:** Docker Compose — one service built on a TeX Live base image; `data/` is a named volume

## Commands

```bash
npm install          # install deps
node server.js       # start server (dev)
docker compose up    # build image and start (production-like)
docker compose build # rebuild image after Dockerfile changes
```

There is no build step, transpiler, or test runner yet. Add commands here as they are introduced.

## Directory Structure

```
server.js              # Express + WS entrypoint, mounts all routes
db.js                  # better-sqlite3 setup and schema migrations
auth.js                # passport-saml config, session middleware
routes/
  api.js               # REST routes
  compile.js           # compile trigger and PDF/log streaming
ws/
  hub.js               # WS upgrade handler, session auth
  collab.js            # per-file in-memory state, broadcast logic
compiler/
  latexmk.js           # child_process.spawn wrapper for latexmk
  workspace.js         # writes project files to disk before compile
public/
  index.html           # project list / dashboard
  editor.html          # editor shell
  editor.js            # Monaco init + WS client
  style.css
data/                  # gitignored — runtime state
  db.sqlite
  projects/            # per-project compile workspaces
Dockerfile
docker-compose.yml
```

## Data Model (SQLite)

```sql
users          (id, saml_id UNIQUE, email, name, created)
projects       (id, name, owner_id, main_file, created, updated)
project_members(project_id, user_id, role)   -- role: owner|editor|viewer
files          (id, project_id, path, content, updated)  -- UNIQUE(project_id, path)
sessions       (sid, data, expires)
```

## REST API

All endpoints require a valid session. Base path `/api`.

```
GET    /projects
POST   /projects
GET    /projects/:pid
GET    /projects/:pid/files/:fid
POST   /projects/:pid/files
PUT    /projects/:pid/files/:fid
DELETE /projects/:pid/files/:fid
POST   /projects/:pid/compile         → { job_id }
GET    /projects/:pid/compile/:job/pdf
GET    /projects/:pid/compile/:job/log
POST   /projects/:pid/members

GET    /auth/saml           → redirect to IdP
POST   /auth/saml/callback  → upsert user, set session, redirect /
GET    /auth/logout
```

## WebSocket Protocol

One connection per editor session, scoped to one project. All messages are JSON.

Client → Server:
```
{ type: "join",    project_id, file_id, token }
{ type: "op",      file_id, rev, delta }   # delta: Monaco IModelContentChange[]
{ type: "cursor",  file_id, line, col }
{ type: "compile" }
```

Server → Client:
```
{ type: "joined",       rev, content }
{ type: "op",           from_user, rev, delta }
{ type: "cursor",       from_user, line, col }
{ type: "ack",          rev }
{ type: "compile_done", job_id, success }
{ type: "error",        message }
```

## Collaboration Model

Last-write-wins with server-side serialization. Each file has a `rev` counter. `ws/collab.js` holds an in-memory `Map<file_id, { rev, content, clients: Set }>` populated lazily, evicted when all clients disconnect.

On receiving an `op`:
- If `client_rev === server_rev`: apply delta, increment rev, persist to SQLite, broadcast to other clients.
- If `client_rev < server_rev`: reject with `{ type: "error", message: "stale" }`. Client re-fetches content via REST and resets Monaco's model.

Cursors are fire-and-forget; not persisted.

## Compilation Pipeline

1. Client sends `{ type: "compile" }` or `POST /api/projects/:pid/compile`.
2. `compiler/workspace.js` writes all project file rows to `data/projects/<pid>/`.
3. `compiler/latexmk.js` spawns:
   ```
   latexmk -pdf -interaction=nonstopmode -halt-on-error -outdir=data/projects/<pid>/out data/projects/<pid>/main.tex
   ```
4. On exit, server broadcasts `compile_done` to all project WS clients.
5. PDF and log are served by streaming the output files.
6. Only one compile per project at a time — a `Set<project_id>` of in-progress jobs enforces this; concurrent requests get 409.

## SAML Auth Flow

1. `/auth/saml` redirects to IdP.
2. IdP posts to `/auth/saml/callback`; `passport-saml` validates and parses.
3. `auth.js` upserts `users` on `saml_id = profile.nameID`, sets `req.session.userId`.
4. WS upgrade requests authenticate by reading the session cookie through the same session middleware.

## Containerization

The Dockerfile is based on a TeX Live image (e.g., `texlive/texlive:latest`) with Node.js installed on top. `data/` is mounted as a named Docker volume so the SQLite database and compiled PDFs survive container restarts.

`docker-compose.yml` exposes port 3000, sets environment variables for SAML config, and declares the `data` volume.

## Key npm Dependencies

| Package | Purpose |
|---|---|
| `express` | HTTP server |
| `ws` | WebSocket server |
| `better-sqlite3` | SQLite driver |
| `passport` + `passport-saml` | SAML SSO |
| `express-session` | Session middleware |
| `better-sqlite3-session-store` | SQLite session store |
| `uuid` | Job IDs |

Monaco is loaded from CDN — not bundled.
