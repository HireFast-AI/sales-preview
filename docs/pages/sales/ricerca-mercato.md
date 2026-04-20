# Ricerca Mercato

**Route**: `/sales/ricerca-mercato`
**File**: [src/pages/sales/SalesRicercaMercato.tsx](../../../src/pages/sales/SalesRicercaMercato.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: tutti i sales (POD Director/Admin vedono anche ricerche del team via filtro — oggi filtro team non implementato)

---

## 1. Overview

Elenco delle indagini di mercato generate per ciascuna lead: stima pool candidati, concorrenza e score di probabilità di chiusura. È lo "strato di validazione pre-preventivo": il sales controlla qui se una lead è davvero lavorabile prima di costruire un'offerta. Dalla lista si apre il dettaglio (`SalesRicercaDetail`) per approvare l'indagine per la fase preventivo.

---

## 2. Viste & Stati

| ID vista | Trigger                           | Descrizione                                                       | Componenti rilevanti                                       |
| -------- | --------------------------------- | ----------------------------------------------------------------- | ---------------------------------------------------------- |
| default  | mount                             | Lista raggruppata (single-section) con score hero, filtro `all`   | `IOSNavBar`, `SegmentedControl`, `GroupedList`             |
| approved | click segmented `Approvate`       | Solo indagini con `approvedForQuote=true`, chip verde             | `SegmentedControl`, `GroupedList`                          |
| pending  | click segmented `In attesa`       | Solo indagini con `approvedForQuote=false`, chip arancione        | `SegmentedControl`, `GroupedList`                          |
| filtered | digita in search box              | Filtro client-side su jobTitle/locationLabel/company.businessName | input `Search`                                             |
| empty    | nessun risultato per filtro+query | `EmptyState` con icona `FileSearch` — copy "Nessuna ricerca"      | [EmptyState](../../../src/components/sales/EmptyState.tsx) |
| loading  | n/a (mock sincrono)               | **Non implementato** — in produzione serve skeleton rows          | —                                                          |
| error    | n/a                               | **Non implementato** — nessun fallback su fetch fallita           | —                                                          |

---

## 3. Dati & Sorgenti

| Entità UI         | Mock (src/data/mock/sales.ts)           | Schema reale (v6.1)                                      | Note                                             |
| ----------------- | --------------------------------------- | -------------------------------------------------------- | ------------------------------------------------ |
| Lista indagini    | `salesMarketInvestigations` (riga 3284) | `sales.market_investigations`                            | Relazione 1-1 con `sales.leads` via `lead_id`    |
| Azienda collegata | `getCompanyById(mi.companyId)`          | `core.companies`                                         | Join su `company_id`                             |
| Score probabilità | `mi.successProbabilityScore` (0-100)    | `sales.market_investigations.success_probability_score`  | Colorazione: <50 rosso, 50-74 arancio, 75+ verde |
| Pool candidati    | `mi.candidatePoolEstimate`              | `sales.market_investigations.candidate_pool_estimate`    | Stima AI/manuale                                 |
| Flag approvazione | `mi.approvedForQuote`                   | `sales.market_investigations.approved_for_quote` (bool)  | Gate verso generazione preventivo                |
| Città/job title   | `mi.locationCity`, `mi.jobTitle`        | `sales.market_investigations.location_city`, `job_title` | Usati anche in ricerca                           |

---

## 4. Pulsanti & Azioni

| Azione            | Trigger UI                   | Effetto attuale (mock)                                     | Effetto target (produzione)                                                            | Gated da            |
| ----------------- | ---------------------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------- |
| Cerca testuale    | input `Cerca figura…`        | Filter client-side in-memory (`useMemo` su array completo) | Full-text su Supabase (`websearch_to_tsquery` su `job_title`,`location_label`,company) | —                   |
| Filtro stato      | `SegmentedControl` 3 opzioni | Filter client-side su `approvedForQuote`                   | Query con `WHERE approved_for_quote = ?` + paginazione                                 | —                   |
| Tap riga indagine | click item `GroupedList`     | `navigate(/sales/ricerca-mercato/:id)` via `to:` prop      | Stesso navigate + prefetch detail + analytics `market_investigation_opened`            | —                   |
| — (mancante)      | CTA "Nuova indagine"         | **Assente** — oggi l'indagine si crea solo dal flusso lead | Button + modal per generare indagine AI su lead senza `marketInvestigationId`          | Lead senza indagine |
| — (mancante)      | bulk approve                 | **Assente**                                                | Checkbox multi-select + "Approva N" per director                                       | `isDirector`        |
| — (mancante)      | export CSV                   | **Assente**                                                | Bottone export per analisi offline                                                     | —                   |

---

## 5. Componenti chiave

- [IOSNavBar](../../../src/components/ios/IOSNavBar.tsx) — large title + subtitle con conteggi
- [SegmentedControl](../../../src/components/ios/IOSPrimitives.tsx) — filtro stato
- [GroupedList](../../../src/components/ios/GroupedList.tsx) — lista iOS-style con leading/title/subtitle/trailing
- [EmptyState](../../../src/components/sales/EmptyState.tsx) — stato vuoto
- `scoreColor()` — helper locale per badge score (non estratto)

---

## 6. Integrazioni future

- **Supabase tables**: `sales.market_investigations`, `sales.market_investigation_kpis`, `sales.market_investigation_sources`, `sales.market_investigation_actions`, `core.companies`, `sales.leads`
- **RLS policy**: sales vede solo proprie + team POD (join su `users.pod_team_id`); director vede intero POD; admin tutto
- **Realtime channel**: `market_investigations:pod_<id>` per aggiornamento score/approvazione live (caso AI job async)
- **Edge function**: `generate-market-investigation` (OpenAI + scraping LinkedIn/Indeed + INPS open-data) — popola KPI, sources, actionPoints
- **n8n workflow**: `market-investigation-refresh` settimanale su indagini > 30gg per ricalcolare pool
- **AI call**: Claude/GPT per `executiveSummary1/2`, `estimationProcess`, `modelDescription`; embedding per similarità tra ricerche (riuso KPI)
- **Analytics event**: `market_investigation_list_viewed`, `market_investigation_filter_changed`, `market_investigation_opened`

---

## 7. UX/UI review

### Funziona bene

- Score a 32px con semaforo colore comunica priorità in un colpo d'occhio — riga scansionabile in <1s.
- Single-section `GroupedList` con leading-score rende la lista molto "product-y" rispetto a una tabella.
- Filtro + ricerca sono sullo stesso livello visuale, no annidamenti: tempi risposta immediati (in-memory).
- Conteggi in segmented control (`Tutte 12 · Approvate 4`) danno feedback senza cliccare.

### Attriti UX attuali

- Nessun sort esplicito: ordine dettato dall'array mock (by `createdAt` implicito?). Serve ordinamento per score/data/pool.
- Empty state è unico: non distingue "nessuna indagine creata" da "filtro troppo stretto" — la seconda situazione dovrebbe offrire "Rimuovi filtri".
- Manca CTA positiva per creare una nuova indagine — il sales deve tornare alla lead, flusso non lineare.
- Il chip "In attesa" arancione è meno leggibile del verde (contrast #a85c07 su var(--color-orange-soft) borderline WCAG AA su sfondo chiaro).
- Il numero score è identico su mobile e desktop (32px) — su mobile competo con title/subtitle per spazio.

### Coerenza UI

- Tipografia: ok — serif numerico coerente col pattern "numero che conta" già usato su Home.
- Spacing: ok — `space-y-5 max-w-[840px]` allineato alle altre pagine (Preventivi, Appuntamenti list).
- Colore: ok per verde/rosso, arancione soft borderline — valutare token dedicato `--color-warning-text-on-soft`.
- Mobile-first: ok — layout fluido; su largo rimane 840px, ma manca stato vuoto su schermi <360px (leading da 48 a 40).

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo | Proposta                                                                                     | Effort |
| --- | -------- | ---- | -------------------------------------------------------------------------------------------- | ------ |
| 1   | P0       | UX   | Ordinamento selezionabile (score desc default, poi data, pool) — oggi ordine implicito       | S      |
| 2   | P0       | UX   | Empty state differenziato: "Nessuna indagine creata" vs "Filtri vuoti" con bottone reset     | S      |
| 3   | P1       | UX   | CTA "Nuova indagine" in navbar: apre bottom-sheet con dropdown lead (quelle senza `miId`)    | M      |
| 4   | P1       | UI   | Skeleton rows durante caricamento reale (3 righe, leading pulsante)                          | S      |
| 5   | P1       | UX   | Filtri avanzati (città, score range, pool range) in drawer — utile con >50 indagini          | M      |
| 6   | P2       | UI   | Mostrare mini-avatar azienda (se `logoUrl` nel CRM) invece del solo businessName             | M      |
| 7   | P2       | UX   | Swipe-left su mobile per "Approva" (director) o "Archivia"                                   | M      |
| 8   | P2       | A11y | Badge "Approvata/In attesa" con icona oltre al colore (contrast AA + screen reader friendly) | S      |
| 9   | P3       | UI   | Header section con distribuzione score (mini barchart rosso/arancio/verde)                   | M      |
| 10  | P3       | UX   | Raggruppamento opzionale per città o per sales owner (toggle nell'header)                    | L      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- Ruolo di creazione: il sales può innescare manualmente una nuova indagine o arriva solo dal flusso lead? Chiarire ownership.
- Soglia score per "approvabile": è hardcoded 75+ o configurabile per POD?
- Una lead può avere più indagini storiche (versioning) o solo una? Schema attuale 1-1 ma logica business può richiedere refresh conservativi.
- Gestione indagini obsolete: auto-scadenza dopo N giorni o revisione manuale?
- `approvedForQuote` implica trigger automatico di generazione preventivo o resta manuale?
