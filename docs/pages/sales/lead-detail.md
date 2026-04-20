# Sales Lead Detail — Dettaglio lead / azienda a 4 viste

**Route**:

- `/sales/lead/:id` (main)
- `/sales/lead/:id/azienda`
- `/sales/lead/:id/preventivi`
- `/sales/lead/:id/storico`

**File**: [src/pages/sales/SalesLeadDetail.tsx](../../../src/pages/sales/SalesLeadDetail.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: tutti — ogni sales puo' aprire le lead che vede (scrittura gated da owner/closer/director)

---

## 1. Overview

Pagina di dettaglio multi-view che ruota intorno all'azienda piu' che alla singola richiesta: la "main" mostra l'hero azienda + assegnazione (setter/closer) + 3 card preview (Azienda, Preventivi, Storico). Le sub-route aprono la vista focalizzata corrispondente. E' il vero workspace del sales — da qui partono tutte le mutazioni (chiamate, note, cambio stato, creazione richieste, upload contratto, assegnazioni setter/closer, lookup P.IVA). Un FAB AI Copilot in basso a destra apre un bottom sheet con prompt preset sul contesto corrente.

---

## 2. Viste & Stati

| ID vista           | Trigger                                                         | Descrizione                                                            | Componenti rilevanti                               |
| ------------------ | --------------------------------------------------------------- | ---------------------------------------------------------------------- | -------------------------------------------------- | ---------------------- | ---------- |
| main               | `/sales/lead/:id`                                               | Hero + Assignment + Status azienda + 3 cards overview                  | `LeadHeroBlock`, `AssignmentBlock`, `MainOverview` |
| azienda            | `/sales/lead/:id/azienda`                                       | `CompanyInfoPanel` completo (anagrafica, multi-contatti, P.IVA lookup) | `CompanyInfoPanel`                                 |
| preventivi         | `/sales/lead/:id/preventivi`                                    | Lista `RequestCard` espandibili + duplica + nuovo                      | `TabPreventiviContent`, `RequestCard`              |
| storico            | `/sales/lead/:id/storico`                                       | `NotesTimeline` full con filtro per richiesta                          | `NotesTimeline`                                    |
| lead_not_found     | `!lead                                                          |                                                                        | !company`                                          | Redirect `/sales/lead` | `Navigate` |
| advance_sheet      | Click "Avanza stato"                                            | BottomSheet con 3 opzioni (next / postponed / discarded) + nota        | `AdvanceStageSheet`                                |
| ai_copilot_sheet   | Tap FAB sparkles                                                | BottomSheet con 4 prompt preset + contesto corrente                    | `BottomSheet`                                      |
| assign_menu_setter | Click setter                                                    | Dropdown lista sales                                                   | Avatar + Check                                     |
| assign_menu_closer | Click closer                                                    | Dropdown lista sales + "Nessuno"                                       | Avatar + Check                                     |
| duplicate_menu     | Click "Duplica" in preventivi                                   | Dropdown selezione richiesta sorgente                                  | —                                                  |
| stale_banner       | `lead.isStale` true in preventivi                               | Banner rosso "Ferma da N giorni"                                       | —                                                  |
| toast              | Qualsiasi mutazione                                             | Toast ink/orange/red bottom center 1.5s                                | `useToast` inline                                  |
| no_contacts        | Azienda senza contatti                                          | CTA tratteggiato "+ Aggiungi contatto"                                 | —                                                  |
| no_requests        | Company senza lead (impossibile arrivando da detail ma gestito) | `EmptyState` "Ancora nessuna richiesta"                                | `EmptyState`                                       |
| no_notes           | `companyNotes.length === 0`                                     | Stato vuoto in NotesTimeline                                           | `NotesTimeline`                                    |
| vat_lookup_loading | `recuperandoDati` true                                          | Spinner nel pannello azienda                                           | —                                                  |
| loading_initial    | Non gestito — dati sincroni                                     | —                                                                      | —                                                  |
| error              | Non gestito (try/catch solo su lookup)                          | —                                                                      | —                                                  |

---

## 3. Dati & Sorgenti

| Entita' UI                       | Mock (src/data/mock/sales.ts)                                               | Schema reale                                                  | Note                                                                                                |
| -------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Lead corrente                    | `getLeadById(id)`                                                           | `sales.leads`                                                 | `id` da `useParams`                                                                                 |
| Azienda                          | `getCompanyById(lead.companyId)`                                            | `core.companies`                                              | Mostra `ragioneSociale ?? businessName`                                                             |
| Tutte le richieste della company | `getLeadsByCompanyId(company.id)` ordinato per `lastActivityAt` desc        | `sales.leads where company_id=X`                              | Una company puo' avere N richieste                                                                  |
| Contatti company                 | `getContactsByCompanyId(company.id)`                                        | `core.contacts where company_id=X`                            | Gestione primary + decision maker + multi-telefono/email                                            |
| Telefoni multipli                | `contact.phones[]`                                                          | `core.contact_phones`                                         | `mobile`, `landline`, `internal_extension`, `fax`, `other`                                          |
| Email multipli                   | `contact.emails[]`                                                          | `core.contact_emails`                                         | `isPrimary` flag                                                                                    |
| Note storico                     | `getNotesByCompanyId(company.id)` filtrate per richiesta se `openRequestId` | `sales.lead_notes` + `sales.company_notes` (TBD unificazione) | Channel: `internal`, `system_*`, `email`, `whatsapp`, `phone_note`                                  |
| Prossimo appt                    | `getNextAppointmentByLead(lead.id)`                                         | `sales.appointments` future scheduled                         | Pill amber nell'hero                                                                                |
| Preventivi per richiesta         | `getQuotesByLead(r.id)`                                                     | `sales.quotes`                                                | —                                                                                                   |
| Indagine mercato                 | `salesMarketInvestigations.filter(leadId=r.id)`                             | `sales.market_investigations`                                 | —                                                                                                   |
| Status azienda                   | `company.status` + `companyStatusLabel`                                     | `core.companies.status` enum `CompanyStatus`                  | 6 valori: nuova, in_trattativa, cliente_attivo, cliente_storico, amministrazione_in_corso, inattiva |
| Setter                           | `lead.assignedToUserId` → `getSalesUserById`                                | `sales.leads.assigned_to_user_id`                             | —                                                                                                   |
| Closer                           | `lead.closerUserId` → `getSalesUserById`                                    | `sales.leads.closer_user_id`                                  | Opzionale                                                                                           |
| Roster assignable                | `salesUsers.filter(role in [sales, pod_director, closer])`                  | `sales.user_profiles`                                         | Filtra admin                                                                                        |
| Lookup P.IVA                     | `fetchCompanyDataByVat(vat)`                                                | Cribis/InfoCamere via Edge fn (TBD)                           | Mock 1.2s                                                                                           |

---

## 4. Pulsanti & Azioni

### Navigazione & hero

| Azione                     | Trigger UI                    | Effetto attuale (mock)                           | Effetto target (produzione)                        | Gated da                |
| -------------------------- | ----------------------------- | ------------------------------------------------ | -------------------------------------------------- | ----------------------- |
| Back mobile                | IOSNavBar back                | Torna a `/sales/lead/:id` (se subview) o history | Idem                                               | View                    |
| Back desktop subview       | ChevronLeft header            | Navigate a `/sales/lead/:id`                     | Idem                                               | Non-main                |
| Apri Azienda               | Card overview / button status | Navigate a `/sales/lead/:id/azienda`             | Idem                                               | —                       |
| Apri Preventivi            | Card overview                 | Navigate a `/sales/lead/:id/preventivi`          | Idem                                               | —                       |
| Apri Storico               | Card overview                 | Navigate a `/sales/lead/:id/storico`             | Idem                                               | —                       |
| Chiama                     | Tel button in hero            | `tel:<clean>` link nativo                        | Idem + log system note `channel=phone_note`        | `primaryPhone` presente |
| WhatsApp                   | WhatsApp button in hero       | Apre `https://wa.me/<digits>` new tab            | Idem + log                                         | `primaryPhone` presente |
| Email                      | Email button in hero          | `mailto:<email>`                                 | Idem + log                                         | `primaryEmail` presente |
| Aggiungi contatto          | CTA tratteggiata in hero      | Navigate `/sales/lead/:id/azienda`               | Idem                                               | Nessun primary          |
| Apri AI Copilot            | FAB sparkles                  | Apre BottomSheet AI                              | Idem + trigger LLM call                            | —                       |
| AI prompt click            | Bottone prompt nel sheet      | Toast demo + chiude sheet                        | Chiamata LLM con contesto (lead + company + notes) | —                       |
| "Apri AI Copilot completo" | Bottone in fondo sheet        | Navigate `/sales/ai-copilot`                     | Idem                                               | —                       |

### Mutazioni (stato, note, preventivi, assegnazioni)

| Azione                      | Trigger UI                    | Effetto attuale (mock)                                                          | Effetto target (produzione)                                            | Gated da                   |
| --------------------------- | ----------------------------- | ------------------------------------------------------------------------------- | ---------------------------------------------------------------------- | -------------------------- |
| Avanza stato                | Button "Avanza stato"         | Apre `AdvanceStageSheet`                                                        | Idem                                                                   | —                          |
| Conferma avanzamento        | Button "Conferma" in sheet    | `updateRequest` + 2 `addLeadNote` (system_status + internal nota)               | Edge fn `advance_lead_stage` con audit + Realtime broadcast            | `selected != null`         |
| Set setter                  | Click in dropdown             | `updateRequest({assignedToUserId})` + toast                                     | UPDATE + notifica push al nuovo setter                                 | Director/Admin             |
| Set closer                  | Click in dropdown             | `updateRequest({closerUserId})` + toast                                         | Idem                                                                   | Director/Admin             |
| Rimuovi closer              | Click "Nessuno"               | `updateRequest({closerUserId: undefined})`                                      | Idem                                                                   | Director                   |
| Nuova richiesta             | Button "Nuovo" in preventivi  | Push in `salesLeads` + switch tab preventivi + toast                            | INSERT `sales.leads` con `status='new'`, return id                     | —                          |
| Duplica richiesta           | Click in dropdown duplica     | Clone record con nuovo id + status new + toast                                  | INSERT con fk originale + audit `duplicated_from`                      | Almeno una richiesta       |
| Upload contratto            | File input dentro RequestCard | `URL.createObjectURL(f)` + `updateRequest({contrattoFirmatoUrl})` + system note | Upload a Supabase Storage `sales-contracts/<lead_id>/` + public URL    | Stage `awaiting_signature` |
| Richiedi indagine           | Button in RequestCard         | Set `indagineUrl` placeholder + system_survey note                              | Edge fn `request_market_investigation(lead_id)` → task in queue per HR | Stage `qualified`          |
| Crea preventivo             | Button in RequestCard         | Toast "TODO"                                                                    | Apertura modale `QuotesBuilder`                                        | —                          |
| Advance da card             | Per ogni richiesta → "Avanza" | Apre AdvanceStageSheet con target richiesta                                     | Idem                                                                   | —                          |
| Toggle espansione richiesta | Header RequestCard            | `setOpenRequestId` toggle                                                       | Idem                                                                   | —                          |

### CompanyInfoPanel (view Azienda)

| Azione                             | Effetto attuale                                               | Effetto target                                     |
| ---------------------------------- | ------------------------------------------------------------- | -------------------------------------------------- |
| Edit azienda (inline)              | `updateCompany(patch)`                                        | UPDATE `core.companies`                            |
| Cambia status azienda              | `updateCompanyStatus`                                         | UPDATE con check permission                        |
| Recupera dati P.IVA                | `fetchCompanyDataByVat` + merge                               | Edge fn reale                                      |
| Aggiungi contatto                  | `handleAddContact` push                                       | INSERT + select                                    |
| Edit contatto                      | `updateContact`                                               | UPDATE                                             |
| Elimina contatto                   | `handleDeleteContact` splice                                  | Soft-delete + RLS check (non se ha note associate) |
| Imposta primary                    | `handleMakePrimaryContact`                                    | UPDATE `is_primary` transazionale                  |
| Add/Update/Delete/SetPrimary phone | 4 handler                                                     | CRUD `core.contact_phones`                         |
| Add/Update/Delete/SetPrimary email | 4 handler                                                     | CRUD `core.contact_emails`                         |
| Apri richiesta da contatto         | `onOpenRequest(rid)` → setOpenRequestId + navigate preventivi | Idem                                               |

### NotesTimeline (view Storico)

| Azione               | Effetto attuale          | Effetto target                      |
| -------------------- | ------------------------ | ----------------------------------- |
| Aggiungi nota        | `addLeadNote` + toast    | INSERT `sales.lead_notes`           |
| Elimina nota         | `deleteLeadNote` + toast | Soft-delete (solo owner o director) |
| Filtra per richiesta | `setOpenRequestId`       | Query param                         |
| Clear filtro         | `onClearFilter`          | Idem                                |

---

## 5. Componenti chiave

- [CompanyInfoPanel](../../../src/components/sales/CompanyInfoPanel.tsx) — pannello completo azienda + multi-contatti + P.IVA lookup (969 righe, probabilmente da spezzare)
- [RequestCard](../../../src/components/sales/RequestCard.tsx) — card espandibile per singola richiesta con stage actions e upload (1025 righe)
- [NotesTimeline](../../../src/components/sales/NotesTimeline.tsx) — timeline note filtrabile con composer
- [MacroStageChip](../../../src/components/sales/LeadRow.tsx) — chip colorato stato
- [EmptyState](../../../src/components/sales/EmptyState.tsx) — stato vuoto
- [Avatar](../../../src/components/common/Avatar.tsx) — avatar user con iniziali
- [IOSNavBar](../../../src/components/ios/IOSNavBar.tsx) — navbar mobile con back
- [BottomSheet](../../../src/components/ios/IOSPrimitives.tsx) — sheet per Advance + AI
- [IOSButton](../../../src/components/ios/IOSPrimitives.tsx) — azioni primarie
- `LeadHeroBlock` (inline) — hero azienda con CTA chiama/whatsapp/email + pill appt
- `MainOverview` (inline) — 3 OverviewCard (Azienda / Preventivi / Storico) con preview
- `OverviewCard` (inline) — card grande con footer CTA colorato
- `AssignmentBlock` (inline) — setter/closer con dropdown inline
- `TabPreventiviContent` (inline) — contenuto tab preventivi con header + lista RequestCard
- `AdvanceStageSheet` (inline) — sheet avanzamento stato con opzioni + nota
- `useToast` (inline) — hook toast semplice

---

## 6. Integrazioni future

- **Supabase tables**: `sales.leads`, `core.companies`, `core.contacts`, `core.contact_phones`, `core.contact_emails`, `sales.lead_notes`, `sales.appointments`, `sales.quotes`, `sales.market_investigations`, `sales.lead_relations`
- **Storage bucket**: `sales-contracts` (upload contratto firmato), `sales-quotes` (PDF preventivi)
- **RLS policy**:
  - SELECT: owner o closer o pod membership o admin
  - UPDATE status: owner, closer, director
  - DELETE nota: solo autore (+7gg) o director
  - UPDATE assigned_to: solo director/admin
- **Realtime channel**: `sales:lead:<lead_id>` per collaborative editing (status + note real-time)
- **Edge function**:
  - `advance_lead_stage(lead_id, next_status, note?)` transazione con audit
  - `lookup_vat(vat_number)` con cache
  - `upload_contract(lead_id, file)` con signed URL + system note
  - `request_market_investigation(lead_id)` crea task in recruiting
- **n8n workflow**:
  - On status change → `awaiting_signature`: genera draft contratto auto
  - On `customer`: crea onboarding board + welcome email
  - On `discarded`: log motivo + analytics funnel
- **AI call**:
  - Riassumi ultimo mese di attivita'
  - Genera email follow-up da contesto
  - Suggerisci prossima azione ottimale
  - Analizza obiezioni emerse
  - Oggi il prompt fa solo toast demo — da cablare su `/sales/ai-copilot` con contesto serialized
- **Analytics event**: `lead_detail_viewed`, `lead_view_tab_change`, `lead_call_click`, `lead_whatsapp_click`, `lead_email_click`, `lead_advance_open`, `lead_advance_confirm` (with from/to), `lead_ai_prompt`, `lead_assign_setter/closer`, `lead_upload_contract`

---

## 7. UX/UI review

### Funziona bene

- Split 4-view con URL persistente (`/azienda`, `/preventivi`, `/storico`) = deep-link condivisibile + back button nativo coerente
- Hero azienda concentra le 3 azioni comunicative primarie (Chiama/WhatsApp/Email) a un tap = sales operativo subito
- MainOverview con preview dei dati di ogni card (ultimo evento, numero richieste, quote inviate) = evita tab-surf per vedere lo stato
- Stale banner rosso nella view preventivi quando la richiesta e' ferma = segnale forte
- Assignment block con dropdown inline + avatar + check = velocissimo, zero modali per un'azione frequente
- AdvanceStageSheet con le 3 opzioni (next / postponed / discarded) + nota opzionale = funnel semplice
- AI Copilot FAB con contesto (azienda + n richieste + n contatti + n eventi) = prompt AI pre-qualificati
- Toast discreti con tone ok/warn/err + auto-dismiss 1.5s
- Scroll unico centrato `max-w-[720px]` anche desktop = nessuna colonna sinistra morta, mobile-first rispettato
- Gestione molteplice di telefoni/email per contatto = modello dati realistico

### Attriti UX attuali

- 4 view navigabili solo via card → da mobile non c'e' tab bar orizzontale per switch rapido (devi fare back + card)
- `handleRecuperaDati` non gestisce errori specifici (rete, rate limit) — catch generico "Errore nel recupero"
- `updateCompany` scrive direttamente sull'oggetto (`Object.assign(company, patch)`) = mutation del mock shared; in produzione con ottimistic UI servira' riconciliazione
- `rerender()` manuale via `setTick` sparso ovunque = indica che lo stato andra' rivisto con Zustand/TanStack Query
- Nessuna gestione conflitti real-time: se il setter cambia stato mentre io guardo, non vedo il diff
- AdvanceStageSheet non mostra implicazioni (es. "passando a proposal_sent verra' creato preventivo draft?") — black box
- FAB AI si sovrappone visivamente al QuickNoteFAB di SalesLayout (entrambi `fixed bottom-20 right-5`) = collisione z-index/posizione (TBD verificare)
- "Aggiungi contatto" su hero porta a Azienda view → ok, ma l'utente non capisce che il nuovo contatto apparira' espanso (scrolla giu' a cercarlo)
- `handleNewRequest` crea lead con `positionTitle: "Nuova richiesta"` e `figuraCercata: "Nuova figura"` = placeholder testuale in DB da rinominare subito — l'utente finisce su una richiesta a nome inventato
- Toast 1.5s troppo breve per conferme critiche (upload contratto, advance stage)
- Nessuna indicazione "loading" durante `upload contratto` o `richiedi indagine` (gia' istantanei nel mock, ma in prod serviranno)
- L'`openRequestId` state e' locale: passando da view Preventivi a Storico il filtro si mantiene, ma non c'e' feedback "stai vedendo filtrato per X"
- La label "Nuova richiesta" non riflette il linguaggio UX del form "Nuovo lead / cliente" — due modalita' create lead (FAB lista vs inline) con esperienze molto diverse

### Coerenza UI

- Tipografia: ok — serif per hero title, sans UI consistente
- Spacing: ok — `space-y-4` nella main, consistent padding nelle cards
- Colore: ok — accent indigo/amber/slate per overview, teal per primary actions, amber per appt prossimo, red per stale/errori
- Mobile-first: ok — colonna unica 720px, IOSNavBar solo mobile, desktop sub-header inline

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorita' | Tipo | Proposta                                                                                                                       | Effort |
| --- | --------- | ---- | ------------------------------------------------------------------------------------------------------------------------------ | ------ |
| 1   | P0        | UX   | Aggiungere tab bar orizzontale (main/Azienda/Preventivi/Storico) sticky sotto NavBar mobile per switch rapido — oggi solo card | M      |
| 2   | P0        | UX   | `handleNewRequest` deve aprire un form inline minimo (figura + headcount + urgency) invece di creare record placeholder        | M      |
| 3   | P0        | FN   | Gate write actions per role: sales regular non deve poter cambiare setter/closer se non e' owner/director                      | S      |
| 4   | P1        | UX   | Preview conseguenze in AdvanceStageSheet ("Passando a proposal_sent verra' generato un draft preventivo")                      | M      |
| 5   | P1        | UI   | Toast critici (upload, advance, delete) con durata 3s + icona tono esplicita + ARIA live                                       | S      |
| 6   | P1        | FN   | Indicatore filtro attivo in Storico ("Stai vedendo solo richiesta X · Clear")                                                  | S      |
| 7   | P1        | UX   | Coordinamento FAB AI + QuickNoteFAB: uno dei due va spostato o unificati in un menu                                            | M      |
| 8   | P2        | UI   | Animazione transizione view (slide) tra main ↔ subview per coerenza iOS                                                        | M      |
| 9   | P2        | FN   | Ottimistic UI + rollback su errore per mutazioni (oggi mutano direttamente)                                                    | L      |
| 10  | P2        | UX   | Breadcrumb desktop: "Lead / Azienda" → oggi solo chevron + titolo                                                              | S      |
| 11  | P2        | UX   | Confirm dialog per azioni distruttive: `handleDeleteContact`, `deleteLeadNote` — oggi immediato                                | S      |
| 12  | P3        | AI   | Pre-generare il prompt AI (non solo suggerire) con "last activity summary" visibile sopra il FAB                               | L      |
| 13  | P3        | UI   | In MainOverview: aggiungere KPI sintetico "€ totale in proposta" nella card Preventivi                                         | S      |

Legenda priorita': P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- La view "main" mostra una sola lead (prima passata come `:id`) ma le 3 card mostrano aggregato COMPANY: il PO vuole che "main" sia orientato all'azienda (come ora) o alla singola richiesta `:id`?
- Se una company ha 5 richieste, quale vediamo come "richiesta corrente" nell'hero e perche'? (oggi `:id` dalla URL)
- Cambio setter su UNA richiesta o su tutte le richieste company? (oggi solo su `lead` corrente: semantica poco chiara per l'utente che crede di cambiare "l'account manager")
- "Indagine richiesta" setta `indagineUrl` placeholder — va chiarito workflow: chi la scrive (HR)? Quanto dura? Dove vede il sales il risultato?
- Contratto firmato: storage bucket public o signed URL? Chi lo vede (client portal?)
- AI Copilot contextual prompt — quali dati includere (note, preventivi, contatti, competitor)? Privacy/DPA?
- `amministrazione_in_corso` compare come macro-stage separato in Pipeline ma qui e' status nascosto — quando/come si passa li'? Oggi non c'e' UI action visibile
- Per una company con status `cliente_storico` (ex cliente) ha senso poter creare nuove richieste con `status=new`? Workflow di riattivazione da definire
- Le note company-level (senza leadId) vs note per richiesta: l'utente in Storico le distingue? Oggi filtro c'e' ma rischio di confondere
- 2 FAB fissi (QuickNoteFAB layout + AI Copilot pagina) — quale ha priorita' visiva sui tap al margine?
