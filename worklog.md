# Worklog — IntentDraft (PromptPilot)

**Ultimo aggiornamento:** 2026-11-01  
**Scopo:** sequenza operativa per riprendere lo sviluppo full-stack da dove ci siamo fermati.

---

## Stato attuale (fatto)

| Area | Stato |
| ---- | ----- |
| **Frontend** | IntentDraft completo in mock; adapter API pronti; test Vitest + Playwright. Spec: [`docs/product/planning/frontend-implementation-spec.md`](docs/product/planning/frontend-implementation-spec.md) |
| **Backend B0** | Fastify, health, errori JSON, env zod, Prisma + Supabase connesso |
| **Backend B1** | 6 tabelle `idt_*` su Supabase, migrazione `init_idt_schema`, health `db: true` |
| **Backend B2** | Auth register/login/logout/me, sessioni cookie, test integrazione |
| **Backend B3** | CRUD conversazioni BE-C01…C05, test integrazione; FE hydrate + adapter API (IF-04) |
| **Backend B4** | POST messaggi stub (BE-M01), titolo auto, test integrazione; FE sync thread + load detail (IF-05 parziale) |
| **Backend B5** | Upload file BE-F01…F04, `@fastify/multipart`, FS `UPLOAD_DIR`; FE preview/download con cookie (M3) |
| **Backend B6** | Profilo/password (BE-U01/U02), prompt API (BE-P01/P02/P05), stub regenerate/skip; FE Settings + debounce prompt |
| **Backend B7** | Motore mock server (`mockEngine`), orchestrator, M4 UX API; Playwright `smoke-api.spec.ts` |
| **Backend B8** | Google OAuth routes + redirect errori; richiede credenziali Google in `.env` |
| **Backend B9** | Anthropic client, orchestrator LLM ANALYZE/GENERATE, fallback mock, test T-CONV/T-PROMPT |
| **Backend B10** | Rate limit auth/messaggi, CI GitHub Actions, test unit/integration, runbook segreti |
| **Dev admin** | `DEV_ADMIN_*` + seed all’avvio (solo development) |
| **Admin console** | Fasi A1–A10: `/api/admin/*`, console `/app/admin`, usage LLM, utenti, legal/cookie — [`admin-capabilities-plan.md`](docs/product/planning/admin-capabilities-plan.md) |
| **Schema DB** | Definito in doc + `backend/prisma/schema.prisma` — tabelle fisiche **`idt_*`** (8 caratteri). Vedi [`docs/product/architecture/database-design.md`](docs/product/architecture/database-design.md) §2.1 |
| **Commit recente** | Frontend checklist + test; backend scaffold (verificare `git log`) |

**Non ancora implementato sul backend:** deploy cloud (Blocco 11 / INF — rinviato).

---

## Prerequisiti sviluppatore

1. Node.js 20+
2. Progetto **Supabase** con `DATABASE_URL` in `backend/.env` (Session pooler consigliato se rete IPv4)
3. `backend/.env`: `PORT=3001`, `CORS_ORIGIN=http://localhost:5173`, `SESSION_SECRET` ≥ 32 caratteri
4. Per API reale FE: `frontend/.env` → `VITE_USE_MOCK=false`, `VITE_API_URL=http://localhost:3001`

**Avvio rapido:**

```bash
cd backend && npm install && npm run db:push && npm run dev
cd frontend && npm run dev
# oppure dalla root: npm install && npm run dev:all
```

---

## Prossimi passi (ordine obbligatorio)

Spunta ogni voce solo quando verificata (build ok + prova manuale o test indicato).

### Blocco 1 — Database su Supabase (B1)

1. [x] Eseguire `cd backend && npm run db:push` e confermare in Supabase **Table Editor** le 6 tabelle: `idt_user`, `idt_sess`, `idt_conv`, `idt_msgs`, `idt_file`, `idt_prmt`.
2. [x] Migrazione versionata `prisma/migrations/20260329220000_init_idt_schema` (baseline dopo `db:push` con `migrate resolve --applied`); flusso documentato in `backend/README.md`.
3. [x] Nessuna tabella legacy `schema_bootstrap` nel schema `public` (verificato 2026-10-29).
4. [x] Verificare `GET http://localhost:3001/api/health` → `{ "ok": true, "db": true }` (`pingDatabase` in `src/lib/db.ts`).
5. [ ] (Opzionale B1-003) Seed dev in `backend/prisma/seed.ts` — utente test; **non** committare password (placeholder: solo `user.count()` fino a B2).

**Riferimenti:** piano [`docs/product/planning/backend-implementation-plan.md`](docs/product/planning/backend-implementation-plan.md) Fase B1 · FOUND-012.

---

### Blocco 2 — Autenticazione (B2, FR-021)

6. [x] `src/services/passwordService.ts` (bcrypt) e `src/services/sessionService.ts` (crea/valida/elimina `idt_sess`).
7. [x] Route auth: `POST /api/auth/register`, `login`, `logout`, `GET /api/auth/me` — contratto §8.1.
8. [x] Cookie sessione httpOnly firmato (`SESSION_COOKIE_NAME`), TTL `SESSION_TTL_DAYS` (`src/app.ts`).
9. [x] Middleware `createRequireAuth` + `request.user` (`src/middleware/requireAuth.ts`); 401 → `UNAUTHORIZED`.
10. [x] Mapper `toUserDto` — allineato a `frontend/src/types/auth.ts`.
11. [x] Test integrazione Vitest: `tests/integration/auth.test.ts` (`npm run test`).
12. [ ] **Verifica FE:** `VITE_USE_MOCK=false`, register/login/logout, F5 su `/app` mantiene sessione (IF-01…03).

