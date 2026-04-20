# Ricerca Mercato · Dettaglio

**Route**: `/sales/ricerca-mercato/:id`
**File**: [src/pages/sales/SalesRicercaDetail.tsx](../../../src/pages/sales/SalesRicercaDetail.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: tutti i sales (approvazione solo se owner POD — oggi non gated)

---

## 1. Overview

Report completo di un'indagine di mercato: hero dark con score grande, executive summary in due paragrafi, griglia KPI, to-do list di azioni consigliate e fonti (link esterni). È la pagina che il sales mostra al cliente in call per giustificare il preventivo e la fattibilità del match. CTA finale: approvare l'indagine per sbloccare il preventivo.

---

## 2. Viste & Stati

| ID vista   | Trigger                         | Descrizione                                                               | Componenti rilevanti                               |
| ---------- | ------------------------------- | ------------------------------------------------------------------------- | -------------------------------------------------- |
| default    | param `id` valido               | Hero + summary + KPI grid + actions + sources + (conditional) CTA approva | `IOSNavBar`, `IOSCard`, `GroupedList`, `IOSButton` |
| pending    | `mi.approvedForQuote === false` | CTA arancione full-width in fondo "Approva per preventivo"                | `IOSButton tone=orange`                            |
| approved   | `mi.approvedForQuote === true`  | CTA nascosta; nessun altro feedback visivo evidente                       | —                                                  |
| no-kpis    | `mi.kpis.length === 0`          | Card KPI non renderizzata                                                 | —                                                  |
| no-actions | `mi.actionPoints.length === 0`  | Sezione "Da fare" nascosta                                                | —                                                  |
| no-sources | `mi.sources.length === 0`       | Sezione "Fonti" nascosta                                                  | —                                                  |
| not-found  | `id` non matcha                 | `<Navigate to="/sales/ricerca-mercato" replace />` (redirect silenzioso)  | `react-router Navigate`                            |
| loading    | n/a                             | **Non implementato** — niente skeleton in attesa fetch reale              | —                                                  |
| error      | n/a                             | **Non implementato** — nessun toast su errore carico                      | —                                                  |

---

## 3. Dati & Sorgenti

| Entità UI                   | Mock (src/data/mock/sales.ts)                                                             | Schema reale (v6.1)                                                    | Note                                                           |
| --------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------- |
| Indagine (root)             | `salesMarketInvestigations.find(id)`                                                      | `sales.market_investigations`                                          | Select by PK                                                   |
| Azienda                     | `getCompanyById(mi.companyId)`                                                            | `core.companies`                                                       | Mostrata in eyebrow + summary                                  |
| Competition level           | `mi.competitionLevel`                                                                     | `sales.market_investigations.competition_level` (enum low/medium/high) | Label via map `competitionLabel`                               |
| Executive summary           | `mi.executiveSummary1`, `mi.executiveSummary2`                                            | `sales.market_investigations.executive_summary_1/2`                    | Due paragrafi narrativi generati da AI                         |
| KPI grid                    | `mi.kpis[]` (SalesMarketInvestigationKpi)                                                 | `sales.market_investigation_kpis` (FK `investigation_id`)              | Campo `isHighlight` pilota stile teal                          |
| Action points               | `mi.actionPoints[]`                                                                       | `sales.market_investigation_actions`                                   | Ordinati per `displayOrder` (non rispettato oggi: map diretto) |
| Fonti                       | `mi.sources[]` (label/url/category)                                                       | `sales.market_investigation_sources`                                   | `target="_blank"` senza tracking                               |
| Pool/disponibili/assorbenti | `mi.candidatePoolEstimate`, `mi.immediateAvailabilityCount`, `mi.absorbingCompaniesCount` | colonne omonime snake_case                                             | Hero stat boxes                                                |
| Data creazione              | `mi.createdAt` → `formatDateIt()`                                                         | `sales.market_investigations.created_at`                               | Localizzato IT                                                 |

---

## 4. Pulsanti & Azioni

| Azione                 | Trigger UI                    | Effetto attuale (mock)               | Effetto target (produzione)                                                                                     | Gated da                     |
| ---------------------- | ----------------------------- | ------------------------------------ | --------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| Back                   | freccia IOSNavBar             | `navigate(-1)`                       | Stesso + preserva filtri lista                                                                                  | —                            |
| Apri fonte             | link `Apri` in sources        | `window.open(url)` new tab           | Stesso + analytics `source_clicked` + telemetry per domain                                                      | —                            |
| Approva per preventivo | IOSButton orange full-width   | **NO-OP** — nessun handler collegato | `UPDATE sales.market_investigations SET approved_for_quote=true, approved_by, approved_at WHERE id=?` + refresh | Solo owner + stato `pending` |
| — (mancante)           | Genera preventivo da indagine | **Assente**                          | Post-approva: CTA secondaria "Genera preventivo" con pre-fill `leadId` + `quoteType` suggerito                  | `approvedForQuote=true`      |
| — (mancante)           | Rigenera indagine             | **Assente**                          | Trigger edge-function `generate-market-investigation` (force refresh)                                           | Owner/Director               |
| — (mancante)           | Condividi/Export PDF          | **Assente**                          | Genera PDF brandizzato → download o link condivisibile (per cliente in call)                                    | —                            |
| — (mancante)           | Note interne                  | **Assente**                          | Campo note libero per sales (non visibile al cliente)                                                           | —                            |
| Redirect id invalido   | `Navigate`                    | Silent redirect lista                | Preferibile toast "Indagine non trovata" + redirect                                                             | —                            |

---

## 5. Componenti chiave

- [IOSNavBar](../../../src/components/ios/IOSNavBar.tsx) — back + title = jobTitle
- [IOSCard](../../../src/components/ios/IOSPrimitives.tsx) — wrapper executive summary + KPI grid
- [GroupedList](../../../src/components/ios/GroupedList.tsx) — sezioni "Da fare" e "Fonti"
- [IOSButton](../../../src/components/ios/IOSPrimitives.tsx) — CTA approva
- `HeroStat` (locale) — box bianco su dark card per metric hero
- Hero dark box inline (non estratto come componente) — gradient radiale arancione

---

## 6. Integrazioni future

- **Supabase tables**: `sales.market_investigations` + 3 tabelle child (kpis/actions/sources) + `core.companies`
- **RLS policy**: read se owner OR stesso POD; write `approved_for_quote` solo owner o director
- **Realtime channel**: `market_investigation:<id>` per aggiornamento live se un altro user la sta rigenerando
- **Edge function**: `generate-market-investigation` (rigenerazione); `export-market-investigation-pdf` (tag brandizzato + fonte dati)
- **n8n workflow**: on-approve → trigger "prepare-quote-draft" che crea `sales.quotes` in `draft` col `quoteType` suggerito
- **AI call**: Claude per narrativa summary; GPT-5 per matching salary benchmarks; embedding su KPI per similarità tra indagini
- **Analytics event**: `market_investigation_viewed`, `market_investigation_approved`, `source_opened`, `pdf_exported`

---

## 7. UX/UI review

### Funziona bene

- Hero dark con score 64px + gradient arancione = "wow factor" per condivisione in call, il cliente ricorda il numero.
- Tre stat boxes (Pool/Disponibili/Concorrenti) sono la giusta sintesi visiva prima della narrativa.
- KPI grid con highlight teal distingue i 2-3 dati chiave dal rumore — pattern coerente col brand.
- Lista fonti con chip category + link esterno comunica "abbiamo basi dati verificabili" (trust signal).

### Attriti UX attuali

- CTA "Approva per preventivo" è un pulsante morto: click non produce feedback né stato ottimistico. Rischio che il sales clicchi più volte pensando sia broken.
- Nessun modo visibile di capire "chi ha creato l'indagine" / "quando è stata approvata" — manca audit trail minimale visibile.
- La scomparsa della CTA a post-approva lascia la pagina senza un next-step: il sales deve ricordare di tornare alla lead e creare il preventivo.
- Le fonti aprono in new tab senza preview (icona, favicon, dominio) — UX da "bookmark list" anni 2010.
- Hero gradient radiale è fisso in basso-destra, non cambia con score: su score basso (rosso) la decorazione arancione confonde la semantica.
- `estimationProcess`, `laborDemandSummary`, `salaryBenchmarkSummary`, `modelDescription` sono nel mock ma **non renderizzati** — potenziale spreco di contenuto AI-generato.

### Coerenza UI

- Tipografia: ok — font-serif score coerente con list e con Home tiles.
- Spacing: ok — `space-y-5` + cards con padding md.
- Colore: ok dark hero; soft teal/orange coerenti; controllare contrast del testo 75% white su ink al 100%.
- Mobile-first: ok — grid KPI passa da 3 a 2 col, hero resta leggibile. CTA full-width ok.

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo   | Proposta                                                                                                    | Effort |
| --- | -------- | ------ | ----------------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0       | UX/Bug | Collegare handler a "Approva per preventivo" (oggi no-op). Stato ottimistico + undo 5s + toast              | M      |
| 2   | P0       | UX     | Dopo approvazione: sostituire CTA con "Genera preventivo" (precompila lead + quoteType)                     | M      |
| 3   | P1       | UX     | Audit strip in header: "creata da X il ggMMM · approvata da Y" + pulsante rigenera per director             | M      |
| 4   | P1       | UI     | Usare i campi non renderizzati (`estimationProcess`, `salaryBenchmarkSummary`, ...) in sezioni collassabili | M      |
| 5   | P1       | UX     | Export PDF cliente-facing + link condivisibile (Supabase signed URL)                                        | L      |
| 6   | P2       | UI     | Hero gradient color-synced col score (rosso/arancio/verde soft)                                             | S      |
| 7   | P2       | UX     | Mostrare favicon/dominio sulle fonti + badge affidabilità (official vs job_board)                           | S      |
| 8   | P2       | A11y   | Hero score con `aria-label="Score probabilità 78 su 100"` + ruolo heading su executive summary              | S      |
| 9   | P3       | UX     | Note interne del sales (non visibili al cliente) per track di obiezioni/punti trattativa                    | M      |
| 10  | P3       | UX     | Comparativa: pulsante "Confronta con indagine simile" (embedding similarity)                                | L      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- L'approvazione è reversibile? Schema prevede history o solo flag boolean?
- `approvedForQuote=true` crea automaticamente un record in `sales.quotes` (draft) o resta manuale?
- Come gestire indagini "vecchie" quando il cliente rilancia 6 mesi dopo: rigenerare, clonare o versionare?
- La pagina è cliente-facing (link pubblico?) o solo interna? Se pubblica, serve layout senza sidebar + token secret URL.
- Sources categoria `internal` è leggibile dal cliente? Potrebbe esporre fornitori dati strategici.
- Campi `modelDescription` e `estimationProcess` rivelano metodologia AI: vogliamo mostrarli al cliente (trust) o nasconderli (IP)?
