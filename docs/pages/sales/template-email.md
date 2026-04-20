# Template Email

**Route**: `/sales/template-email`
**File**: [src/pages/sales/SalesTemplateEmail.tsx](../../../src/pages/sales/SalesTemplateEmail.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: tutti (lettura template organizzazione); admin per editing/creazione

---

## 1. Overview

Libreria interna di **template email pre-approvati** organizzati per categoria (primo contatto, follow-up, offerta, firma, pagamento, grazie, risveglio, onboarding, referral, stagionali). Il sales cerca per nome/oggetto, filtra per categoria via SegmentedControl, apre un template, copia il testo o lo pre-compila per una lead selezionata (lead picker in un secondo BottomSheet con anteprima sostituita delle variabili `{{company}}`, `{{position}}`, `{{headcount}}`). È la fonte unica di verità per il tone-of-voice HireFast sulla comunicazione scritta e l'innesco rapido di invii da Conversazioni/LeadDetail.

---

## 2. Viste & Stati

| ID vista            | Trigger                             | Descrizione                                                                                                                | Componenti rilevanti                                         |
| ------------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `default`           | Default mount                       | Search sticky + SegmentedControl categorie + GroupedList template filtrati                                                 | `IOSNavBar`, `SegmentedControl`, `GroupedList`               |
| `empty-filtered`    | Nessun template matcha `cat+search` | EmptyState "Nessun template · Prova un'altra categoria o ricerca" con icon `FileText`                                      | [`EmptyState`](../../../src/components/sales/EmptyState.tsx) |
| `detail-sheet`      | `selectedId !== null`               | BottomSheet con chip categoria, subject, body con `{{var}}` in teal, lista variabili, CTA Copia/Usa per lead               | `BottomSheet`, `IOSButton`, `Chip`                           |
| `detail-copied`     | Click "Copia testo"                 | CTA cambia label in "Copiato" con icon `Check` per 1.6s                                                                    | feedback visivo                                              |
| `lead-picker-sheet` | Click "Usa per lead"                | Secondo BottomSheet con prime 12 lead; tap apre preview                                                                    | lista lead button                                            |
| `lead-preview`      | `selectedLeadId !== null`           | Preview sostituito (subject + body con `{{company}}/{{position}}/{{headcount}}` risolti) + CTA Cambia lead / Componi email | `LeadPreview`                                                |
| loading             | —                                   | Non implementato (sync)                                                                                                    | —                                                            |
| error               | —                                   | Non implementato                                                                                                           | —                                                            |

---

## 3. Dati & Sorgenti

| Entità UI               | Mock (src/data/mock/sales.ts)                         | Schema reale                                        | Note                                                                                                                  |
| ----------------------- | ----------------------------------------------------- | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Template                | `salesEmailTemplates` (10 seed, `SalesEmailTemplate`) | `sales.email_templates` (schema v6.1)               | Campi: `name`, `category`, `language`, `subject`, `bodyText`, `variables[]`, `suggestedForStatuses[]`, `displayOrder` |
| Categoria → label       | `emailTemplateCategoryLabel` (record)                 | Enum DB o tabella `sales.email_template_categories` | 10 categorie mappate                                                                                                  |
| Lead picker             | `salesLeads.slice(0, 12)`                             | `sales.leads`                                       | Prime 12 (paginazione fittizia)                                                                                       |
| Company (preview)       | `getCompanyById(l.companyId)`                         | `core.companies`                                    | Resolve `{{company}}`                                                                                                 |
| Lead per preview        | `salesLeads.find(l => l.id === leadId)`               | `sales.leads`                                       | Resolve `{{position}}`, `{{headcount}}`                                                                               |
| Abbreviazioni categoria | `CATEGORIES` const locale                             | UI-only                                             | 11 entry con `label` + `abbr`                                                                                         |

---

## 4. Pulsanti & Azioni

| Azione           | Trigger UI                  | Effetto attuale (mock)                                           | Effetto target (produzione)                                                                                                                            | Gated da                      |
| ---------------- | --------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------- |
| Search           | Input field                 | Filtra su `name`+`subject` case-insensitive                      | Idem + full-text su `bodyText`                                                                                                                         | —                             |
| Filtro categoria | SegmentedControl            | `setCat(id)`                                                     | Idem + analytics                                                                                                                                       | —                             |
| Apri template    | Tap riga lista              | `setSelectedId(id)` → BottomSheet                                | Idem + prefetch variabili richieste                                                                                                                    | —                             |
| Copia testo      | CTA `Copia testo`           | `navigator.clipboard.writeText(bodyText)` + flash "Copiato" 1.6s | Idem + log `template_copied`                                                                                                                           | navigator.clipboard support   |
| Usa per lead     | CTA `Usa per lead` (orange) | Apre secondo sheet `leadSheetOpen=true`                          | Idem                                                                                                                                                   | Template selezionato          |
| Seleziona lead   | Tap lead                    | `setSelectedLeadId(id)` → switch a preview                       | Idem + fetch full lead context (company, contacts, quote, history)                                                                                     | —                             |
| Cambia lead      | CTA preview                 | `setSelectedLeadId(null)` torna a picker                         | Idem                                                                                                                                                   | —                             |
| Componi email    | CTA preview (orange)        | **No-op**                                                        | Crea draft in `communications.conversations` (se non esiste) + navigate `/sales/conversazioni/:convId` con body prefilled, oppure apre composer inline | Lead ha email primaria valida |
| Chiudi sheet     | Tap fuori / handle          | `setSelectedId(null)` / chiusura stati                           | Idem                                                                                                                                                   | —                             |

---

## 5. Componenti chiave

- [`IOSNavBar`](../../../src/components/ios/IOSNavBar.tsx) — titolo + conteggio template in subtitle
- [`SegmentedControl`](../../../src/components/ios/IOSPrimitives.tsx) — filtro categorie scrollable
- [`GroupedList`](../../../src/components/ios/GroupedList.tsx) — lista template con leading chip abbreviata
- [`BottomSheet`](../../../src/components/ios/IOSPrimitives.tsx) — detail template + lead picker (due sheet separati)
- [`IOSButton`](../../../src/components/ios/IOSPrimitives.tsx) — CTA Copia / Usa per lead / Componi email
- [`Chip`](../../../src/components/common/Chip.tsx) — categoria + variabili
- [`EmptyState`](../../../src/components/sales/EmptyState.tsx) — fallback filtro senza risultati
- Helper locale: `renderBody(text)` — split regex `\{\{[^}]+\}\}` e rendering span teal per evidenziare variabili
- Helper locale: `fill(text)` — sostituzione semplice 3 variabili hardcoded in `LeadPreview`

---

## 6. Integrazioni future

- **Supabase tables**: `sales.email_templates`, `sales.email_template_categories`, `sales.email_template_usage` (log ogni copia/invio per ranking), `core.companies`, `sales.leads`
- **RLS policy**: template `language='it'` + `org_id = current_user.org_id`; editing solo per ruolo admin o owner del template
- **Realtime channel**: nessuno critico; nice-to-have `template:org:<orgId>` per broadcast update quando admin modifica un template condiviso
- **Edge function**:
  - `template-fill` → riceve `templateId` + `leadId`, resolve TUTTE le variabili (non solo 3) via join con `core.contacts`, `sales.quotes`, `core.users` (sender)
  - `email-compose-from-template` → crea draft in `communications.conversations`, associa a conversation esistente per `leadId+contactId` se presente
- **n8n workflow**: `template-suggester` — cron giornaliero che analizza stato lead e suggerisce template appropriato (via `suggestedForStatuses`) pushando notifica in Home sales
- **AI call**: Anthropic Claude Sonnet 4.5 per (a) **riadattare tono** di un template a persona specifica ("rendilo più casual", "più formale", "più breve"), (b) **generare nuovo template** partendo da brief + 3 template simili come few-shot, (c) **spell/grammar check** multilingua prima dell'invio
- **Analytics event**: `template_opened{templateId,category}`, `template_copied{templateId}`, `template_applied_to_lead{templateId,leadId}`, `template_search{query,resultsCount}`, `template_category_filtered{cat}`

---

## 7. UX/UI review

### Funziona bene

- Search + SegmentedControl sticky top: filtri sempre accessibili anche dopo scroll
- Evidenziazione `{{variabili}}` in teal semibold nel body: colpo d'occhio immediato su cosa va personalizzato
- Chip abbreviata ("F-up", "Cold", "Offer") come leading delle righe lista: riconoscimento rapido della categoria senza verbosità
- Due BottomSheet in cascata (template → lead picker → preview) mantiene mobile-first senza navigate tra pagine
- Feedback "Copiato" con transizione icon `Copy`→`Check` per 1.6s: micro-interazione soddisfacente
- `whitespace-pre-wrap` e font-sans sul pre-body: leggibile e rispetta line-break dei template
- Lista `variables` esposta esplicitamente: aiuta sales a capire cosa serve sapere del lead prima di usare

### Attriti UX attuali

- **Solo 3 variabili risolte** in `LeadPreview` (`company`, `position`, `headcount`); gli altri placeholder (`{{first_name}}`, `{{sender_name}}`, `{{total_amount}}`, `{{invoice_number}}`, `{{sector}}`, `{{signature_link}}`, `{{sender_phone}}`) rimangono **letterali nel preview**: l'utente vede `{{first_name}}` non sostituito e può confondersi
- Lead picker mostra **solo prime 12 lead** (`.slice(0, 12)`), senza search o paginazione: con 100+ lead inutilizzabile
- Nessun **suggerimento contestuale**: il template ha `suggestedForStatuses: ['new', 'qualified']` ma la UI non ordina né evidenzia i template adatti allo status del lead scelto
- CTA "Componi email" è **no-op**: il flusso termina nel vuoto
- Nessun **editing inline** del testo prima dell'invio (l'utente deve copiare e incollare altrove)
- Nessuna distinzione lingua IT/EN visibile nelle righe lista
- Nessun **preview mobile/desktop** (come renderizza in un client email)
- `SegmentedControl` con 11 opzioni **overflow orizzontale** su mobile: scroll non ovvio
- Nessun contatore di **utilizzo** per template ("usato 12 volte · tasso reply 18%")
- `displayOrder` non visibile e non modificabile in UI
- Nessun **versioning** né storico modifiche dei template
- Copy plain text: i template rich (bold, link) non sono supportati

