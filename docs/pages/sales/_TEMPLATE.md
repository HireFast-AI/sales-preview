# {{Page Name}}

**Route**: `/sales/{{path}}`
**File**: [src/pages/sales/{{ComponentName}}.tsx](../../../src/pages/sales/{{ComponentName}}.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: {{tutti | POD Director | Admin}}

---

## 1. Overview

Riga 1-3: scopo della pagina in linguaggio business. Chi la usa, cosa risolve, dove si inserisce nel flusso sales.

---

## 2. Viste & Stati

Ogni "vista" = stato visuale distinto che l'utente può vedere. Elenca tutti (incluso empty, loading, error).

| ID vista | Trigger | Descrizione | Componenti rilevanti |
| -------- | ------- | ----------- | -------------------- |
| default  | -       | -           | -                    |
| empty    | -       | -           | -                    |

---

## 3. Dati & Sorgenti

Mock attuali e fonte dati reale (schema Supabase v6.1). Usa questa sezione per guidare il binding futuro.

| Entità UI | Mock (src/data/mock/sales.ts) | Schema reale | Note |
| --------- | ----------------------------- | ------------ | ---- |
| -         | -                             | -            | -    |

---

## 4. Pulsanti & Azioni

Ogni interazione click/tap/submit. Specifica effetto attuale (mock) e effetto target (produzione).

| Azione | Trigger UI | Effetto attuale (mock) | Effetto target (produzione) | Gated da |
| ------ | ---------- | ---------------------- | --------------------------- | -------- |
| -      | -          | -                      | -                           | -        |

---

## 5. Componenti chiave

Componenti custom usati da questa pagina (con percorso).

- [ComponentName](../../../src/components/...) — ruolo

---

## 6. Integrazioni future

API, webhook, job background, notifiche, eventi telemetry. Solo cose che servono in produzione.

- **Supabase tables**: -
- **RLS policy**: -
- **Realtime channel**: -
- **Edge function**: -
- **n8n workflow**: -
- **AI call**: -
- **Analytics event**: -

---

## 7. UX/UI review

Analisi critica dello stato attuale. Cosa funziona, cosa no.

### ✅ Funziona bene

- -

### ⚠️ Attriti UX attuali

- -

### 🎨 Coerenza UI

- Tipografia: ok / da sistemare
- Spacing: ok / da sistemare
- Colore: ok / da sistemare
- Mobile-first: ok / da sistemare

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo | Proposta | Effort |
| --- | -------- | ---- | -------- | ------ |
| 1   | P0       | UX   | -        | S      |
| 2   | P1       | UI   | -        | M      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- -