**Riferimenti:** [`docs/product/architecture/frontend-backend-operations.md`](docs/product/architecture/frontend-backend-operations.md) OP-A01…A04.

---

### Blocco 3 — Conversazioni CRUD (B3, FR-022)

13. [x] Modulo `src/modules/conversations/` — `GET/POST /api/conversations`, `GET/PATCH/DELETE /api/conversations/:id`.
14. [x] Scope rigoroso `user_id` da sessione; 404 se conversazione altrui.
15. [x] DTO allineati a `frontend/src/types/api.ts` (`ConversationSummaryDto`, `ConversationDetailDto`, `metadata` per pin/progetto).
16. [x] Titolo auto dal primo messaggio user (POST messaggi, max 48 caratteri).
17. [x] **Fix frontend:** `apiConversationAdapter` — `create()` async con id reale; `hydrateConversationsFromApi` al bootstrap auth (IF-04).
18. [ ] **Verifica FE:** sidebar lista, nuova chat, rename, pin, delete, refresh pagina (test manuali § sotto).

**Riferimenti:** piano backend §8 BE-C01…C05.

---

### Blocco 4 — Messaggi stub (B4, FOUND-021)

19. [x] `POST /api/conversations/:id/messages` — validazione lunghezza (`MAX_MESSAGE_CHARS`), INSERT `idt_msgs` user.
20. [x] Risposta stub assistant (testo fisso) + aggiornamento `idt_conv.updated_at`.
21. [ ] **Verifica FE:** invio messaggio in API mode (thread persistito dopo F5); accettare UX limitata fino a B7 (test manuali § sotto).

---

### Blocco 5 — Upload file (B5, FR-023) → **Milestone M3**

22. [x] `@fastify/multipart` — `POST /api/conversations/:id/files`, salvataggio su `UPLOAD_DIR`, INSERT `idt_file`.
23. [x] Allowlist MIME + `MAX_UPLOAD_BYTES`; path sicuro (UUID, no traversal).
24. [x] `GET .../files`, `DELETE /api/files/:id`, `GET /api/files/:id/download` (stream).
25. [ ] **Verifica FE:** upload, anteprima/download, delete; test T-FILE-01 (test manuali § sotto).

**DoD demo persistenza:** M3 — auth + chat + file su Supabase senza mock.

---

### Blocco 6 — Profilo, prompt API, settings (B6)

26. [x] `PATCH /api/users/me`, `POST /api/users/me/password`.
27. [x] `GET .../prompts/latest`, `PUT /api/prompts/:id`, feedback stub su `idt_prmt.feedback`.
28. [x] Endpoint `regenerate` / `skip-clarification` (stub in B6; logica completa in B7).
29. [ ] **Verifica FE:** pagina Settings in API mode (test manuali § sotto).

---

### Blocco 7 — Motore conversazione lato server (B7) → **Milestone M4**

30. [x] Portare logica da `frontend/src/lib/mockConversationEngine.ts` → `backend/src/services/orchestrator/mockEngine.ts`.
31. [x] `POST .../messages` usa mockEngine; risposta `SendMessageResponse` completa (phase, prompt; stato in `ai_state` + GET detail).
32. [x] Regenerate / skip-clarification; refusal medico/legal (FR-012).
33. [x] Contratto esteso: `interpretation`, `pendingQuestions`, `clarificationTurns` su GET conversation; adapter FE allineati.
34. [x] Playwright `e2e/smoke-api.spec.ts` (skip se backend non raggiungibile).
35. [ ] Test manuali §18 CHAT (a fine checklist).

**DoD:** stessa UX del mock, dati su Supabase.

---

### Blocco 8 — Google OAuth (B8, opzionale)

36. [x] `GET /api/auth/google`, callback, utente `authProvider: google`, `hasPassword: false` (richiede env Google).
37. [x] Redirect errori → `/#/login?oauth=error` (+ `reason` se non configurato).
38. [ ] Test AUTH-06, AUTH-07 (§18) — manuali dopo configurazione OAuth.

---

### Blocco 9 — LLM Anthropic (B9) → **Milestone M5 / beta Must**

39. [x] `LLM_ENABLED`, client Anthropic server-side only (NFR-001).
40. [x] Orchestrator ANALYZE / GENERATE — [`docs/product/ai/ai-prompt-engine-design.md`](docs/product/ai/ai-prompt-engine-design.md).
41. [x] Persistenza `idt_prmt` versionata; retry 429/503; no content in log (NFR-005).
42. [x] Fixture e test T-CONV, T-PROMPT in [`docs/product/quality/testing-and-evaluation.md`](docs/product/quality/testing-and-evaluation.md).

---

### Blocco 10 — Hardening & CI (B10)

43. [x] Rate limit su auth e messaggi (NFR-003).
44. [x] Suite integration backend con DB test (`test:integration`, Postgres in CI).
45. [x] CI: lint + test + build `frontend/` e `backend/` su PR (`.github/workflows/ci.yml`).
46. [x] Runbook rotazione segreti (`docs/ops/secrets-rotation-runbook.md`).

---

### Blocco 11 — Deploy cloud & utenti globali (INF, post-funzionale) -- evitalo per ora.. TO DO In the future

**Quando:** dopo **M3** (minimo) e idealmente insieme a **B10**; go-live pubblico solo con **M4–M5** se serve LLM in prod.  
**Guida unica:** [`docs/product/architecture/cloud-deployment-guide.md`](docs/product/architecture/cloud-deployment-guide.md) (provider, costi, env, GDPR, **multi-utente da reti globali** §9).

