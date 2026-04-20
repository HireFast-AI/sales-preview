# Preventivi

**Route**: `/sales/preventivi`
**File**: [src/pages/sales/SalesPreventivi.tsx](../../../src/pages/sales/SalesPreventivi.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: tutti i sales (director/admin vedono anche team — TBD, filtro non presente)

---

## 1. Overview

Hub preventivi del sales: 4 colonne logiche (da inviare · inviati · accettati · scaduti) con counter live, lista iOS-style per ogni stato e bottom-sheet per creare un nuovo preventivo (da template o da zero) legato a una lead esistente. Sostituisce il giro "Notion → PDF → email" con un flusso one-click pre-strutturato.

---

## 2. Viste & Stati

| ID vista       | Trigger                                | Descrizione                                                                      | Componenti rilevanti                           |
| -------------- | -------------------------------------- | -------------------------------------------------------------------------------- | ---------------------------------------------- |
| default-draft  | mount (`filter='draft'`)               | Lista draft con icona `FileText`, trailing "Draft" uppercase                     | `IOSNavBar`, `SegmentedControl`, `GroupedList` |
| sent           | tab `Inviati`                          | Solo `status=sent && !isExpired`, icona `Send` arancione, trailing "sent X fa"   | idem                                           |
| accepted       | tab `Accettati`                        | `status=accepted`, icona `CheckCircle2` verde, trailing relative dall'acceptedAt | idem                                           |
| expired        | tab `Scaduti`                          | `isExpired(q)` true (validUntil < now), icona `AlertCircle` rossa                | idem                                           |
| empty-draft    | `filtered.length=0 && filter=draft`    | Copy "Nessun preventivo da inviare" + subtitle specifica                         | `EmptyState`                                   |
| empty-accepted | `filtered.length=0 && filter=accepted` | Copy "Nessun accettato per ora" + tone=positive                                  | `EmptyState`                                   |
| empty-generic  | altri filtri vuoti                     | Copy generica "Nessun preventivo"                                                | `EmptyState`                                   |
| sheet-new      | click "Nuovo" in navbar                | BottomSheet con choice template/zero + lead select + tipo + importo + 2 CTA      | `BottomSheet`, `IOSButton`                     |
| loading/error  | n/a                                    | **Non implementato**                                                             | —                                              |

---

## 3. Dati & Sorgenti

| Entità UI           | Mock (src/data/mock/sales.ts)                    | Schema reale (v6.1)                         | Note                                                          |
| ------------------- | ------------------------------------------------ | ------------------------------------------- | ------------------------------------------------------------- |
| Preventivi          | `salesQuotes` (riga 4137)                        | `sales.quotes`                              | 5 stati: draft/sent/accepted/rejected/expired                 |
| Template            | `salesQuoteTemplates` (riga 5090)                | `sales.quote_templates`                     | Usato solo per counter nel sheet; non c'è picker vero         |
| Lead collegata      | `salesLeads` → `getCompanyById(l.companyId)`     | `sales.leads` + `core.companies`            | Select HTML standard, no search                               |
| Tipo preventivo     | `QuoteType` enum (6 valori) + `quoteTypeLabel`   | `sales.quotes.quote_type`                   | interviews_package, subscription, flash_subscription, ecc.    |
| Status              | `QuoteStatus` enum + `isExpired()` helper locale | `sales.quotes.status` + derivato validUntil | `expired` come filtro = `isExpired()` calcolato client-side   |
| Importo             | `totalAmount` → `formatEuro(...)`                | `sales.quotes.total_amount`                 | Numeric/Decimal col server                                    |
| Azienda             | `getCompanyById(q.companyId)`                    | `core.companies`                            | Join denormalizzato in UI                                     |
| Scadenza            | `q.validUntil` (opzionale)                       | `sales.quotes.valid_until`                  | Timestamp; se null, mai scade                                 |
| SentAt / acceptedAt | `q.sentAt`, `q.acceptedAt`                       | colonne omonime                             | `formatRelative()` localizzato (oggi approssimato, 30gg=mese) |
| quoteNumber         | `q.quoteNumber`                                  | `sales.quotes.quote_number`                 | Serial tipo "Q-2026-0412" — generation server-side            |

---

## 4. Pulsanti & Azioni

| Azione                 | Trigger UI                       | Effetto attuale (mock)                     | Effetto target (produzione)                                                                           | Gated da               |
| ---------------------- | -------------------------------- | ------------------------------------------ | ----------------------------------------------------------------------------------------------------- | ---------------------- |
| Apri sheet nuovo       | button "Nuovo" in navbar         | `resetForm()` + `setSheetOpen(true)`       | Stesso                                                                                                | —                      |
| Cambia tab             | `SegmentedControl` (4 tab)       | `setFilter`                                | Stesso + analytics                                                                                    | —                      |
| Tap riga preventivo    | click item `GroupedList`         | **NO-OP** (nessun `to:` né `onClick`)      | Naviga a `/sales/preventivi/:id` con detail page (preview PDF + history stato + invio/re-send/cancel) | —                      |
| Scegli source template | tile "Da template"               | `setSource("template")` + highlight border | Stesso + mostra lista template filtrabile per `quoteType`                                             | —                      |
| Scegli source zero     | tile "Da zero"                   | `setSource("zero")` + highlight border     | Stesso + form wizard campi avanzati                                                                   | —                      |
| Seleziona lead         | `<select>` nativo                | `setLeadId`                                | Autocomplete + preview azienda + pre-fill importo da `lead.budgetDisponibileEur`                      | —                      |
| Seleziona tipo         | `<select>` nativo su 6 QuoteType | `setQuoteType`                             | Stesso + card template con pricing default dal `salesQuoteTemplates`                                  | —                      |
| Inserisci importo      | input number                     | `setAmount`                                | Stesso + validazione range per tipo + warning "molto inferiore al pacchetto tipico"                   | —                      |
| Salva draft            | IOSButton tinted                 | `setSheetOpen(false)` — **nessun insert**  | `INSERT sales.quotes (status='draft', ...)` + chiudi sheet + toast                                    | Campi minimi richiesti |
| Genera e invia         | IOSButton orange                 | `setSheetOpen(false)` — **nessun invio**   | Edge fn `generate-quote-pdf` → signed URL → email via SendGrid/Gmail + UPDATE status=sent + sentAt    | Lead + importo + tipo  |
| Chiudi sheet           | backdrop/X                       | `setSheetOpen(false)`                      | Stesso + conferma se form dirty                                                                       | —                      |
| — (mancante)           | Detail preventivo                | Assente                                    | Pagina `/sales/preventivi/:id` con preview PDF + timeline (draft→sent→opened→accepted)                | —                      |
| — (mancante)           | Duplicare preventivo             | Assente                                    | Menu "Duplica" su riga esistente                                                                      | —                      |
| — (mancante)           | Re-send                          | Assente                                    | Re-send email + update `resentAt` (schema TBD)                                                        | `status=sent`          |
| — (mancante)           | Segna accettato manualmente      | Assente                                    | UPDATE status=accepted (caso firma offline)                                                           | —                      |
| — (mancante)           | Cancella draft                   | Assente                                    | DELETE o soft-delete                                                                                  | `status=draft`         |

---

## 5. Componenti chiave

- [IOSNavBar](../../../src/components/ios/IOSNavBar.tsx) — large title + subtitle counter + action "Nuovo"
- [SegmentedControl](../../../src/components/ios/IOSPrimitives.tsx) — filtro 4 stati
- [GroupedList](../../../src/components/ios/GroupedList.tsx) — lista preventivi
- [BottomSheet, IOSButton](../../../src/components/ios/IOSPrimitives.tsx) — sheet creazione
- [EmptyState](../../../src/components/sales/EmptyState.tsx) — stato vuoto
- Helpers locali: `statusMeta()`, `formatRelative()`, `isExpired()` (non estratti)

---

## 6. Integrazioni future

- **Supabase tables**: `sales.quotes`, `sales.quote_templates`, `sales.quote_line_items` (TBD se schema prevede righe dettaglio), `sales.leads`, `core.companies`, `storage.objects` (PDF)
- **RLS policy**: read/write su `sales.quotes` se `created_by_user_id = auth.uid()` o stesso POD se director; insert vincolato a lead assegnata
- **Realtime channel**: `quotes:<userId>` per aggiornamento live quando cliente apre/accetta (trigger webhook PDF viewer)
- **Edge function**:
  - `generate-quote-pdf` — render Handlebars/React-PDF → upload Supabase Storage → signed URL
  - `send-quote-email` — SendGrid/Gmail con tracking pixel
  - `quote-webhook-signed` — ricezione webhook da DocuSign/Yousign per accettazione digitale
- **n8n workflow**:
  - `quote-sent-follow-up` — dopo 3gg senza apertura invia reminder
  - `quote-accepted` → crea lead status=cliente_attivo + genera `core.contracts`
  - `quote-expiring-soon` — alert a 48h da `validUntil`
- **AI call**: Claude per generare `notes` personalizzate dal profilo lead + market investigation; smart pricing suggestion basato su `budgetDisponibileEur` + storico POD
- **Analytics event**: `quote_created`, `quote_sent`, `quote_opened_by_client`, `quote_accepted`, `quote_expired`, `quote_filter_changed`

---

## 7. UX/UI review

### Funziona bene

- Counter nel subtitle navbar (`3 da inviare · 7 in attesa`) dà immediatamente senso di urgenza.
- Segmented control a 4 tab con badge numerici è scansionabile istantaneamente.
- Leading colorato per status (draft grigio, sent arancio, accepted verde, expired rosso) = semaforo chiaro.
- `formatRelative` fa sembrare ogni riga "viva" ("3g fa", "2h fa").
- Sheet di creazione minimale (4 campi) è più veloce di Notion: tradeoff forma-contenuto ok.

### Attriti UX attuali

- **Row non cliccabile**: mancando `to:` / `onClick`, il preventivo è dead-end. Il sales vede "€6000 in attesa" ma non può ispezionarlo/re-inviarlo/duplicarlo.
- **"Genera e invia" chiude il sheet senza fare nulla**: demo misleading — sembra funzionare, non è così.
- Select `<select>` nativo per lead è imbarazzante con 100+ leads (nessuna ricerca, scroll infinito in dropdown del browser). Pattern già risolto in `NewAppointmentSheet` con autocomplete — incoerenza.
- Source picker template/zero non porta a nulla di diverso: stessi campi mostrati comunque. Il choice è cosmetico.
- `formatRelative` ha edge cases strani: "fra 3g fa" per future date negative, "mes" abbreviato (mese? mesi?) — i18n approssimativo.
- Nessun validation: si può salvare un draft senza lead né importo. Poi in lista apparirà "— · €0".
- Manca il concetto di "richiede attenzione": un draft vecchio di 15gg = peso morto ma in lista appare come gli altri.
- Filtro `Scaduti` conta `isExpired(q)` che include `status=sent` passati validUntil ma anche `status=expired`: overlap semantico nei conteggi.

### Coerenza UI

- Tipografia: ok — tabular-nums per orari/importi coerente.
- Spacing: ok — `max-w-[840px]`, `space-y-5` allineato a Ricerca Mercato.
- Colore: ok — palette status coerente; draft grigio un po' spento, draft vecchi potrebbero usare accent arancione.
- Mobile-first: ok — sheet full-height su mobile, segmented scorre orizzontalmente se necessario (TBD, dipende da primitive).

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo   | Proposta                                                                                            | Effort |
| --- | -------- | ------ | --------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0       | UX/Bug | Riga cliccabile → detail page (preview PDF + timeline + actions re-send/cancel/segna accettato)     | L      |
| 2   | P0       | UX/Bug | "Genera e invia" wire-up reale con edge function + stato loading + toast success/error              | L      |
| 3   | P0       | UX     | Validation client-side: lead required, amount > 0, tipo required — oggi si chiude senza feedback    | S      |
| 4   | P1       | UX     | Lead picker con autocomplete (reuso pattern `NewAppointmentSheet`) invece di `<select>` nativo      | M      |
| 5   | P1       | UX     | Source "Da template" apre lista template concreta con pricing default per tipo                      | M      |
| 6   | P1       | UX     | Badge "Richiede attenzione" su draft > 3gg o sent non aperto > 5gg                                  | S      |
| 7   | P1       | UX     | Pre-fill importo da `lead.budgetDisponibileEur` e warning se diverge >20%                           | S      |
| 8   | P2       | UI     | Mostrare data scadenza + distanza (`scade fra 2g` in arancio, `scaduto 3g fa` in rosso)             | S      |
| 9   | P2       | UX     | Bulk actions (multi-select + "Rinvia selezionati")                                                  | M      |
| 10  | P2       | A11y   | `<select>` nativo non accessibile con VoiceOver iOS sotto a 50 righe — sostituire con combobox ARIA | M      |
| 11  | P3       | UX     | Drag-drop di PDF signati per marcare accettato manualmente (caso firma offline)                     | L      |
| 12  | P3       | UX     | Timeline storico versioni (se modifica importo post-invio, segna v2)                                | L      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- Il `quoteNumber` è generato server-side con sequence per anno/POD o globale? Formato definitivo?
- Una lead può avere più preventivi attivi contemporaneamente (ipotesi A = subscription, ipotesi B = flash) o uno solo alla volta?
- Accettazione: firma digitale integrata (DocuSign/Yousign) o upload PDF firmato manuale?
- "Sent" = email inviata oppure "consegnata + aperta"? Serve pixel tracking e distinzione stati (opened/viewed).
- Expired automatico: cron n8n mette status=expired o resta `sent` e UI lo deriva? Meglio cron per analytics.
- `salesQuoteTemplates` ha 5090-relative records — come sono legati ai quote types (1-N)? Filtrabile nel picker?
- Righe dettaglio (line items): schema attuale ha solo `totalAmount` flat — serve tabella `quote_line_items` per preventivi granulari?
- Integrazione con `core.contracts`: quando accettato, chi genera il contratto? Auto vs. manual step?
- Gestione IVA/imponibile: oggi solo `totalAmount` — in IT serve `imponibile` + `iva_amount` separati per fatturazione.
