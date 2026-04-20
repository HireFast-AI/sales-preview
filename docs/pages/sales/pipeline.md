# Sales Pipeline — Kanban / Accordion

**Route**: `/sales/pipeline`
**File**: [src/pages/sales/SalesPipeline.tsx](../../../src/pages/sales/SalesPipeline.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: tutti (il sales vede il proprio lavoro, il director puo' switchare con filtro "Solo mie" OFF)

---

## 1. Overview

Vista kanban orizzontale (desktop) / accordion verticale (mobile) dell'intera pipeline commerciale. Raggruppa le 12 micro-status in 4-5 macro-colonne (qualify / offer / sign / admin_issue / closed) con conteggi lead e posizioni richieste. E' lo strumento "weekly review" del sales e del director: capire dove si impilano le lead e dove c'e' bottleneck. A differenza della Lead list (ordinata), qui la struttura e' colonnare e ogni card e' una sintesi veloce (stato, preventivi, prossimo appt).

---

## 2. Viste & Stati

| ID vista          | Trigger                                    | Descrizione                                                          | Componenti rilevanti                |
| ----------------- | ------------------------------------------ | -------------------------------------------------------------------- | ----------------------------------- |
| desktop_kanban    | Breakpoint `md+`                           | 3 o 4 colonne affiancate (admin_issue appare solo se presente)       | `PipelineColumn`, `PipelineCard`    |
| mobile_accordion  | `< md`                                     | Sezioni collassabili full-width; qualify+offer aperte di default     | `MobileSection`, `PipelineCard`     |
| closed_collapsed  | `closedTotal > 0`, `showClosed = false`    | Sezione "Chiuse" compressa in fondo                                  | Button + `ChevronDown`              |
| closed_expanded   | Click sulla sezione chiuse                 | Griglia 3 colonne desktop (customer / discarded / postponed)         | `ClosedGrid`                        |
| empty_filtered    | `filtered.length === 0` con filtri attivi  | `EmptyState` "Nessun risultato" + bottone Reset filtri               | `EmptyState` tone=neutral           |
| empty_column      | Lead presenti globalmente ma colonna vuota | Testo muted "Nessuna lead in questa fase"                            | —                                   |
| search_active     | `search.length > 0`                        | X clear button visibile; badge conteggi ridotti                      | Input `Search`                      |
| quick_action_open | Tap singolo su una card                    | `QuickActionSheet` bottom sheet con call/email/whatsapp/note/advance | `BottomSheet` via `useQuickActions` |
| loading           | Non gestito — dati sincroni                | —                                                                    | —                                   |
| error             | Non gestito                                | —                                                                    | —                                   |

---

## 3. Dati & Sorgenti

| Entita' UI       | Mock (src/data/mock/sales.ts)               | Schema reale                                         | Note                                                            |
| ---------------- | ------------------------------------------- | ---------------------------------------------------- | --------------------------------------------------------------- |
| Lista lead       | `salesLeads` (array diretto)                | `sales.leads`                                        | Filtro client-side su macro-stage, urgency, stale, assegnazione |
| Azienda card     | `getCompanyById(companyId)`                 | `core.companies`                                     | Mostra `businessName`                                           |
| Contatto card    | `getContactById(primaryContactId)`          | `core.contacts`                                      | Mostra `fullName` e `role`                                      |
| Preventivi       | `getQuotesByLead(leadId)`                   | `sales.quotes`                                       | Badge "Preventivo OK / inviato / bozza / ko"                    |
| Prossimo appt    | `getNextAppointmentByLead(leadId)`          | `sales.appointments` (TBD view `v_next_appointment`) | Pill "Oggi/Domani/gg/data HH:mm"                                |
| Macro-stage      | `getMacroStage(status)`                     | derivato lato DB via `sales.v_leads_macro_stage`     | Map 12→5                                                        |
| Stale flag       | `lead.isStale` + `daysSinceLastContact`     | `sales.leads.is_stale` (computed col/view TBD)       | Badge rosso                                                     |
| Urgency          | `lead.urgencyLevel`                         | `sales.leads.urgency_level`                          | Filtro `high`/`critical`                                        |
| Headcount totale | Somma `lead.requestedHeadcount` per colonna | —                                                    | Header colonna                                                  |

---

## 4. Pulsanti & Azioni

| Azione                    | Trigger UI                 | Effetto attuale (mock)                                                  | Effetto target (produzione)                                    | Gated da                        |
| ------------------------- | -------------------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------- | ------------------------------- |
| Search                    | Input testo                | Filtra in-memory su `businessName`, `positionTitle`, `contact.fullName` | Idem + potenzialmente full-text search lato DB                 | —                               |
| Clear search              | X in input                 | Reset search                                                            | Idem                                                           | Presenza testo                  |
| Filtro "Solo mie"         | Chip toggle                | Filtra `assignedToUserId === CURRENT_SALES_USER_ID`                     | Idem con `auth.uid()`                                          | —                               |
| Filtro "Solo urgenti"     | Chip toggle                | Filtra `urgency in (high, critical)`                                    | Idem                                                           | —                               |
| Filtro "Solo ferme"       | Chip toggle                | Filtra `isStale === true`                                               | Idem                                                           | —                               |
| Tap card (singolo)        | `onClick` su PipelineCard  | Apre `QuickActionSheet` (call/email/whatsapp/note/advance)              | Idem + log evento                                              | —                               |
| Double tap card           | `onDoubleClick`            | Navigate a `/sales/lead/:id`                                            | Idem (mobile: long press o click diretto — gesture ambigua)    | —                               |
| Expand/collapse accordion | Click su header mobile     | Toggle `openMobile[macro]`                                              | Idem, salvare preferenza utente in profile                     | —                               |
| Show/Hide Chiuse          | Click su sezione Chiuse    | Toggle `showClosed`                                                     | Idem                                                           | `closedTotal > 0`               |
| Click lead chiusa         | Click su riga `ClosedGrid` | Navigate a `/sales/lead/:id`                                            | Idem                                                           | —                               |
| Reset filtri              | Bottone in EmptyState      | Reset search + 3 chip + filtri                                          | Idem                                                           | `filtered.length === 0`         |
| Drag-and-drop card        | —                          | Non implementato                                                        | Spostare lead tra colonne adiacenti aggiornando `status` + log | Permission: only owner/director |

---

## 5. Componenti chiave

- [EmptyState](../../../src/components/sales/EmptyState.tsx) — stato vuoto con CTA reset
- [MacroStageChip](../../../src/components/sales/LeadRow.tsx) — chip colorato macro-stage (usato in ClosedGrid)
- [useQuickActions](../../../src/components/sales/QuickActionSheet.tsx) — hook che espone `open(lead)` + sheet per azioni rapide
- `PipelineColumn` (inline) — colonna kanban desktop con header, strip colorata, body scrollabile, sub-raggruppamento per micro-status
- `MobileSection` (inline) — accordion mobile con stesso contenuto
- `PipelineCard` (inline) — card lead: company, contact, position, quotes, next appt, stale badge
- `ClosedGrid` (inline) — griglia 3 colonne per customer/discarded/postponed
- `FilterChip` (inline) — chip toggle riutilizzato dalla Lead list

---

## 6. Integrazioni future

- **Supabase tables**: `sales.leads`, `sales.quotes`, `sales.appointments`, `core.companies`, `core.contacts`
- **View**: `sales.v_leads_macro_stage` — computed macro stage + stale flag lato DB
- **RLS policy**: `(assigned_to_user_id = auth.uid()) OR (pod_team_id in my_pods)` — director vede il POD
- **Realtime channel**: `sales:pipeline:<pod_id>` → re-render colonna quando una lead cambia status (anche da altri)
- **Edge function**: `advance_lead_stage(lead_id, next_status, note?)` — chiamata dal QuickActionSheet e dal futuro drag-drop, scrive audit + lead_note
- **n8n workflow**: alert Slack al director quando una colonna >threshold (es. 10 lead in "Nuovo" = setter lento)
- **AI call**: "riordina colonne per priorita' di chiusura" = score AI su (budget _ probability _ recency)
- **Analytics event**: `pipeline_viewed`, `pipeline_filter_toggle`, `pipeline_card_tap`, `pipeline_card_dbltap`, `pipeline_closed_expand`

---

## 7. UX/UI review

### Funziona bene

- Macro-stage a 4 colonne = cognitive load basso, cifra di riferimento immediata
- Sub-raggruppamento per micro-status dentro ogni colonna con label eyebrow — si capisce subito dove siamo (es. "Nuovo · 3 · Qualificato · 2")
- Strip colorata top sulle colonne e le card assicura gerarchia visiva + coerenza cross-page con i chip MacroStage
- Badge "Preventivo OK" in colonna offer/sign e "gg stale" in rosso sono segnali forti a colpo d'occhio
- Pill prossimo appt con tono "Oggi/Domani" fa l'effetto "Hey, ti serve ora"
- Reset filtri contestuale nell'empty state evita che l'utente resti bloccato
- Chiuse compresse di default, non consumano spazio nella vista primaria

### Attriti UX attuali

- Tap singolo vs double tap: pattern non standard su mobile, il sales non sa se aprire la lead con un tap o due. Il doppio tap su touch non e' nativo → `onDoubleClick` funziona male su iOS Safari
- Nessun drag-and-drop: aspettativa utente molto forte su un kanban; l'assenza delude e forza l'uso del QuickAction
- Nessuna indicazione di "blocco" per lead non-tue in modalita' director: si puo' aprire tutto, ma se la policy reale nega scritture, l'utente scopre l'errore solo al tap
- Filtri chip sono tutti "only" — nessun "Filtra per owner diverso da me" o "per settore" (limite in vista director)
- L'header pagina ("Pipeline · N attive · N chiuse") e' piccolo e non sticky con dettaglio — si perde durante scroll colonne
- Nessun conteggio value (€) per colonna — solo headcount, ma cio' che interessa al director e' revenue in-flight
- Colonne scrollable ma non con sticky header → scroll lungo → si perde il contesto colonna
- "Nessuna lead in questa fase" e' identico a "colonna vuota per filtro" — potrebbe suggerire reset

### Coerenza UI

- Tipografia: ok — gerarchia chiara header > sub-eyebrow > card title
- Spacing: ok — card a respiro giusto, 12px gap tra card
- Colore: ok — accent palette coerente (indigo/amber/emerald/red/slate) usata identicamente in Lead list e LeadDetail
- Mobile-first: ok — transizione kanban→accordion pulita, non c'e' scroll orizzontale mobile

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorita' | Tipo | Proposta                                                                                                                            | Effort |
| --- | --------- | ---- | ----------------------------------------------------------------------------------------------------------------------------------- | ------ | ---- | ----------------------------------------------------------------------------------------------- | --- |
| 1   | P0        | UX   | Unificare gesture: tap singolo = apre lead detail, long-press (mobile) o icona dedicata su card = QuickAction. Rimuovere double tap | M      |
| 2   | P0        | FN   | Leggere `?stage=qualify                                                                                                             | offer  | sign | closed` dal query param (usato da SalesHome StageCard) e aprire accordion mobile corrispondente | S   |
| 3   | P1        | FN   | Drag-and-drop card tra colonne adiacenti (qualify→offer, offer→sign) con conferma note in bottom sheet                              | L      |
| 4   | P1        | UI   | Aggiungere totale € in header colonna (somma quote value della miglior quote per lead)                                              | M      |
| 5   | P1        | UX   | Sticky sub-header colonna durante scroll (nome colonna + count restano visibili)                                                    | M      |
| 6   | P2        | UX   | Aggiungere filtri avanzati: per owner (director), per settore, per urgency=low (oggi solo "urgent" = high+crit)                     | M      |
| 7   | P2        | UI   | Badge "€ X" dentro card quando esiste quote accepted/sent, non solo chip testo                                                      | S      |
| 8   | P2        | UX   | Conteggio chip "· N" sui filtri della pipeline come fatto nella Lead list (oggi manca)                                              | S      |
| 9   | P3        | UX   | Permettere salvataggio preset filtri ("My urgent qualify")                                                                          | L      |
| 10  | P3        | UI   | Dark mode: la strip colorata top su sfondo scuro diventa piu' leggibile con outline                                                 | M      |

Legenda priorita': P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- Modalita' director: deve vedere tutto il POD di default o solo le sue come un sales? (oggi "Solo mie" e' OFF di default → vede tutte)
- Drag-and-drop: quali transizioni sono ammesse? (es. da `qualify` a `sign` saltando `offer` dovrebbe essere bloccato o richiede conferma?)
- Colonna `admin_issue`: quando appare? Oggi solo se ha almeno una lead — il PO vuole forse fissarla sempre visibile per training mentale?
- "Chiuse" include postponed e discarded assieme a customer — separare in tre sezioni o tenere grid?
- Deep-link a una card specifica (`?lead=l-123`): utile per condivisione in chat Slack?
- Performance con >500 lead: oggi e' client-side, passera' a server con paginazione?