47. [ ] **INF-001** — Preparazione repo: `Dockerfile` backend, `.dockerignore`, `frontend/.env.production.example` (§12 Fase 0).
48. [ ] **INF-002** — Ambiente **staging** cloud: dominio, Supabase staging, API + volume upload, FE su CDN, smoke auth/cookie cross-subdomain (§12 Fase 1).
49. [ ] **INF-003** — **Produzione beta:** secret CI, monitoring su `/api/health`, privacy/OQ-012 (§12 Fase 2).
50. [ ] **INF-004** — **Accesso globale:** CDN edge per `frontend/dist`; API dietro proxy/WAF (es. Cloudflare); rate limit **per IP e per sessione** (NFR-003); download file via storage + CDN quando si scala (§9).
51. [ ] **INF-005** — Verifica isolamento **multi-utente** (già B3/B3-M06): ogni tenant logico = `user_id`; nessun dato cross-account sotto carico da reti diverse.
52. [ ] **INF-006** — Piano evolutivo multi-regione (solo se metriche latenza lo richiedono): object storage replicato, read replica DB, eventuale seconda region API — documentare decisione in guida §9.3.

**Nota:** multi-tenant **organizzazioni/team** (workspace condivisi) resta **fuori scope** beta (OQ-007); qui si intende **molti utenti singoli** da Internet mondiale.

---

## Milestone (riepilogo)

| ID | Descrizione | Blocchi |
| -- | ----------- | ------- |
| **M0** | Health + DB | 1 |
| **M1** | Auth | 2 |
| **M2** | Chat persistita (stub) | 3–4 |
| **M3** | + File | 5 |
| **M4** | UX mock su API | 7 |
| **M5** | Claude Sonnet | 9 |
| **M6** | Deploy cloud staging/prod + readiness globale | 10–11 |

---

## Documenti chiave (non duplicare qui)

| Documento | Uso |
| --------- | --- |
| [`worklog.md`](worklog.md) | **Questo file** — ordine esecuzione |
| [`docs/product/planning/backend-implementation-plan.md`](docs/product/planning/backend-implementation-plan.md) | Dettaglio task B0–B10, DoD, API checklist |
| [`docs/product/architecture/database-design.md`](docs/product/architecture/database-design.md) | Schema `idt_*` |
| [`docs/product/planning/frontend-implementation-spec.md`](docs/product/planning/frontend-implementation-spec.md) | Contratto API §8, test §18 |
| [`docs/product/planning/beta-roadmap.md`](docs/product/planning/beta-roadmap.md) | FOUND / BETA roadmap storica |
| [`backend/README.md`](backend/README.md) | Setup Supabase |
| [`frontend/README.md`](frontend/README.md) | Mock vs API, script test |
| [`docs/product/architecture/cloud-deployment-guide.md`](docs/product/architecture/cloud-deployment-guide.md) | Deploy localhost → cloud, costi, provider, utenti globali |
| [`docs/product/decisions/implementation-amendments.md`](docs/product/decisions/implementation-amendments.md) | Decisioni operative repo (Postgres, B0–B10, LLM, admin) |

---

## Dove riprendere **adesso**

**Prossima azione consigliata:** completare le **verifiche manuali in API mode** (checkbox § Blocchi 2–8) e la checklist §18 in [`frontend-implementation-spec.md`](docs/product/planning/frontend-implementation-spec.md); poi avviare la **fase test** (EV + rubric, § analisi gap in fondo a questo file). **Blocco 11 (INF deploy)** resta rinviato.

Dopo ogni blocco: `npm run build` in `backend/`, commit atomico, aggiornare le checkbox in questo file.

---

## Note per commit / review

- Non committare `backend/.env` né segreti Supabase.
- Dopo cambi schema: `npm run db:push` o migrate + rigenerare client (`prisma generate`).
- Allineare sempre API al §8 frontend spec; in caso di breaking change, aggiornare spec + adapter FE nello stesso PR.

---

## Test manuali — Blocco 3 (Conversazioni CRUD, B3)

Prerequisiti: backend su `:3001` con `db: true`, frontend con `VITE_USE_MOCK=false` e `VITE_API_URL=http://localhost:3001`, utente registrato e loggato su `/app`.

- **B3-M01 — Hydrate sidebar (IF-04):** Dopo login, la sidebar mostra le chat esistenti senza azioni extra. In Supabase Table Editor verifica righe in `idt_conv` per il tuo `user_id`; i titoli devono corrispondere. F5 su `/app`: stessa lista, cookie di sessione invariato.

- **B3-M02 — Nuova chat:** Clic «Nuova chat» crea una conversazione vuota e la seleziona. In DevTools → Network compare `POST /api/conversations` con `201` e un UUID in risposta. Un secondo click su «Nuova chat» non deve creare duplicati vuoti se esiste già una chat recente senza messaggi.

- **B3-M03 — Rinomina:** Dal menu della chat, rinomina (es. «Prova API»). Network: `PATCH /api/conversations/:id` con `{ title }` e `200`. Il titolo resta dopo F5 e in Supabase la colonna `title` di `idt_conv` è aggiornata.

- **B3-M04 — Pin e progetto:** Fissa una chat e spostane un’altra in «Progetto». Network: `PATCH` con `{ metadata: { pinned: true } }` o `{ inProject: true }`. Dopo F5 le sezioni Fissate/Progetti restano corrette; in DB `metadata` JSON su `idt_conv` riflette i flag.

