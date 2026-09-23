# TODO

## Foundation
1. [ ] Initialize `package.json` with dependencies (`express`, `ws`, `better-sqlite3`, `passport`, `passport-saml`, `express-session`, `better-sqlite3-session-store`, `uuid`)
2. [ ] Create `db.js` — open SQLite connection, run schema migrations for `users`, `projects`, `project_members`, `files`, `sessions`
3. [ ] Add `data/` and `data/projects/` to `.gitignore`

## Server entrypoint
4. [ ] Create `server.js` — Express app, mount routes, attach WS server to HTTP server, start listening on port 3000

## Auth
5. [ ] Create `auth.js` — configure `passport-saml` strategy, `express-session` with SQLite store, upsert user on `saml_id`
6. [ ] Implement `GET /auth/saml` — redirect to IdP
7. [ ] Implement `POST /auth/saml/callback` — validate assertion, set session, redirect to `/`
8. [ ] Implement `GET /auth/logout` — destroy session, redirect to IdP SLO or `/`
9. [ ] Add session auth middleware used by all `/api` routes and WS upgrade

## REST API
10. [ ] Create `routes/api.js` — project CRUD (`GET/POST /projects`, `GET /projects/:pid`)
11. [ ] File CRUD (`GET/POST/PUT/DELETE /projects/:pid/files/:fid`)
12. [ ] Member management (`POST /projects/:pid/members`)
13. [ ] Create `routes/compile.js` — `POST /projects/:pid/compile`, `GET .../pdf`, `GET .../log`

## Compiler
14. [ ] Create `compiler/workspace.js` — write all project file rows to `data/projects/<pid>/`
15. [ ] Create `compiler/latexmk.js` — spawn `latexmk`, capture stdout/stderr to `out/compile.log`, enforce one-at-a-time per project (409 on concurrent)
16. [ ] Broadcast `compile_done` over WS on process exit

## WebSocket collaboration
17. [ ] Create `ws/hub.js` — handle WS upgrade, authenticate via session cookie, route to correct project
18. [ ] Create `ws/collab.js` — in-memory `Map<file_id, { rev, content, clients }>`, lazy load from SQLite, evict on zero clients
19. [ ] Handle `join` message — send `joined` with current rev + content
20. [ ] Handle `op` message — apply delta if `client_rev === server_rev`, persist, broadcast; reject with `stale` otherwise
21. [ ] Handle `cursor` message — fire-and-forget broadcast to other clients
22. [ ] Handle `compile` message — delegate to compiler pipeline

## Frontend
23. [ ] Create `public/index.html` — project list/dashboard, fetch from `/api/projects`
24. [ ] Create `public/editor.html` — editor shell layout (file tree, Monaco pane, PDF pane, compile button)
25. [ ] Create `public/editor.js` — load Monaco from CDN, open WS connection, send/receive ops, apply remote deltas to Monaco model, show remote cursors
26. [ ] Create `public/style.css` — minimal layout styles

## Containerization
27. [ ] Write `Dockerfile` — base on `texlive/texlive:latest`, install Node.js, copy app, set entrypoint
28. [ ] Write `docker-compose.yml` — expose port 3000, mount `data/` as named volume, pass SAML env vars
29. [ ] Document required environment variables (`SAML_ENTRY_POINT`, `SAML_CERT`, `SESSION_SECRET`, etc.) in README
