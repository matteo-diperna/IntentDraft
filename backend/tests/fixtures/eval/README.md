# Evaluation fixtures (EV-*)

Dataset per rubric umana e gate **G2/G3** — vedi [`docs/product/quality/testing-and-evaluation.md`](../../../../docs/product/quality/testing-and-evaluation.md) §4.

| ID | Caso pilota | Stato |
| -- | ----------- | ----- |
| EV-01 | Appunti / note strutturate | Da aggiungere (`ev-01.json` + input utente) |
| EV-02 | Post LinkedIn breve | Da aggiungere |
| EV-03 … EV-15 | Casi complessi / refusal / professionali | Backlog pilota |

Formato suggerito per file `ev-NN.json`:

```json
{
  "id": "EV-01",
  "locale": "it",
  "userMessages": ["..."],
  "expected": {
    "minQuestions": 0,
    "mustRefuse": false,
    "rubricNotes": "..."
  }
}
```

Non committare dati personali reali senza consenso e anonimizzazione.