- **B3-M05 — Elimina:** Elimina una chat confermando la modale. Network: `DELETE /api/conversations/:id` → `204`. La voce sparisce dalla sidebar; F5 non la ripresenta. La riga scompare da `idt_conv` (cascade su `idt_msgs` se presenti).

- **B3-M06 — Isolamento utente:** Con due account (A e B), crea una chat con A e copia l’UUID dall’URL o dalla risposta POST. Loggato come B, apri DevTools e `GET /api/conversations/<uuid-A>`: deve essere `404 NOT_FOUND`, non i dati di A.

- **B3-M07 — Dettaglio vuoto:** `GET /api/conversations/:id` (cookie valido) su chat senza messaggi: `200` con `messages: []`, `files: []`, `phase: "idle"`, `latestPrompt: null`. Coerente con contratto §8.3 per chat appena create.

- **B3-M08 — Build automatici:** Da repo root, `cd backend && npm run build && npm run test` e `cd frontend && npm run build` senza errori. Opzionale: verifica che i test integrazione `conversations.test.ts` siano verdi sul tuo `DATABASE_URL`.

---

## Test manuali — Blocco 4 (Messaggi stub, B4)

Prerequisiti come B3; chat vuota selezionata in API mode.

- **B4-M01 — Invio messaggio:** Scrivi un testo nel composer e invia. Network: `POST /api/conversations/:id/messages` → `201` con `userMessage` e `assistantMessage`. In UI compaiono il tuo messaggio e la risposta stub («Motore AI non ancora collegato…»). In Supabase `idt_msgs` ha due righe (user + assistant) per quella conversazione.

- **B4-M02 — Persistenza dopo F5:** Dopo l’invio, ricarica `/app`. Il thread (user + assistant) resta visibile nella chat attiva; la sidebar mostra titolo auto dal primo messaggio (primi ~48 caratteri). `GET /api/conversations/:id` restituisce `messages` con length ≥ 2.

- **B4-M03 — Titolo sidebar:** Su chat con titolo vuoto, il primo invio aggiorna il titolo in sidebar senza PATCH manuale. Verifica colonna `title` su `idt_conv` allineata al testo troncato.

- **B4-M04 — Validazione:** Prova invio solo spazi (bloccato lato FE) o contenuto oltre `MAX_MESSAGE_CHARS` (blocco FE; opzionale POST diretto → `400 VALIDATION_ERROR`). Allegati nel composer in API mode: toast tipo file fino a B5 (upload non implementato).

- **B4-M05 — UX limitata (atteso):** Dopo la risposta stub, `phase` è `idle` e il composer può non essere più visibile con messaggi presenti — comportamento accettabile fino a B7 (motore mock server). Rigenera/skip/chiarimenti restano non implementati o falliscono fino a B6/B7.

- **B4-M06 — Test automatici:** `backend/tests/integration/messages.test.ts` verde con `DATABASE_URL`; build FE/BE ok.

---

## Test manuali — Blocco 5 (Upload file, B5 / M3)

Prerequisiti come B3; chat con almeno un messaggio user (per allegati al messaggio) o composer (invio con file).

- **B5-M01 — Upload su messaggio:** Aggiungi un `.txt` o `.pdf` ≤ 5 MB a un messaggio utente esistente. Network: `POST /api/conversations/:id/files` multipart campo `file` → `201` `{ file }`. Supabase `idt_file` popolata; file su disco in `backend/uploads/<userId>/`.

- **B5-M02 — Upload con invio:** Allega file nel composer e invia messaggio. Prima uno o più POST file, poi POST messages con `attachments: [uuid…]`; gli allegati compaiono sul messaggio user dopo refresh.

- **B5-M03 — Anteprima e download:** Apri anteprima documento dalla sidebar o dal messaggio. Il pannello carica via `GET /api/files/:id/download` con cookie (fetch credenziali). Download salva il file con nome originale.

- **B5-M04 — Elimina allegato:** Rimuovi file da messaggio. `DELETE /api/files/:id` → `204`; scompare da UI e da DB; file rimosso da `UPLOAD_DIR`.

- **B5-M05 — Tipi e dimensioni:** File non in allowlist → toast tipo; file > 5 MB → toast dimensione. POST diretto con MIME non ammesso → `400 VALIDATION_ERROR`.

- **B5-M06 — Persistenza F5:** Dopo upload, F5 su `/app`: allegati ancora listati su chat/messaggio (`GET /api/conversations/:id` include `files`).

- **B5-M07 — Test automatico:** `backend/tests/integration/files.test.ts` (T-FILE-01) verde; build FE/BE ok.

---

## Test manuali — Blocco 6 (Settings & prompt API, B6)

Prerequisiti come B3; account locale con password (`hasPassword: true`).

- **B6-M01 — Profilo:** In `/app/settings` cambia display name e salva. Network: `PATCH /api/users/me` → `200` `{ user }`. Nome aggiornato in header/UI; Supabase `idt_user.display_name` allineato.

- **B6-M02 — Lingua:** Toggle IT/EN in preferenze. `PATCH` con `{ locale }`; i18n cambia subito; dopo F5 la lingua resta quella scelta (`idt_user.locale`).

- **B6-M03 — Password:** Cambia password (corrente + nuova + conferma). `POST /api/users/me/password` → `204`. Logout e login con nuova password ok; vecchia rifiutata.

- **B6-M04 — Account Google/mock:** Account senza password mostra messaggio «no password»; form cambio password nascosto (comportamento atteso).

