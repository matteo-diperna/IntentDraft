# Frontend screenshots — IntentDraft

Reference captures of the **implemented UI** (Italian copy, light/dark themes). Use these in reviews, portfolio README, and UX docs. Paths are relative to this folder.

| File | Route / context | Description |
| ---- | --------------- | ----------- |
| [`landing-light.png`](landing-light.png) | `/` (public) | Marketing landing: badge “Assistente prompt per l’AI”, gradient title, **Accedi** / **Registrati**, three value props (guided conversation, clearer instructions, ready for your LLM). Light theme with grid + central glow (`PublicPageFrame`, `BrandScene`). |
| [`landing-dark.png`](landing-dark.png) | `/` (public) | Same landing in **dark mode** — demonstrates token parity in `brand-tokens.css` and `.dark` semantic colors. |
| [`login-light.png`](login-light.png) | `/login` | Auth card (`AuthPageShell`): Google OAuth, email/password, link to register, “Limiti e privacy” in footer. |
| [`chat-empty-light.png`](chat-empty-light.png) | `/app` | Signed-in **empty chat**: collapsible sidebar (projects, pinned, recent, uploaded docs), hero empty state (“Descrivi in poche frasi…”), glass composer with attachment + 12k char limit. |
| [`chat-empty-dark.png`](chat-empty-dark.png) | `/app` | Same chat shell in dark mode — sidebar chrome + `AppPanelFrame` with softer scene. |
| [`admin-dashboard-light.png`](admin-dashboard-light.png) | `/app/admin` | **Admin console** dashboard tab: user/conversation/message counts, LLM usage (7d), token totals, DB health, rejected file scans. |

**Related docs:** [`../frontend-brand-ui.md`](../frontend-brand-ui.md) · [`../user-journeys-and-ux.md`](../user-journeys-and-ux.md) · [`../../planning/admin-capabilities-plan.md`](../../planning/admin-capabilities-plan.md)
