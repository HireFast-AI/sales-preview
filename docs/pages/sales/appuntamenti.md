# Appuntamenti

**Route**: `/sales/appuntamenti`
**File**: [src/pages/sales/SalesAppuntamenti.tsx](../../../src/pages/sales/SalesAppuntamenti.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: tutti i sales (vedono solo propri — director/admin filtro team TBD)

---

## 1. Overview

Agenda del sales: setter call, closer call, video call e appuntamenti in presenza collegati a una lead. Due modalità (Lista e Calendario settimanale), FAB per creare al volo, bottom-sheet dettaglio con azioni (genera link video, registra esito, vai alla lead). È il cockpit operativo quotidiano del sales: oggi, domani, passati da chiudere.

---

## 2. Viste & Stati

| ID vista         | Trigger                                   | Descrizione                                                                   | Componenti rilevanti                                 |
| ---------------- | ----------------------------------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------- |
| list-today       | default / tab `Oggi`                      | Raggruppata per giorno (`Oggi · N`) con accent bar colorata per tipo          | `AppointmentList`, `AppointmentRow`                  |
| list-tomorrow    | tab `Domani`                              | Solo appuntamenti giorno successivo                                           | idem                                                 |
| list-week        | tab `Settimana`                           | Finestra next-7gg a partire da `SALES_NOW_ISO`                                | idem                                                 |
| list-overdue     | tab `Passati` o click banner overdue      | Scheduled < now, chip rosso "Esito mancante"                                  | idem                                                 |
| calendar-desktop | toggle `Calendario` su ≥md                | Grid 7col × 12h (8-20) con now-line rossa, blocchi colorati per tipo          | `CalendarView`, `WeekGrid`, `DayColumn`, `ApptBlock` |
| calendar-mobile  | toggle `Calendario` su <md                | Day picker orizzontale + `DayAgenda` verticale                                | `CalendarView`, `DayAgenda`, `AgendaCard`            |
| banner-overdue   | `overdueList.length > 0` && tab ≠ overdue | Banner rosso "Aggiorna N esiti" con CTA "Apri"                                | inline JSX                                           |
| sheet-detail     | click riga/blocco                         | BottomSheet size=md con header + quando/dove + videocall + esito + CTA azioni | `BottomSheet`, `AppointmentDetailSheet`              |
| sheet-new        | FAB `+` o click slot calendario           | BottomSheet con lead search + type + data/ora/durata + location + note        | `BottomSheet`, `NewAppointmentSheet`                 |
| empty            | `list.length === 0`                       | Copy contestuale per tab (Oggi vuoto = copy positiva per overdue)             | `EmptyState`                                         |
| video-empty      | `isVideo && !videoUrl`                    | Provider chooser (meet/zoom/teams/whereby) + button "Genera link"             | inline sheet                                         |
| video-ready      | `isVideo && videoUrl`                     | Url mono + 3 IOSButton (Avvia/Copia/Invia)                                    | inline sheet                                         |
| outcome-past     | `isPast` in sheet                         | Grid 2×2 outcome (completato/no-show/ripianificato/cancellato)                | inline sheet                                         |
| loading/error    | n/a                                       | **Non implementato**                                                          | —                                                    |

---

## 3. Dati & Sorgenti

| Entità UI         | Mock (src/data/mock/sales.ts)                               | Schema reale (v6.1)                                  | Note                                                           |
| ----------------- | ----------------------------------------------------------- | ---------------------------------------------------- | -------------------------------------------------------------- |
| Appuntamenti      | `salesAppointments` (riga 3808)                             | `sales.appointments`                                 | Filtrato client-side su `scheduledAt`                          |
| Appuntamenti oggi | `getAppointmentsToday()` (hardcoded `2026-04-18`)           | idem + `WHERE scheduled_at::date = CURRENT_DATE`     | **Bug mock**: data hardcoded, non coerente con `SALES_NOW_ISO` |
| Overdue           | `getOverdueAppointments()` (status='scheduled' AND < now)   | idem                                                 | Status resta `scheduled` finché non si registra esito          |
| Azienda/contatto  | `getCompanyById`, `getContactById`                          | `core.companies`, `core.contacts`                    | Join richiesto                                                 |
| Lead (sheet new)  | `salesLeads` filtrati per `CURRENT_SALES_USER_ID`           | `sales.leads WHERE assigned_to_user_id = auth.uid()` | Ricerca su `businessName` + `positionTitle`                    |
| Tipo appuntamento | `AppointmentType` enum 5-valori                             | `sales.appointments.appointment_type`                | `TYPE_UI` map per label/short/accent/icon                      |
| Status            | `AppointmentStatus` enum                                    | `sales.appointments.status`                          | 5 valori — enum vincola transizioni                            |
| Video provider    | hardcoded 4 (meet/zoom/teams/whereby) in `generateVideoUrl` | TBD: `sales.appointment_providers`                   | Oggi URL pseudo-random, nessuna integrazione API               |
| Durata            | `durationMinutes` dropdown [15,20,30,45,60,75,90]           | `sales.appointments.duration_minutes`                | —                                                              |
| Esito             | `OutcomeType` in state locale                               | `sales.appointments.status` + `outcome_notes`        | Submit non collegato (button "Conferma esito" = `onClose`)     |

---

## 4. Pulsanti & Azioni

| Azione                  | Trigger UI                                  | Effetto attuale (mock)                                       | Effetto target (produzione)                                                                 | Gated da                     |
| ----------------------- | ------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------- | ---------------------------- |
| Switch vista            | `ViewToggle` Lista/Calendario               | `setView`                                                    | Stesso + persist in `localStorage` o profilo utente                                         | —                            |
| Cambia tab lista        | `SegmentedControl` oggi/domani/sett/passati | `setListTab`                                                 | Stesso + analytics                                                                          | —                            |
| Settimana prev/next     | frecce calendar header                      | `setWeekAnchor(±7g)`                                         | Stesso + prefetch server-side                                                               | —                            |
| Oggi (calendar)         | pill Oggi                                   | Reset anchor a start-of-week(now)                            | Stesso                                                                                      | —                            |
| Apri dettaglio          | click row/block                             | Open BottomSheet detail                                      | Stesso + fetch dettaglio fresh                                                              | —                            |
| FAB nuovo               | tap `+`                                     | Open NewAppointmentSheet senza slot hint                     | Stesso                                                                                      | —                            |
| Crea da slot calendar   | click cella oraria                          | Open sheet con `slotHint={day, hour}` pre-fill               | Stesso + detect conflitti                                                                   | —                            |
| Selezione lead in sheet | input search + click row                    | `setLeadId`                                                  | Autocomplete server-side con debouncing                                                     | —                            |
| Crea appuntamento       | button `Crea`                               | `salesAppointments.push(newAppt)` — **mutazione array mock** | `INSERT sales.appointments` + invalidate cache + push su calendar provider (Google/Outlook) | Lead + data/ora required     |
| Genera link video       | button `Genera link`                        | `Math.random()` URL fasullo con prefisso provider            | Edge function `create-meeting-link` che chiama Meet/Zoom/Teams API e persiste URL           | `appointmentType=video_call` |
| Avvia video             | button `Avvia`                              | `window.open(url)`                                           | Stesso + log evento `meeting_started`                                                       | `videoUrl` presente          |
| Copia video URL         | button `Copia`                              | `navigator.clipboard.writeText(url)`                         | Stesso + toast                                                                              | —                            |
| Invia video URL         | button `Invia`                              | **NO-OP** (nessun handler)                                   | Apre drawer per scegliere canale (WhatsApp/Email/SMS) + invia via Twilio/SendGrid           | —                            |
| Scegli provider         | chip meet/zoom/teams/whereby                | `setVideoProvider`                                           | Stesso + filtra se integrazione disabilitata per org                                        | —                            |
| Registra esito          | 4 button outcome                            | `setOutcomeValue` (state locale)                             | `UPDATE sales.appointments SET status, outcome_notes, completed_at`                         | `isPast`                     |
| Conferma esito          | button "Conferma esito"                     | **NO-OP** (chiama solo `onClose`)                            | Persist outcome + trigger n8n next-step (follow-up task/quote)                              | Esito selezionato            |
| Vai alla lead           | button "Vai alla lead"                      | `navigate(/sales/lead/:leadId)` + close sheet                | Stesso                                                                                      | —                            |
| Chiudi                  | button Chiudi                               | `onClose`                                                    | Stesso                                                                                      | —                            |
| Apri da banner overdue  | button Apri nel banner                      | Imposta vista=list + tab=overdue                             | Stesso                                                                                      | `overdueList.length>0`       |
| Cambia lead selezionata | X nel chip lead                             | `setLeadId("")`                                              | Stesso                                                                                      | —                            |
| Apri lead da sheet new  | link "Apri la lead" in fondo                | `navigate(/sales/lead/:id)` + close sheet                    | Stesso                                                                                      | lead selezionata             |

---

## 5. Componenti chiave

- [BottomSheet, IOSButton, SegmentedControl](../../../src/components/ios/IOSPrimitives.tsx)
- [EmptyState](../../../src/components/sales/EmptyState.tsx)
- `ViewToggle` (locale) — Lista/Calendario
- `AppointmentList` → `AppointmentRow` (locale) — lista raggruppata per giorno
- `CalendarView` → `WeekGrid` → `DayColumn` → `ApptBlock` (locale desktop)
- `CalendarView` → `DayAgenda` → `AgendaCard` (locale mobile)
- `AppointmentDetailSheet` (locale) — bottom sheet dettaglio
- `NewAppointmentSheet` (locale) — form creazione
- Helpers locali: `startOfWeek`, `addDays`, `sameDay`, `fmtRange`, `generateVideoUrl`, `toDateInput`, `toTimeInput`
- **Nota**: esiste [AppointmentEditor](../../../src/components/sales/AppointmentEditor.tsx) ma **non è usato qui** — duplicazione probabile con `NewAppointmentSheet`

---

## 6. Integrazioni future

- **Supabase tables**: `sales.appointments`, `sales.leads`, `core.companies`, `core.contacts`, `sales.users`, TBD `sales.meeting_links`
- **RLS policy**: sales vede solo `owner_user_id = auth.uid()` + team POD se director; write solo proprio owner
- **Realtime channel**: `appointments:<ownerUserId>` per now-line + aggiornamento live se collega cambia esito
- **Edge function**:
  - `create-meeting-link` (integrazioni Google Meet/Zoom/Teams/Whereby)
  - `sync-calendar` push/pull due-way con Google Calendar/Outlook
  - `send-meeting-invite` via Gmail/Outlook API
- **n8n workflow**:
  - `appointment-reminder-30min` via WhatsApp/email
  - `outcome-recorded` → se `completed` + tipo `closer_call` trigger `prepare-quote-draft`
  - `overdue-appointment-alert` se status resta `scheduled` > 2h dopo `scheduledAt`
- **AI call**: Claude per riassunto outcome notes post-call (se abbinato a speech-to-text Whisper); suggerimento agenda pre-call basato su lead stage
- **Analytics event**: `appointment_created`, `appointment_opened`, `meeting_link_generated`, `outcome_recorded`, `overdue_banner_clicked`, `calendar_view_changed`

---

## 7. UX/UI review

### Funziona bene

- Doppia vista Lista/Calendario copre sia mindset "task list" (oggi/domani) che "pianificazione visiva" — switch immediato.
- Now-line rossa + slot cliccabile sulle celle vuote = pattern calendar moderno, molto intuitivo per creare al volo.
- Banner overdue è la feature più impattante: non lascia esiti pending in limbo, forza chiusura ciclo.
- Accent color per tipo (setter=amber, closer=emerald, video=indigo, presence=red) è coerente e velocizza lo scanning.
- Sheet dettaglio contestuale per tipo (video mostra provider picker solo se manca URL, esito solo se `isPast`).
- Mobile day picker orizzontale + agenda verticale è molto iOS-native.

### Attriti UX attuali

- **Diverse azioni core sono no-op**: "Conferma esito", "Invia link video", creazione link è pseudo-random. Demo OK, pre-produzione serve wire-up o banner "Coming soon".
- `getAppointmentsToday()` ha data hardcoded `2026-04-18` mentre `SALES_NOW_ISO` può essere diverso → "Oggi" non corrisponde alla giornata corrente in demo.
- Fuso orario: zero gestione TZ; `new Date(SALES_NOW_ISO)` + `setHours(0)` in locale può sballare ai bordi giorno.
- Range orario calendario 8-20 hardcoded: appuntamenti fuori range (alba/sera) spariscono silenziosamente (`if (h < 8 || h >= 20) return null`).
- Blocchi sovrapposti nel week grid non gestiti (overlay su stessa colonna senza side-by-side layout).
- Duplicazione dei form: esiste `AppointmentEditor` in `components/sales/` non usato qui — debito tecnico.
- Outcome options solo 4 stati — manca "Rimandato a data X" con picker o "Esito parziale" per multi-step setter→closer.
- Form new: no validazione su conflitti orario, no campo contatto override (usa `primaryContactId` silenziosamente).
- Hover `Nuovo` sugli slot: label appare solo su hover desktop, invisibile su touch — serve tap-and-hold o onclick diretto.

### Coerenza UI

- Tipografia: ok — numeriche tabular-nums consistenti, font-mono per orari.
- Spacing: ok — `max-w-[1280px]` più largo delle altre pagine (giustificato per calendar).
- Colore: ok — accent system ben costruito, ma mancano i token `--color-{indigo,amber,emerald,red,slate}-soft/dark` nell'snippet CSS (vedere `theme.css` per validare che esistano; se no i chip risulterebbero trasparenti).
- Mobile-first: ok — layout scrolla bene, FAB a 80px da bottom (sopra SalesTabBar) corretto.

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo | Proposta                                                                                                  | Effort |
| --- | -------- | ---- | --------------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0       | Bug  | Collegare handler Crea (oggi push su array mock, pipeline produzione deve persistere)                     | M      |
| 2   | P0       | Bug  | Collegare `Conferma esito` al submit effettivo (UPDATE appointment.status + outcome_notes + completed_at) | M      |
| 3   | P0       | UX   | Fix `getAppointmentsToday` — derivare giorno da `SALES_NOW_ISO` non hardcoded                             | S      |
| 4   | P0       | UX   | Link video reale via edge function (Meet/Zoom/Teams API) — oggi URL fasullo non apribile                  | L      |
| 5   | P1       | UX   | Range orario calendar configurabile (default 8-20 ma surface se presenti appuntamenti fuori range)        | M      |
| 6   | P1       | UX   | Gestione conflitti/sovrapposizioni nel week grid (colonne affiancate o badge "+2")                        | M      |
| 7   | P1       | UI   | Consolidare `AppointmentEditor` e `NewAppointmentSheet` in un unico componente riusabile                  | L      |
| 8   | P1       | UX   | Add validation conflitti orario in `NewAppointmentSheet`                                                  | S      |
| 9   | P2       | UX   | Drag-to-reschedule sul week grid (drag block su nuovo slot)                                               | L      |
| 10  | P2       | UX   | Outcome "Ripianificato" apre subito date/time picker per creare nuovo appt collegato                      | M      |
| 11  | P2       | UX   | Contatto selezionabile nella form (oggi prende silenziosamente `primaryContactId`)                        | S      |
| 12  | P2       | A11y | Calendar grid con `role=grid`, celle `role=gridcell`, focus visibile su now-line                          | M      |
| 13  | P3       | UX   | Integrazione due-way Google Calendar/Outlook                                                              | XL     |
| 14  | P3       | UX   | Vista "Giorno" (day view) desktop oltre a week                                                            | L      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- Fuso orario di default: UTC, Europe/Rome, o locale utente? Impatta tutto su DST.
- Video provider default per org: hardcoded meet o configurabile in admin? Serve `org_settings`?
- Come gestire appuntamenti "ricorrenti" (setter call settimanale su stessa lead)? Non previsto in schema.
- Cancellazione: soft-delete (status=cancelled) o hard-delete? Schema suggerisce soft via enum status.
- Calendar permission: director/admin deve poter vedere calendario di tutto il POD? Overlay o filtro dropdown?
- Trigger sync con Google/Outlook: manuale (button) o automatico a ogni INSERT/UPDATE?
- `outcome_notes` deve essere obbligatorio per `no_show`/`cancelled` (per audit) o opzionale?
- Setter → Closer handoff: quando un setter completa call con "qualified", crea auto closer call? Chi owner?