### Coerenza UI

- Tipografia: ok — font-serif per titoli, mono-like per body text wrappable
- Spacing: ok — padding sheet e card coerenti
- Colore: ok — teal semibold per variabili coerente con accent brand
- Mobile-first: ok nel flusso principale; il SegmentedControl a 11 opzioni è l'unico punto debole

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo | Proposta                                                                                                                                                                                         | Effort |
| --- | -------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------ |
| 1   | P0       | UX   | **Resolve completa variabili**: fetch context lead (contact primario, user sender, quote collegato) e sostituire tutte le `variables[]` nel preview; evidenziare in rosso quelle non risolvibili | M      |
| 2   | P0       | UX   | **Wiring CTA Componi email**: crea/apre conversation, prefilla subject+body e manda sul /sales/conversazioni/:id con draft non ancora inviato                                                    | M      |
| 3   | P1       | UX   | **Lead picker con search + paginazione** (virtual list); ordina per "Suggerito per questo template" usando `suggestedForStatuses` match con lead.status                                          | M      |
| 4   | P1       | UX   | **Editor inline** nel preview (textarea editabile prima di confermare invio) con undo e reset                                                                                                    | M      |
| 5   | P1       | UI   | **Badge lingua** (IT/EN) sulla riga lista; filtro lingua in testa                                                                                                                                | S      |
| 6   | P1       | UX   | **Usage stats** per template (utilizzi 30gg, reply rate, open rate) mostrate nel detail; ranking automatico                                                                                      | L      |
| 7   | P2       | AI   | **Riscrittura AI**: bottone "Rendi più breve / più formale / più casual" che chiama Claude con istruzioni short-form e aggiorna preview                                                          | M      |
| 8   | P2       | UX   | **Preview rendering HTML** (non solo plain), con toggle mobile/desktop                                                                                                                           | L      |
| 9   | P2       | UX   | **Hot-swap template** da Conversazioni: pulsante "usa template" nel composer apre questa pagina come modal filtrato per `status=conv.lead.status`                                                | M      |
| 10  | P3       | UI   | **Editor admin** per creare/modificare template con live preview e variabili auto-detect                                                                                                         | XL     |
| 11  | P3       | UX   | **Preferiti / pin personale** per accedere rapido ai 3 template più usati del sales                                                                                                              | S      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- Pluralità lingue (IT/EN ora): pipeline i18n / i18n per template stessi o per UI dei placeholder?
- Template a livello org vs team vs personale: gerarchia e override (es. POD Milano Alpha ha override su `first_contact`)?
- Rich text/HTML support: serve MJML, Markdown o plain text solo?
- Come gestiamo **attachment** in template (preventivi PDF generati on-the-fly al momento dell'invio)?
- Variabili derivate: `{{sender_phone}}` va dal profilo utente loggato o dal POD?
- Serve A/B testing built-in (variant A/variant B con performance tracking)?
- Approval workflow prima che un template diventi disponibile al team?