- **B6-M05 — Prompt API (se esiste `idt_prmt`):** Con prompt in DB per una chat, modifica testo risultato: dopo ~600 ms debounce, `PUT /api/prompts/:id`. Feedback utile/non utile → `POST .../feedback` → `204` e colonna `feedback` in `idt_prmt`.

- **B6-M06 — Rigenera:** Con chat in `prompt_ready`, Rigenera → `POST .../regenerate` → `200` e nuovo testo prompt (variante mock).

- **B6-M07 — Test automatici:** `users.test.ts` e `prompts.test.ts` verdi; build FE/BE ok.

---

## Test manuali — Blocco 7 (Motore mock API, B7 / M4)

Prerequisiti: stack API mode; utente loggato.

- **B7-M01 — Flusso appunti:** Messaggio «organizza appunti riunione» → prompt copiabile (`prompt_ready`), textarea piena, fase coerente dopo F5 (`ai_state` + `idt_prmt`).

- **B7-M02 — Chiarimenti LinkedIn:** Messaggio vago tipo «post LinkedIn» → risposta assistant con domande, fase `clarifying`; risposta o Salta → prompt finale.

- **B7-M03 — Refusal:** Testo con tema medico/legal (es. «diagnosi medica») → messaggio assistant di rifiuto, nessun prompt.

- **B7-M04 — Rigenera / Salta:** Rigenera cambia il markdown; Salta in chiarimenti produce prompt con assunzione aggiuntiva.

- **B7-M05 — E2E opzionale:** Con backend attivo e FE `VITE_USE_MOCK=false`, `npx playwright test smoke-api.spec.ts --project=smoke-api`.

---

## Test manuali — Blocco 10 (Hardening & CI, B10)

- **B10-M01 — Rate limit auth:** Da stesso IP, >30 login/register in 15 min → `429` / `RATE_LIMITED` (JSON coerente con API).

- **B10-M02 — Rate limit messaggi:** Burst oltre soglia configurata su `POST .../messages` → `429`.

- **B10-M03 — CI locale:** `cd backend && npm run ci`; root `npm run ci` (backend + build/test frontend).

- **B10-M04 — GitHub Actions:** Push/PR su `main` → workflow **CI** verde (unit + integration con Postgres service).

- **B10-M05 — Runbook:** Leggere [`docs/ops/secrets-rotation-runbook.md`](docs/ops/secrets-rotation-runbook.md) e verificare che i secret non siano in git.

---

## Test manuali — Blocco 9 (LLM Anthropic, B9 / M5)

Prerequisiti: API mode; `LLM_ENABLED=true` e chiave Anthropic valida in `backend/.env`; riavvio API.

- **B9-M01 — Mock fallback:** Con `LLM_ENABLED=false`, invio messaggio → stesso comportamento M4 (mock), nessuna chiamata esterna.

- **B9-M02 — ANALYZE semplice:** Messaggio chiaro (es. appunti riunione) → prompt generato senza domande intermedie (T-CONV-01).

- **B9-M03 — Chiarimenti:** Richiesta vaga marketing → almeno una domanda assistant, poi risposta → prompt (T-CONV-02).

- **B9-M04 — Refusal:** Tema medico/legal → rifiuto, nessun `idt_prmt` (T-SEC-01).

- **B9-M05 — Rigenera / Salta:** Con LLM attivo, rigenera e salta producono nuova versione in `idt_prmt`.

- **B9-M06 — Errori provider:** Chiave invalida o rate limit → UI errore / 503 `LLM_UNAVAILABLE` (T-ERR-01).

- **B9-M07 — Test automatici:** `cd backend && npm run test` (unit `llm/*` + integrazione esistenti).

---

## Test manuali — Blocco 8 (Google OAuth, B8)

**Prerequisito configurazione (tua azione):** Google Cloud Console → OAuth client Web; redirect URI `http://localhost:3001/api/auth/google/callback`; in `backend/.env` e `frontend/.env` imposta `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `FRONTEND_URL`, `VITE_GOOGLE_AUTH_START_URL`.

- **B8-M01 — Non configurato:** Senza env Google, «Continua con Google» → redirect login con `oauth=error` (reason `not_configured`).

- **B8-M02 — Login Google (AUTH-06):** Con credenziali ok, login Google → sessione cookie → `/app`, `authProvider: google`, `hasPassword: false` in Settings.

- **B8-M03 — Errore OAuth (AUTH-07):** Simula callback invalido o annulla su Google → redirect `/#/login?oauth=error`.

---

















## Analisi gap — verso la primissima versione testabile (2026-11-01)

Questa sezione sintetizza **tutto ciò che manca** (software, verifica, qualità, documentazione) per passare da “codice implementato” a **prodotto locale funzionante e pronto alla fase di test** (pilota interno / 5–10 utenti), incrociando questo worklog, [`docs/product/requirements/product-requirements.md`](docs/product/requirements/product-requirements.md), [`docs/product/planning/beta-roadmap.md`](docs/product/planning/beta-roadmap.md), [`docs/product/quality/testing-and-evaluation.md`](docs/product/quality/testing-and-evaluation.md) e [`docs/product/planning/frontend-implementation-spec.md`](docs/product/planning/frontend-implementation-spec.md) §18.

