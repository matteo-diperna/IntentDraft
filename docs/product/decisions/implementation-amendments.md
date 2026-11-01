# Emendamenti implementativi — registro (post decisioni founder 2026-10-25)

**Documento:** `implementation-amendments.md`  
**Scopo:** tracciare decisioni **operative** prese durante l’implementazione, quando divergono dal testo storico di roadmap/requirements (es. MySQL) o quando funzionalità passano da «rinviato» a «fatto».  
**Non sostituisce** [`open-questions-and-decisions.md`](open-questions-and-decisions.md): le OQ restano la fonte delle intenzioni founder; qui si allinea la **realtà repo**.

---

## 1. Database — PostgreSQL (Supabase) al posto di MySQL

| Campo | Valore |
| ----- | ------ |
| **Data** | 2026-10-29 |
| **Contesto** | OQ-003/OQ-010/OQ-015 richiedevano «DB relazionale + account»; i doc 2026-10-25 citavano **MySQL** come ipotesi localhost. |
| **Decisione implementativa** | **PostgreSQL** managed (**Supabase**) + Prisma `provider = postgresql`; tabelle fisiche `idt_*`. |
| **Motivazione** | DB cloud già disponibile, migrazioni Prisma, nessun vincolo prodotto sul motore SQL specifico. |
| **Doc aggiornati** | ADR-001 (Decisione 1), `database-design.md`, README, `backend/README.md`. |
| **NFR-013** | Soddisfatto con migrazioni versionate in `backend/prisma/migrations/` (non MySQL locale). |

---

## 2. Backend e integrazione FE — FOUND/BETA eseguiti (B0–B10)

| Campo | Valore |
| ----- | ------ |
| **Data** | 2026-10-29 … 2026-10-31 |
| **Decisione** | Monorepo `frontend/` + `backend/` operativo: auth cookie, CRUD, file, orchestrator mock (M4), Anthropic ANALYZE/GENERATE (M5), rate limit, CI GitHub Actions. |
| **FE** | `VITE_USE_MOCK=false` + adapter API; motore demo client-side resta per sviluppo UI isolato. |
| **Riferimento operativo** | [`../../../worklog.md`](../../../worklog.md) · [`../planning/backend-implementation-plan.md`](../planning/backend-implementation-plan.md) §12 |

---

## 3. OQ-002 — Anthropic Claude Sonnet

| Campo | Valore |
| ----- | ------ |
| **Stato doc originale** | Provider accettato; integrazione **Deferred** fino a fondamenta FE/BE. |
| **Stato repo (2026-10-31)** | **Implementato** (B9): `LLM_ENABLED`, client server-side, fallback mock engine se disabilitato o errore provider. |
| **ADR-002** | Formalizzazione breve in §4 sotto; documento ADR dedicato opzionale. |

---

## 4. ADR-002 (sintesi) — LLM server-side

| Aspetto | Scelta |
| ------- | ------ |
| Provider | Anthropic Claude (`ANTHROPIC_MODEL`, default Sonnet family) |
| Fasi | ANALYZE (IntentModel JSON) → GENERATE (PromptArtifact JSON) |
| Segreti | Solo `backend/.env`; mai nel bundle frontend (NFR-001) |
| Usage | Eventi in `idt_usag`; admin console |
| OQ-008 | Output prodotto = **testo prompt** copiabile; il server **invoca** il modello per interpretazione/composizione, non sostituisce l’esecuzione arbitraria del task utente in-app |

---

## 5. Console admin (RBAC)

| Campo | Valore |
| ----- | ------ |
| **Data** | 2026-10-31 |
| **Decisione** | Ruolo `admin`, route `/api/admin/*`, UI `/app/admin` (fasi A1–A10). Non era nel pacchetto FR originale 2026-10-25; pianificato in [`../planning/admin-capabilities-plan.md`](../planning/admin-capabilities-plan.md). |

---

## 6. Prodotto — chiarimento valore (2026-11)

| Campo | Valore |
| ----- | ------ |
| **Messaging** | IntentDraft non si limita a «generare testo da incollare»: orchestra intento, chiarimenti e prompt engineering **via LLM server-side** (Claude o mock); routing multi-provider in roadmap [`../ai/llm-routing-strategy.md`](../ai/llm-routing-strategy.md). |
| **Doc** | README root (EN), [`../ai/llm-routing-strategy.md`](../ai/llm-routing-strategy.md), worklog § analisi gap |

---

## 7. Ancora **non** deciso / rinviato

- **OQ-007** pagamenti — Deferred  
- **OQ-012** hosting produzione / GDPR completo — Deferred (guida INF scritta, non deployato)  
- **Multi-LLM automatico** (R2–R5) — pianificato, non implementato  
- **Team/workspace** multi-tenant — fuori scope beta  

---

## 8. Manutenzione documentazione

Quando si chiude una milestone in [`worklog.md`](../../../worklog.md):

1. Aggiornare checklist in `backend-implementation-plan.md` §12 e, se serve, voci in `beta-roadmap.md`.  
2. Verificare assenza riferimenti **MySQL-only** o «greenfield» nei doc architecture/requirements.  
3. Aggiungere riga in questo file se la decisione è nuova e non coperta da OQ.
