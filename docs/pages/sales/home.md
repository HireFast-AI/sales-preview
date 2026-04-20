# Sales Home — "Oggi"

**Route**: `/sales` (index)
**File**: [src/pages/sales/SalesHome.tsx](../../../src/pages/sales/SalesHome.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: tutti (sales, closer, pod_director, admin)

---

## 1. Overview

Dashboard operativa "Oggi" del sales: punto di ingresso giornaliero che concentra in un'unica pagina mobile-first (1) l'hero con l'azione piu' urgente del giorno, (2) la todo-list delle 5 attivita' prioritarie, (3) l'agenda appuntamenti, (4) lo snapshot pipeline a 4 colonne e (5) le lead ferme. Rimpiazza l'uso del CRM a tab — il sales apre questa pagina la mattina, sa cosa fare per i prossimi 30 minuti e quali lead stanno rallentando.

---

## 2. Viste & Stati

| ID vista       | Trigger                                | Descrizione                                  | Componenti rilevanti                            |
| -------------- | -------------------------------------- | -------------------------------------------- | ----------------------------------------------- |
| default        | Utente con pipeline attiva             | Hero + todo + agenda + snapshot + lead ferme | `HeroAI`, `TaskRow`, `GroupedList`, `StageCard` |
| hero_overdue   | `getOverdueAppointments().length > 0`  | Hero arancio "Aggiorna N esiti scaduti"      | `HeroAI` variant dark                           |
| hero_new_leads | Nessun overdue, lead assegnate oggi    | Hero "N nuove lead da lavorare"              | `HeroAI`                                        |
| hero_stale     | Nessun overdue/new, presenza stale     | Hero "N lead da risvegliare"                 | `HeroAI`                                        |
| hero_clear     | Tutto pulito                           | Hero "Tutto sotto controllo" + CTA Pipeline  | `HeroAI` variant primary                        |
| no_agenda      | `getAppointmentsToday().length === 0`  | Sezione "Agenda oggi" non renderizzata       | —                                               |
| no_stale       | `getStaleLeads().length === 0`         | `EmptyState` positivo "Nessuna lead ferma"   | `EmptyState` tone=positive                      |
| empty_todo     | Nessuna delle 5 condizioni soddisfatta | Sezione "Da fare adesso" nascosta            | —                                               |
| loading        | Non gestito — dati sincroni da mock    | —                                            | —                                               |
| error          | Non gestito                            | —                                            | —                                               |

---

## 3. Dati & Sorgenti

| Entita' UI           | Mock (src/data/mock/sales.ts)                  | Schema reale                                                                 | Note                                                  |
| -------------------- | ---------------------------------------------- | ---------------------------------------------------------------------------- | ----------------------------------------------------- |
| Utente corrente      | `getSalesUserById(CURRENT_SALES_USER_ID)`      | `core.users` + `sales.user_profiles` (TBD)                                   | Hardcoded a `u-gab-001`                               |
| Saluto / data        | Stringa "Sabato 18 aprile" hardcoded           | —                                                                            | Da sostituire con `toLocaleDateString`                |
| Appuntamenti oggi    | `getAppointmentsToday()` (prefix `2026-04-18`) | `sales.appointments` filtro `scheduled_at::date = current_date`              | Cutoff "now" hardcoded a `2026-04-18T10:30`           |
| Appuntamenti overdue | `getOverdueAppointments()`                     | `sales.appointments` where `status='scheduled' AND scheduled_at < now()`     | —                                                     |
| Nuove lead oggi      | `getLeadsAssignedToday()`                      | `sales.leads` where `assigned_at::date = current_date AND assigned_to = :me` | Non filtra per owner                                  |
| Stale leads          | `getStaleLeads()`                              | `sales.leads` where `is_stale = true` (view `v_stale_leads` TBD)             | Computed via `lastActivityAt + followUpFrequencyDays` |
| Count pipeline       | `getLeadsByStatuses([...])`                    | `sales.leads.status`                                                         | 4 macro-stage aggregati                               |
| Azienda associata    | `getCompanyById(companyId)`                    | `core.companies`                                                             | —                                                     |
| Contatto agenda      | `getContactById(contactId)`                    | `core.contacts`                                                              | —                                                     |

---

## 4. Pulsanti & Azioni

| Azione                                   | Trigger UI                    | Effetto attuale (mock)                                                        | Effetto target (produzione)                           | Gated da   |
| ---------------------------------------- | ----------------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------- | ---------- |
| Refresh                                  | Pulsante "Aggiorna" in NavBar | `aria-label` solo — nessun handler                                            | Invalida query SWR/TanStack e ripolla viste           | —          |
| Hero CTA                                 | Click `hero.ctaTo`            | `<Link>` a `/sales/appuntamenti` o `/sales/lead?filter=X` o `/sales/pipeline` | Idem + analytics `home_hero_click`                    | Stato hero |
| Hero "Vedi tutto"                        | Link sotto CTA                | `<Link>` a `/sales/lead`                                                      | Idem                                                  | —          |
| TaskRow "Esito" (overdue)                | Bottone inline                | Navigate a `/sales/lead/:id` (dettaglio)                                      | Aprire direttamente il form outcome dell'appuntamento | —          |
| TaskRow "Chiama"                         | Bottone nuova lead            | Navigate a `/sales/lead/:id`                                                  | `tel:` link diretto + log attivita' automatico        | —          |
| TaskRow "Apri" (stale / appt / proposal) | Bottone                       | Navigate a `/sales/lead/:id`                                                  | Idem + marcare lead come "touched"                    | —          |
| "Agenda oggi" → Tutti                    | NavLink                       | `/sales/appuntamenti`                                                         | Idem con filtro `?date=today`                         | —          |
| AgendaList item                          | Click su riga agenda          | Navigate a `/sales/lead/:leadId`                                              | Stesso comportamento                                  | —          |
| StageCard                                | Click su card kpi             | Navigate a `/sales/pipeline?stage=X`                                          | Pre-filtra kanban. Query param gia' supportato? (TBD) | —          |
| "Lead ferme" item                        | Click su riga                 | Navigate a `/sales/lead/:id`                                                  | Idem                                                  | —          |
| Azioni rapide Conversazioni              | Click card                    | Navigate a `/sales/conversazioni`                                             | Idem                                                  | —          |
| Azioni rapide Copilot                    | Click card                    | Navigate a `/sales/ai-copilot`                                                | Idem                                                  | —          |

---

## 5. Componenti chiave

- [HeroAI](../../../src/components/ios/IOSPrimitives.tsx) — hero card scura/primaria con CTA
- [IOSNavBar](../../../src/components/ios/IOSNavBar.tsx) — top bar iOS con titolo grande + actions
- [GroupedList](../../../src/components/ios/GroupedList.tsx) — lista iOS-style per agenda e stale
- [IOSButton](../../../src/components/ios/IOSPrimitives.tsx) — pulsante iOS standardizzato
- [EmptyState](../../../src/components/sales/EmptyState.tsx) — stato vuoto positivo "Nessuna lead ferma"
- `StageCard` (inline) — card compatta per snapshot pipeline
- `TaskRow` (inline) — riga todo con icona colorata + CTA

---

## 6. Integrazioni future

- **Supabase tables**: `sales.leads`, `sales.appointments`, `core.companies`, `core.contacts`, `sales.user_profiles`
- **RLS policy**: filtro automatico per `assigned_to_user_id = auth.uid()` oppure membership `pod_team` per director
- **Realtime channel**: `sales:home:<user_id>` → aggiorna badge overdue/new/stale senza refresh
- **Edge function**: `compute_daily_home(user_id)` che restituisce tutti i conteggi in una sola call (evita N query lato client)
- **n8n workflow**: `daily_morning_push_8am` invia notifica push quando hero passa a stato "overdue"
- **AI call**: generare un summary del giorno stile "oggi hai 3 call calde e 2 offerte in attesa" via Claude/OpenAI
- **Analytics event**: `home_viewed`, `home_hero_click`, `home_task_action`, `home_stage_click` con property `stage`/`action_type`

---

## 7. UX/UI review

### Funziona bene

- Hero dinamico che cambia messaggio in base alla priorita' (overdue > new > stale > clear) — focus mentale efficace
- Un solo CTA primario per hero — niente paralisi da scelta
- TaskRow piccoli con CTA esplicite (Esito / Chiama / Apri) — mobile-first chiaro
- Snapshot 4-colonne con stessi macro-stage della Pipeline e della Lead list → consistenza cross-page
- EmptyState positivo quando non ci sono stale — rinforzo comportamentale

### Attriti UX attuali

- "Oggi / Sabato 18 aprile" hardcoded: demotiva la sensazione di app "viva"
- Pulsante Refresh nella NavBar non ha handler — click a vuoto, frustrazione
- TaskRow massimo 5 ma non c'e' "Vedi altre" se ce ne fossero 10+
- Tap su intera riga TaskRow porta alla lead ma il bottone sovrascrive — se l'utente tocca a sinistra dell'icona non succede nulla di diverso: gesture ambigua
- Nessun loading skeleton — quando arriveranno dati async ci sara' flash
- "Da fare adesso" mescola 5 tipologie diverse (overdue/new/stale/appt/proposal) senza raggruppare — difficile scansione
- Hero CTA secondaria "Vedi tutto" in bianco 80% su sfondo scuro ha contrasto basso

### Coerenza UI

- Tipografia: ok — font-serif per titoli, tracking coerente
- Spacing: ok — scale `space-y-6` consistente con il resto delle pagine sales
- Colore: ok — palette CSS vars rispettata; orange/teal/ink/red usati con significato coerente
- Mobile-first: ok — `max-w-[840px]` centrato, grid responsive sulle azioni rapide

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorita' | Tipo | Proposta                                                                                                                                                               | Effort |
| --- | --------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0        | UX   | Data/ora dinamiche (sostituire "Sabato 18 aprile" con `toLocaleDateString("it-IT", { weekday: "long", day: "numeric", month: "long" })` e time cutoff da `Date.now()`) | S      |
| 2   | P0        | UX   | Collegare bottone Refresh a un reload effettivo (ricalcolo delle 5 sorgenti) o rimuoverlo                                                                              | S      |
| 3   | P1        | UX   | Raggruppare TaskRow per tipologia (ribbon: "Urgenti / Nuove / Follow-up") quando sono >3                                                                               | M      |
| 4   | P1        | UI   | Skeleton loader per hero + todo + agenda quando si passera' ad API async                                                                                               | M      |
| 5   | P2        | UX   | Aggiungere pulsante "Vedi altre N" in fondo a todo-list se eccedono 5                                                                                                  | S      |
| 6   | P2        | UI   | Alzare contrasto "Vedi tutto" bianco sull'hero scuro (es. underline o outline)                                                                                         | S      |
| 7   | P2        | UX   | Click su StageCard dovrebbe scrollare il kanban alla colonna corrispondente (oggi solo query param, da verificare che Pipeline lo legga — TBD)                         | S      |
| 8   | P2        | FN   | Badge numerico mini anche su TaskRow quando ci sono N overdue/stale simili ("3 lead come questa")                                                                      | M      |
| 9   | P3        | UX   | Quick action "Nuova lead" in evidenza nella pagina Oggi (oggi si raggiunge solo da Lead → FAB)                                                                         | S      |
| 10  | P3        | AI   | Integrare un "daily summary" AI sotto l'hero: "Ieri hai chiuso 2, oggi hai 4 appuntamenti, obiettivo settimana 12k"                                                    | L      |

Legenda priorita': P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- L'orario di riferimento per overdue/new/stale deve essere il fuso del POD del sales o UTC? (demo usa `Europe/Rome` hardcoded)
- Quali task precedenza hanno quando coesistono (es. ho 5 overdue E 3 nuove lead): regola attuale mostra solo prime di ognuno, dovremmo aumentare?
- Snapshot pipeline: "Chiusi" conta customer+discarded+postponed — il PO vuole solo customer per metrica motivazionale?
- Azioni rapide in fondo (Conversazioni / Copilot): servono davvero qui o bastano nella sidebar?
- `getLeadsAssignedToday()` non filtra per `assignedToUserId` — bug o intenzionale (vista director)?