**Nota di lettura:** gran parte dei blocchi B0–B10 e admin A1–A10 è **già implementata in repo**; le checkbox ancora `[ ]` nel worklog indicano soprattutto **prove manuali non ancora eseguite o non documentate**, non necessariamente lavoro di coding mancante. La [`beta-roadmap.md`](docs/product/planning/beta-roadmap.md) e parti del [`backend-implementation-plan.md`](docs/product/planning/backend-implementation-plan.md) restano **indietro rispetto al codice** (MySQL vs Supabase, FOUND/BETA non spuntati) — vedi §F sotto.

### Sintesi esecutiva

| Area | Stato codice | Cosa manca davvero |
| ---- | ------------ | ------------------- |
| **Frontend UI** | Completo (mock + adapter API, admin, bulk delete chat, screenshot in docs) | Passare checklist §18; responsive/a11y spot-check |
| **Backend API** | Auth, CRUD, file, orchestrator mock/LLM, admin, rate limit, CI | Verifiche manuali + LLM con chiave reale; admin §8 manuale |
| **Test automatici** | Unit/integration BE, smoke Playwright mock + `smoke-api` opzionale | Gate **G1**: T-E2E-01…05 non tutti in CI; cartella **`tests/fixtures/eval/` assente** |
| **Deploy cloud (M6 / INF)** | Non implementato | **Fuori scope** per la primissima versione test locale/team |
| **Routing multi-LLM (R2–R5)** | Solo Claude + mock | **Fuori scope** pre-pilota; documentato in [`llm-routing-strategy.md`](docs/product/ai/llm-routing-strategy.md) |

**Definition of Done “pronta fase test” (locale):** stack avviabile con PM2 o `dev:all`, `VITE_USE_MOCK=false`, Supabase collegato, tutte le verifiche manuali B3–B9 e §18 FE spuntate, `npm run ci` verde, almeno una passata B9 con `LLM_ENABLED=true`, smoke-api Playwright eseguito almeno una volta, gate **G4** (no secret nel bundle) e **G5** (pagina Limiti/privacy coerente con API reale) verificati, dataset EV-01/EV-02 pronti per rubric umana.

---

### A. Implementazione software — già presente (non rifare)

Riferimento rapido per non duplicare lavoro:

- **Auth:** register/login/logout/me, cookie `idt_sess`, forgot/reset password (route + pagine FE).
- **Conversazioni:** CRUD, metadata pin/progetto, titolo auto, isolamento `user_id`.
- **Messaggi + AI:** orchestrator server (`mockEngine` + `llmEngine` con ANALYZE/GENERATE), regenerate/skip, refusal medico/legal, `ai_state`, `idt_prmt` versionati.
- **File:** multipart upload, download con cookie, allowlist MIME, test integrazione.
- **Profilo:** PATCH utente, cambio password, Settings FE.
- **Google OAuth:** route BE + UI (richiede env Google per test reali).
- **Admin:** `/api/admin/*`, console `/app/admin`, usage LLM (`idt_usag`), template prompt piattaforma, audit, legal/cookie API.
- **Hardening:** rate limit auth/messaggi, GitHub Actions CI, runbook rotazione segreti.
- **Frontend:** i18n IT/EN, tema chiaro/scuro, HashRouter, anteprima PDF, selezione multipla ed eliminazione bulk chat.

---

### B. Blocco critico — verifiche manuali end-to-end (API mode)

Prerequisiti comuni per ogni sessione di test:

1. `backend/.env`: `DATABASE_URL`, `SESSION_SECRET`, `CORS_ORIGIN=http://localhost:5173`; opzionale `LLM_ENABLED` + `ANTHROPIC_API_KEY`.
2. `frontend/.env`: `VITE_USE_MOCK=false`, `VITE_API_URL=http://localhost:3001`.
3. Migrazioni admin applicate se DB vecchio: `cd backend && npx prisma migrate deploy` (vedi `backend/README.md`).
4. Avvio: `npm run pm2:start` dalla root **oppure** `npm run dev:all` — **non** mescolare porte occupate. Se PM2 è vuoto (`pm2 list` senza processi), usare **`npm run pm2:start`** prima di `pm2 restart intentdraft`.

#### B.1 Checkbox worklog ancora aperte (priorità P0)

| ID worklog | Attività | Riferimento dettaglio |
| ---------- | -------- | --------------------- |
| 12 | Verifica FE auth IF-01…03 | Registrazione, login, logout, F5 su `/app` |
| 18 | Verifica FE conversazioni | B3-M01 … B3-M08 (§ sopra) |
| 21 | Verifica FE messaggi | B4-M01 … B4-M06 — oggi atteso flusso **M4/M5**, non stub B4 |
| 25 | Verifica FE file | B5-M01 … B5-M07 |
| 29 | Verifica FE Settings | B6-M01 … B6-M07 |
| 35 | Test manuali chat M4 | B7-M01 … B7-M05 + `smoke-api.spec.ts` |
| 38 | Google OAuth | B8-M01 … B8-M03 (AUTH-06/07) — richiede Google Cloud Console |

#### B.2 Checklist §18 frontend (non tracciata nel worklog — P0)

In [`frontend-implementation-spec.md`](docs/product/planning/frontend-implementation-spec.md) §18 restano **tutte** le voci PRE, PUB, AUTH, SET, SH, CHAT, FILE, A11Y, I18N, VIS, BE non spuntate. Vanno eseguite **due volte** se possibile:

- una passata con **`VITE_USE_MOCK=true`** (regression UI rapida);
- una passata con **`VITE_USE_MOCK=false`** (BE-01 … BE-05).

Ordine suggerito: **PRE → PUB → AUTH → SET → SH → CHAT → FILE → BE** (API mode), poi A11Y/I18N/VIS.

