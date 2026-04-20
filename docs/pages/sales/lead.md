# Sales Lead — Lista lead attive + Nuovo lead

**Route**: `/sales/lead`
**File**: [src/pages/sales/SalesLead.tsx](../../../src/pages/sales/SalesLead.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: tutti (sales vede assegnate; director/admin con filtro "Solo mie" OFF vede tutte)

---

## 1. Overview

Lista verticale scrollabile di tutte le lead attive (esclude `customer`, `discarded`, `postponed`) con ricerca, segmented control per macro-stage e 3 filtri rapidi (urgenti / ferme / mie). E' il punto di atterraggio quando il sales vuole "lavorare la mia lista" in modalita' lineare, alternativa complementare al kanban della Pipeline. Include un FAB in basso a destra che apre il form completo a 6 sezioni "Nuovo lead / cliente" — l'unico punto di ingresso per creare un nuovo lead nel portale.

---

## 2. Viste & Stati

| ID vista           | Trigger                                | Descrizione                                            | Componenti rilevanti          |
| ------------------ | -------------------------------------- | ------------------------------------------------------ | ----------------------------- |
| default            | Segmented "Tutte" + nessun filtro      | Tutte le lead attive in lista piatta, ordine mock      | `LeadRow`                     |
| filtered_stage     | SegmentedControl su qualify/offer/sign | Solo lead di quel macro-stage                          | `LeadRow`                     |
| filtered_urgent    | Chip "Solo urgenti"                    | Lead con urgency high/critical                         | `LeadRow`                     |
| filtered_stale     | Chip "Solo ferme"                      | Lead `isStale=true`                                    | `LeadRow`                     |
| filtered_mine      | Chip "Solo mie"                        | `assignedToUserId === CURRENT_SALES_USER_ID`           | `LeadRow`                     |
| search             | Input testo                            | Match su azienda / contatto / posizione                | `LeadRow` + SearchInput       |
| empty              | `filtered.length === 0`                | EmptyState "Nessuna lead qui" + Reset se filtri attivi | `EmptyState`                  |
| new_lead_sheet     | Click FAB                              | BottomSheet "Nuovo lead / cliente" con 6 sezioni       | `NewLeadSheet`, `BottomSheet` |
| vat_lookup_loading | Click "Recupera" in sezione Azienda    | Spinner inline, disabled input                         | `IOSButton` disabled          |
| vat_lookup_done    | Risposta mock arrivata                 | Card teal con dati Camera di Commercio recuperati      | —                             |
| form_error         | Submit con campi mancanti              | Banner rosso "Inserisci X"                             | —                             |
| loading            | Non gestito — dati sincroni            | —                                                      | —                             |
| error              | Non gestito (solo validazione client)  | —                                                      | —                             |

---

## 3. Dati & Sorgenti

| Entita' UI                  | Mock (src/data/mock/sales.ts)                                  | Schema reale                                                    | Note                                                  |
| --------------------------- | -------------------------------------------------------------- | --------------------------------------------------------------- | ----------------------------------------------------- |
| Lista lead                  | `salesLeads` (mutabile: `push` / `unshift`)                    | `sales.leads`                                                   | Filtrata client-side su macro-stage ≠ closed          |
| Count macro-stage           | `useMemo` da salesLeads + `getMacroStage`                      | view `sales.v_leads_macro_stage` (TBD)                          | Badges sul SegmentedControl                           |
| Count urgent / stale / mine | `useMemo` filtri `urgencyLevel`, `isStale`, `assignedToUserId` | `sales.leads` colonne dirette                                   | Mostrati sui chip                                     |
| Nuova azienda               | push in `salesCompanies`                                       | `core.companies`                                                | Full record + campi opzionali CRM Notion              |
| Nuovi contatti              | push in `salesContacts`                                        | `core.contacts` + `core.contact_phones` + `core.contact_emails` | Multi-telefono (mobile + fisso + int.) + email        |
| Nuova lead                  | `unshift` in `salesLeads`                                      | `sales.leads`                                                   | Creata con `status='new'`, `source` mappato da origin |
| Lookup P.IVA                | `fetchCompanyDataByVat(vat)` (~1.2s delay, dati random)        | Cribis / InfoCamere / OpenAPI.it (TBD Edge fn)                  | Merge su company                                      |
| Sales assegnato             | `salesUsers`                                                   | `sales.user_profiles` / `core.users`                            | Select sezione 6                                      |
| Tipo preventivo             | enum locali `quoteTypeSegments`                                | `sales.quote_types` enum                                        | 6 opzioni                                             |
| Origine lead                | enum `originSegments`                                          | `sales.leads.origin`                                            | 7 opzioni                                             |

---

## 4. Pulsanti & Azioni

| Azione                                        | Trigger UI                               | Effetto attuale (mock)                                                                        | Effetto target (produzione)                                                       | Gated da                                                                 |
| --------------------------------------------- | ---------------------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------ | --- | -------- |
| SearchInput                                   | Digita testo                             | Filtro client-side                                                                            | `RPC search_leads(q, limit)` con `websearch_to_tsquery`                           | —                                                                        |
| Clear search                                  | X                                        | Reset query                                                                                   | Idem                                                                              | Query non vuota                                                          |
| SegmentedControl stage                        | Tap tab                                  | Filtra macro-stage                                                                            | Idem, con badge real-time                                                         | —                                                                        |
| Chip Solo urgenti/ferme/mie                   | Tap chip                                 | Toggle booleani                                                                               | Idem                                                                              | —                                                                        |
| Reset filtri                                  | Bottone EmptyState                       | Reset tutto                                                                                   | Idem                                                                              | Stato empty filtrato                                                     |
| Tap LeadRow                                   | Intera riga                              | Navigate `/sales/lead/:id` (swipe actions gestite da LeadRow)                                 | Idem                                                                              | —                                                                        |
| Swipe row                                     | Gesture su LeadRow (prop `swipeActions`) | Apre azioni (vedi LeadRow)                                                                    | Idem                                                                              | Solo touch                                                               |
| FAB "Nuovo lead"                              | Tap floating button                      | Apre `NewLeadSheet`                                                                           | Idem                                                                              | —                                                                        |
| SegmentedControl origine                      | Tap segment                              | Set `origin`                                                                                  | Idem                                                                              | —                                                                        |
| Input P.IVA                                   | Digita                                   | Update state `vat`                                                                            | Idem                                                                              | —                                                                        |
| Recupera P.IVA                                | Click bottone "Recupera"                 | `fetchCompanyDataByVat(vat)` → merge su companyName + dipendenti                              | Chiamata Edge fn reale — gated da quota/day                                       | `!vat.trim()                                                             |     | loading` |
| Select settore                                | `<select>`                               | Setta settore                                                                                 | Idem                                                                              | —                                                                        |
| Select dipendenti / fatturato                 | `<select>` range                         | Setta range                                                                                   | Idem                                                                              | —                                                                        |
| Aggiungi interlocutore                        | Bottone tratteggiato                     | `addContact()` push in state                                                                  | Idem                                                                              | —                                                                        |
| Rimuovi interlocutore                         | Icona Trash                              | `removeContact(tempId)`                                                                       | Idem                                                                              | `contacts.length > 1`                                                    |
| Radio "Contatto primario"                     | Radio                                    | `setPrimaryContact(tempId)`                                                                   | Idem                                                                              | —                                                                        |
| Checkbox "Decision maker"                     | Checkbox                                 | Update flag                                                                                   | Idem                                                                              | —                                                                        |
| SegmentedControl urgenza                      | Tap segment                              | Set urgency                                                                                   | Idem                                                                              | —                                                                        |
| Chip servizi                                  | Tap chip                                 | Toggle Set<ServiceId>                                                                         | Idem                                                                              | —                                                                        |
| Input budget                                  | Digita                                   | Number state                                                                                  | Idem                                                                              | —                                                                        |
| Select tipo preventivo                        | Select                                   | Set QuoteType                                                                                 | Idem                                                                              | —                                                                        |
| Toggle "Dettagli posizione"                   | Header con chevron                       | `showAdvanced` toggle                                                                         | Idem                                                                              | —                                                                        |
| Segmented livello / assunzione / retribuzione | Tap                                      | Set state                                                                                     | Idem                                                                              | —                                                                        |
| Input zona / raggio / mansioni                | Digita                                   | Set state                                                                                     | Idem                                                                              | —                                                                        |
| Select sales assegnato                        | `<select>`                               | Set `assignedTo`                                                                              | Idem. Aggiungere guard: solo director puo' assegnare ad altri                     | `isDirector`                                                             |
| Textarea note                                 | Digita                                   | `leadNote`                                                                                    | Idem                                                                              | —                                                                        |
| Crea lead                                     | Button "Crea lead"                       | Push company/contacts/phones/emails/lead in mock arrays + navigate a `/sales/lead/:newLeadId` | INSERT transazionale su 5 tabelle + return leadId + Realtime broadcast + navigate | Validazione: companyName + figuraCercata + almeno un contatto con mobile |
| Annulla sheet                                 | Button "Annulla" o click fuori           | `resetAll()` + onClose                                                                        | Idem con conferma se draft non salvato                                            | —                                                                        |

---

## 5. Componenti chiave

- [LeadRow](../../../src/components/sales/LeadRow.tsx) — riga lead con swipe actions, macro chip, badge stale
- [EmptyState](../../../src/components/sales/EmptyState.tsx) — stato vuoto con CTA reset
- [SegmentedControl](../../../src/components/ios/IOSPrimitives.tsx) — tab segmentate macro-stage + badge count
- [BottomSheet](../../../src/components/ios/IOSPrimitives.tsx) — sheet dello sheet nuovo lead (size lg)
- [IOSButton](../../../src/components/ios/IOSPrimitives.tsx) — bottoni form (tinted/filled)
- `SearchInput` (inline) — input ricerca con X
- `FilterChip` (inline) — chip toggle con badge count
- `NewLeadSheet` (inline) — form a 6 sezioni (Origine, Azienda, Interlocutori, Richiesta, Dettagli posizione opz., Assegnazione)
- `SectionHeader` (inline) — header numerato sezione
- `Field` (inline) — wrapper label+hint+required

---

## 6. Integrazioni future

- **Supabase tables**: `sales.leads`, `core.companies`, `core.contacts`, `core.contact_phones`, `core.contact_emails`, `sales.leads_services` (M:N per multi-service)
- **RLS policy**: INSERT solo se `auth.uid()` e' sales attivo; `assigned_to` deve essere se stesso oppure chiamante e' `pod_director`
- **Realtime channel**: `sales:lead_created:<pod_id>` broadcast su creazione (director e setter subscribed)
- **Edge function**:
  - `lookup_vat(vat_number)` → proxy a Cribis/InfoCamere con cache 30gg
  - `create_lead_with_company(payload)` → transazione atomica multi-tabella con return leadId
- **n8n workflow**: su creazione lead → assegna automaticamente in base a load-balancing, crea draft email di presentazione
- **AI call**:
  - Auto-suggest "tipo preventivo" in base a headcount + urgency + sector
  - "Analizza P.IVA" per dire "azienda in crescita, buon target" dopo lookup
- **Analytics event**: `lead_list_viewed`, `lead_list_filter`, `new_lead_opened`, `new_lead_vat_lookup`, `new_lead_created` (con property origin/urgency/services)

---

## 7. UX/UI review

### Funziona bene

- SegmentedControl con badge count per macro-stage = navigazione chiara con feedback quantitativo
- 3 chip filtri con count (urgent / stale / mine) = comportamento scansionabile
- FAB fisso (bottom-right) = standard mobile, scoperta immediata
- Sheet "Nuovo lead" a 6 sezioni numerate con icone = percezione di progresso e completezza
- P.IVA lookup con auto-compilazione dei campi vuoti = riduce data entry drastica
- Gestione multi-interlocutore con primary + decision maker chip = modello realistico (non 1-to-1)
- Sticky action bar "Annulla / Crea lead" in fondo allo sheet = sempre accessibile anche su form lungo
- Toggle "Dettagli posizione" opzionale = non sovraccarica chi vuole creare lead rapida
- Validazione inline con banner rosso evidente

### Attriti UX attuali

- FAB a `bottom-20 right-5` si sovrappone al QuickNoteFAB della layout? (TBD — verificare z-index, entrambi sono `fixed`)
- Il sales che non ha ancora lavorato una lead non ha indicazione visiva tra "nuova" e "giorni fa" fuori dalla LeadRow
- La select "Sales assegnato" e' visibile a tutti: un sales regular non dovrebbe poter assegnare ad altri, questa leva va ruolo-gated
- Il form non salva come draft — se l'utente chiude per sbaglio perde tutto
- "Annulla" reset immediato senza conferma → distruttivo
- Nessuna indicazione "X campi obbligatori" in cima al form → il validate fa scroll manuale
- Il badge "+N altri" nei filtri count e' piccolo, non suggerisce che e' cliccabile
- "+Aggiungi interlocutore" tratteggiato e' corretto ma la riga aggiunta e' visivamente identica alla prima → non chiaro che il primo e' il primary di default
- Input P.IVA italiane accetta qualsiasi formato — no validazione checksum; rischio lookup fallato
- Lookup restituisce sempre mock random (anche con P.IVA valida) — useful per demo, ingannevole per testing con PO
- Il segmento "Tutte" non mostra `closed` — ok per default, ma non c'e' via per vederle qui (solo in Pipeline) → decisione intenzionale ma non esplicitata

### Coerenza UI

- Tipografia: ok — stesso stile SegmentedControl / Chip / Input del resto
- Spacing: ok — `space-y-7` tra sezioni ben respirate
- Colore: ok — teal primario per progressi, red per errori, orange per decision maker (chiaramente diverso da teal primary)
- Mobile-first: ok — form sheet scale bene; action bar sticky corretta con `env(safe-area-inset-bottom)`

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorita' | Tipo | Proposta                                                                                                                     | Effort |
| --- | --------- | ---- | ---------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0        | UX   | Confirm modal su "Annulla" se form ha campi compilati (evita perdita dati)                                                   | S      |
| 2   | P0        | UX   | Validazione P.IVA italiana (lunghezza 11 + checksum) prima di abilitare Recupera                                             | S      |
| 3   | P0        | FN   | Gate "Sales assegnato" select: solo director/admin vedono la select, sales regular auto-assegnato a se stesso                | S      |
| 4   | P1        | UX   | Auto-save draft in localStorage + ripristina su apertura successiva                                                          | M      |
| 5   | P1        | UI   | Contatore "X di 6 sezioni complete" in cima allo sheet per far percepire progresso                                           | M      |
| 6   | P1        | FN   | Rilevare duplicato company: check su P.IVA o nome prima di creare → suggerisci "Esiste gia', apri?"                          | M      |
| 7   | P2        | UX   | Long-press su una lead row per quick actions (stesso pattern della Pipeline) invece di dover aprire detail                   | M      |
| 8   | P2        | UI   | Separare visivamente "Contatto #1 (primario)" con background teal-soft sempre, non solo ring — e "Contatto #2" con bg neutro | S      |
| 9   | P2        | UX   | Filter "Solo chiuse" opzionale o view toggle per admin — oggi si deve andare in Pipeline solo per vederle                    | M      |
| 10  | P3        | AI   | Preview AI "Sintesi lead" sotto il titolo dopo aver compilato Azienda + Richiesta                                            | L      |
| 11  | P3        | UI   | Sostituire FAB + icona con bottone esteso "Nuovo lead" su breakpoint tablet/desktop (piu' scopribile)                        | S      |

Legenda priorita': P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- La lista mostra tutto (anche non-mie) di default: un sales regular dovrebbe vedere solo le sue di default con chip "Solo mie" ON pre-selezionato?
- Il chip "Solo mie" per un sales vale "assegnate a me"; per un director invece dovrebbe essere "nel mio POD" o "di cui sono closer"?
- Servizio `performance_marketing` e' sales-facing o solo admin? Oggi sempre visibile
- Quando la P.IVA lookup restituisce dati, dovremmo auto-compilare anche settore/fatturato/sede o chiedere conferma?
- L'auto-set `followUpFrequencyDays` (1/2/3) basato su urgency e' cablato — il PO vuole settarlo manuale?
- Il `NewLeadSheet` permette creazione senza P.IVA — vogliamo renderla obbligatoria per le aziende business?
- Dopo creazione si naviga a `/sales/lead/:id` — il PO preferirebbe restare sulla lista mostrando la nuova in cima con un toast "Lead creata"?