#### B.3 Admin console — verifica manuale (P1)

[`admin-capabilities-plan.md`](docs/product/planning/admin-capabilities-plan.md) §13: unica voce ancora `[ ]` — **Verifica manuale con account §8** (login `DEV_ADMIN_*`, navigazione tab Dashboard/Utenti/Uso API/Prompt/Piattaforma, export usage se presente, nessun accesso admin con utente `user`).

#### B.4 LLM Anthropic in condizioni reali (P0 per pilota “Must”)

Con `LLM_ENABLED=true` e chiave valida, eseguire **B9-M01 … B9-M07** (§ sopra). Senza questa passata, il pilota rischia di testare solo il mock server-side.

#### B.5 Hardening — conferma operativa (P1)

B10-M01 … B10-M05: rate limit e CI GitHub — verificare almeno una volta su branch/PR e in locale `npm run ci`.

---

### C. Test automatici e gap rispetto ai gate qualità

Fonte: [`testing-and-evaluation.md`](docs/product/quality/testing-and-evaluation.md).

#### C.1 Gate G1 — E2E Must in CI (gap principale)

| Test | Stato attuale | Azione |
| ---- | ------------- | ------ |
| **T-E2E-01** | Coperto parzialmente da smoke mock / smoke-api | Estendere Playwright: interpretazione visibile dopo invio |
| **T-E2E-02** | Non dedicato | Scenario LinkedIn / 2 turni chiarimenti in `e2e/` |
| **T-E2E-03** | **`smoke-api.spec.ts`** copia clipboard | Aggiungere a CI con servizio Postgres + backend (oggi spesso skip se BE down) |
| **T-E2E-04** | Non automatizzato | Test rigenera prompt |
| **T-E2E-05** | Non automatizzato | Nuova conversazione resetta thread |

File E2E esistenti: `frontend/e2e/smoke.spec.ts`, `smoke-api.spec.ts`, `responsive.spec.ts`, `admin.spec.ts`. **Manca** un job CI che avvii backend + DB e lanci `smoke-api` in modo deterministico (oggi CI root fa build FE + test unit, non full stack E2E API).

#### C.2 Dataset valutazione EV-* (P0 per fase test qualità, non per “funziona”)

- Cartella **`tests/fixtures/eval/`** (o `backend/tests/fixtures/eval/`) **non esiste** — roadmap BETA-083.
- Creare almeno **EV-01** (appunti) e **EV-02** (post LinkedIn) come JSON/markdown con input utente + criteri rubric attesi.
- EV-03 … EV-15 possono seguire durante il pilota.

#### C.3 Gate G2, G3 — valutazione umana (fase test vera)

- **G2:** ≥80% casi EV pass rubric (§5 testing doc) — richiede **2 revisori** e sessioni dedicate, non solo sviluppo.
- **G3:** confronto baseline vs prompt IntentDraft (§6) — stesso modello downstream, ordine randomizzato.
- **BETA-084 / BETA-085** in beta-roadmap: ancora da pianificare.

#### C.4 Test T-ERR / T-SEC in automazione

- **T-SEC-01** (refusal): coperto in unit/integration BE; verificare anche **CHAT-08** manuale.
- **T-ERR-01** (503 LLM): prova manuale B9-M06; UI `error_recoverable` — verificare copy e retry.
- **T-SEC-02** (no API key in bundle): `grep`/audit build `frontend/dist` — gate **G4**.

---

### D. Gap software minori (coding) — opzionali pre-pilota, utili pre-go-live

Non bloccano un test interno se le verifiche manuali passano, ma compaiono in spec/UX come incomplete:

| Gap | Fonte | Impatto | Priorità |
| --- | ----- | ------- | -------- |
| Stato UI **`error_fatal`** | UX doc + FE spec §6.6 | Config server assente (es. LLM richiesto ma disabilitato) — messaggio admin beta | P2 |
| Sezione spec §2.2 “Assente o stub” | FE spec (obsoleta) | Confonde chi legge — **aggiornare doc**, non necessariamente codice | P2 doc |
| **FR-018** progresso domande | Could | Nice-to-have | P3 |
| **FR-019** export `.txt` file | Could / post-beta | Non richiesto per pilota | P3 |
| **RoutingHints / multi-provider** R2–R5 | llm-routing-strategy | Evoluzione prodotto, non beta locale | Futuro |
| Seed opzionale **B1-003** | worklog item 5 | Comodo per demo; esiste già `DEV_ADMIN_*` + seed | P3 |
| Rate limit **per sessione** oltre IP | INF-004 / NFR-003 | Parziale oggi; rilevante solo con deploy pubblico | Post-INF |
| Object storage upload (S3) | INF, cloud guide | Locale usa `UPLOAD_DIR` — OK per test | Post-INF |

#### D.1 Requisiti Should da valutare in pilota

- **FR-017** feedback utile/non utile — UI presente; verificare in API mode (B6-M05, CHAT-09).
- **FR-020** etichette sezioni per tipo attività — dipende da qualità LLM/GENERATE; rubric in pilota.
- **NFR-006 / NFR-007 / NFR-008** — responsive Playwright esiste; audit a11y manuale (A11Y-01/02) consigliato.

---

### E. Requisiti Must — matrice “implementato vs da validare”

| Requisito | Implementazione attesa | Cosa manca per dichiararlo chiuso |
| --------- | ---------------------- | --------------------------------- |
| FR-001 … FR-015 | Orchestrator + UI stati | Checklist §18 CHAT + B7/B9 manuali |
| FR-016 Limiti/privacy | `LimitsPage` + API legal admin | **G5:** testo allineato a stack reale (PostgreSQL/Supabase, invio dati a Anthropic se LLM on) — verificare copy i18n non obsoleta (“MySQL”, “solo browser”) |
| FR-021 Auth | BE + FE | Item 12, AUTH-* §18 |
| FR-022 Persistenza | BE + hydrate FE | B3-M*, BE-03 |
| FR-023 Upload | BE + FE | B5-M*, FILE-* §18 |
| FR-024 i18n | FE + locale utente in DB | I18N-* §18, B6-M02 |
| NFR-001 | LLM server-side | T-SEC-02 audit bundle |
| NFR-003 | Rate limit | B10-M01/M02 |
| NFR-004 | Max chars | CHAT-11, B4-M04 |
| NFR-005 | Log senza content | Review log BE con LLM on |

---

### F. Documentazione — allineamento (2026-11-01)

Sync doc ↔ decisioni/repo eseguito. Fonte emendamenti: [`docs/product/decisions/implementation-amendments.md`](docs/product/decisions/implementation-amendments.md).

| Documento | Stato |
| --------- | ----- |
| `beta-roadmap.md`, `backend-implementation-plan.md`, `technical-architecture.md` | Aggiornati (Postgres, B0–B10) |
| `product-requirements.md` v1.2, `open-questions-and-decisions.md` v1.3 | Aggiornati |
| `frontend-implementation-spec.md` v1.4, `frontend-backend-operations.md` v1.1 | Aggiornati |
| `frontend-roadmap.md`, `testing-and-evaluation.md`, `database-design.md` | Aggiornati |
| `backend/tests/fixtures/eval/README.md` | Creato (EV da popolare) |
| Copy i18n Limiti/privacy (FR-016) | Allineato in `frontend/src/i18n/locales/*.ts` |

Manutenzione: dopo ogni milestone, aggiornare `implementation-amendments.md` + checklist in questo file.

Screenshot UI: [`docs/product/experience/screenshots/README.md`](docs/product/experience/screenshots/README.md).

---

### G. Esplicitamente fuori scope — “primissima versione test” locale

Non serve completare questi punti per iniziare test con founder/team su localhost:

- **Blocco 11 / M6 / INF-001 … INF-006** — Docker, staging cloud, CDN, WAF, multi-regione.
- **BETA-100 … BETA-112** — pilota utenti esterni, Go/No-Go formale (dopo test interno).
- **Multi-tenant team / workspace** — OQ, post-beta.
- **Pagamenti** — OQ-007.
- **Routing automatico multi-LLM** (Gemini/OpenAI) — R2–R5.
- **ADR-002** Anthropic dedicato — nice doc, non blocca test.

---

### H. Sequenza consigliata (ordine operativo)

1. **Ambiente stabile** — Supabase OK, migrate deploy, PM2 start, health `db: true`.
2. **CI verde** — `npm run ci` in root; fix eventuali rossi.
3. **Passata mock** — §18 PRE/PUB/AUTH rapid su `VITE_USE_MOCK=true`.
4. **Passata API** — spuntare worklog 12, 18, 21, 25, 29, 35 + §18 BE/CHAT/FILE/SET/SH.
5. **LLM reale** — B9-M01…M07 con chiave Anthropic; registrare costi/token in admin (tab Uso API).
6. **OAuth** — B8 se serve test Google; altrimenti documentare “skipped”.
7. **Admin** — verifica §8 admin plan con account dev admin.
8. **Automazione** — far girare `npx playwright test e2e/smoke-api.spec.ts` con stack up; pianificare T-E2E-02/04/05.
9. **Dataset EV** — creare EV-01, EV-02 in fixtures; prima sessione rubric (G2).
10. **Triage** — bug P0 da AUTH/CHAT/FILE in API mode; solo dopo aprire pilota BETA-100+.

---

### I. Checklist riassuntiva — “pronta fase test” (spuntare)

**Ambiente e build**

- [ ] `GET /api/health` → `{ ok: true, db: true }`
- [ ] Frontend API mode configurato e login reale ok
- [ ] `npm run ci` verde su `main`
- [ ] PM2 o script avvio documentato per chi testa

**Funzionale Must (manuale)**

- [ ] Worklog: voci 12, 18, 21, 25, 29, 35, 38 (se OAuth) completate
- [ ] FE spec §18: almeno AUTH, SH, CHAT, FILE, BE in API mode
- [ ] B9 con `LLM_ENABLED=true` almeno su scenari EV-01 ed EV-02
- [ ] Admin: verifica manuale account §8
- [ ] FR-016 / G5: pagina Limiti e privacy corretta per stack attuale
- [ ] G4: nessuna chiave Anthropic/session secret nel bundle FE

**Qualità pilota (fase test)**

- [ ] `tests/fixtures/eval/` creato con EV-01, EV-02 (+ istruzioni rubric)
- [ ] Prima sessione rubric 2 revisori su ≥2 casi
- [ ] `smoke-api.spec.ts` eseguito con successo almeno una volta
- [ ] Backlog P0 bug aperto = 0 prima di reclutare esterni

**Dopo la fase test (non ora)**

- [ ] INF cloud (M6)
- [ ] E2E full stack in CI (G1 completo)
- [ ] Pilota 5–10 utenti (BETA-100+)
- [ ] Go/No-Go founder (BETA-110/111)

---

*Fine analisi gap — aggiornare questa sezione quando si chiudono le checkbox o si aprono nuovi P0 dal pilota.*
