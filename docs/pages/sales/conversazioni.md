# Conversazioni

**Route**: `/sales/conversazioni` · `/sales/conversazioni/:id`
**File**: [src/pages/sales/SalesConversazioni.tsx](../../../src/pages/sales/SalesConversazioni.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: tutti i ruoli sales (owner vede i propri thread, director/admin li vedono tutti una volta abilitata la RLS)

---

## 1. Overview

Inbox unificata stile WhatsApp Desktop per i sales: raggruppa tutti i thread cliente (email, WhatsApp Business, note telefoniche) attorno a un singolo numero condiviso WABA (`+39 02 9476 8810`) che viene instradato al sales owner del lead. Il sales vede la lista dei thread a sinistra (con badge unread, tempo relativo, preview), il thread attivo a destra con bubble a cluster in stile IM, channel picker inline per rispondere scegliendo Email o WhatsApp, e quick actions nell'header (call, wa.me, mailto, deep-link al lead). È il centro operativo di tutta la comunicazione 1:1 nel funnel post-qualificato.

---

## 2. Viste & Stati

| ID vista                | Trigger                                       | Descrizione                                                                                                                             | Componenti rilevanti                                         |
| ----------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `list-empty`            | Nessuna conversazione in `salesConversations` | EmptyState "Nessuna conversazione" con icona `MessageSquare`                                                                            | [`EmptyState`](../../../src/components/sales/EmptyState.tsx) |
| `list-empty-filtered`   | Search o filterTab non matcha                 | EmptyState "Nessun risultato · Prova con un altro termine" con icona `Filter`                                                           | [`EmptyState`](../../../src/components/sales/EmptyState.tsx) |
| `list-default`          | `salesConversations.length > 0`, nessun `:id` | Lista conversazioni ordinate per `lastMessageAt` desc, con search + filter tabs (Tutte/Da leggere/WhatsApp/Email)                       | `ConversationListColumn`                                     |
| `desktop-empty-pane`    | Desktop, nessun `:id`                         | Pane destro con illustrazione e hint WABA                                                                                               | `EmptyConversation`                                          |
| `thread-default`        | `:id` presente, messaggi > 0                  | Thread con date divider giornalieri, bubble cluster (5 min gap), delivery status check/check-check, input multilinea con channel picker | `ConversationPane`                                           |
| `thread-empty-messages` | Conv senza `salesMessages` associati          | Placeholder italico "Nessun messaggio ancora. Scrivi il primo sotto."                                                                   | `ConversationPane`                                           |
| `mobile-list`           | `lg:hidden` + no `:id`                        | Lista full-width; tap su riga fa navigate alla conv                                                                                     | `ConversationListColumn` mobile                              |
| `mobile-thread`         | `lg:hidden` + `:id`                           | Thread full-screen con back button `ChevronLeft`                                                                                        | `ConversationPane` mobile                                    |
| loading                 | —                                             | Non implementato (dati sincroni da mock)                                                                                                | —                                                            |
| error                   | —                                             | Non implementato (no try/catch, no ErrorBoundary locale)                                                                                | —                                                            |

---

## 3. Dati & Sorgenti

| Entità UI                     | Mock (src/data/mock/sales.ts)                            | Schema reale                                             | Note                                                                                                      |
| ----------------------------- | -------------------------------------------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Thread riga lista             | `salesConversations` (6 seed, `SalesConversation`)       | `communications.conversations`                           | `channel` ∈ {email, whatsapp, phone_note}; chiavi: `leadId`, `companyId`, `contactId`, `assignedToUserId` |
| Messaggi thread               | `salesMessages` (25 seed, filtrati per `conversationId`) | `communications.messages`                                | `direction` in/out, `deliveryStatus` ∈ {sent, delivered, read, failed}                                    |
| Azienda header + avatar       | `getCompanyById(conv.companyId)`                         | `core.companies`                                         | Fornisce `businessName` per initials/avatar deterministico (`avatarColor` hash)                           |
| Contatto header               | `getContactById(conv.contactId)`                         | `core.contacts`                                          | `fullName`, `role`, `phones[]`, `emails[]` → quick actions                                                |
| Lead collegata (deep link)    | `getLeadsByCompanyId(company.id)[0]`                     | `sales.leads`                                            | Apre `/sales/lead/:id` tramite `ExternalLink`                                                             |
| Banner WABA                   | `salesWabaConfig.sharedPhoneNumber`                      | `communications.waba_config` (o `core.org_integrations`) | Provider `360dialog`, routing via webhook n8n                                                             |
| `SALES_NOW_ISO` per `timeAgo` | mock constante                                           | `now()` runtime                                          | In prod sostituire con `Date.now()`                                                                       |

---

## 4. Pulsanti & Azioni

| Azione                                 | Trigger UI                  | Effetto attuale (mock)                                                                                       | Effetto target (produzione)                                                                                                                        | Gated da                                                    |
| -------------------------------------- | --------------------------- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| Selezione conversazione                | Tap riga lista              | `navigate('/sales/conversazioni/:id')` + `unreadCount = 0` in-memory                                         | Update `conversations.read_at`, broadcast a Realtime, stop notifiche push                                                                          | —                                                           |
| Filtro tab (all/unread/whatsapp/email) | Click chip tab              | `setFilterTab` state locale                                                                                  | Idem (no server)                                                                                                                                   | —                                                           |
| Search                                 | Input field                 | Filtro client-side su `businessName/fullName/subject`                                                        | Idem + fallback full-text su body (Postgres `tsvector`)                                                                                            | —                                                           |
| Clear search                           | `X` icon                    | `setQuery('')`                                                                                               | Idem                                                                                                                                               | —                                                           |
| Call                                   | Icon `Phone` header         | `tel:<phoneClean>` nativo                                                                                    | Idem + log call intent in `communications.call_attempts`                                                                                           | contact.phone non vuoto                                     |
| WhatsApp web/app                       | Icon `MessageCircle` header | `https://wa.me/<phoneClean>` new tab                                                                         | Apre thread WABA interno (no leak su app consumer) + pre-fill                                                                                      | contact.phone non vuoto                                     |
| Email client                           | Icon `Mail` header          | `mailto:<email>`                                                                                             | Apre composer interno con template auto-selezionato                                                                                                | contact.email non vuoto                                     |
| Vai alla lead                          | Icon `ExternalLink` header  | `navigate('/sales/lead/:leadId')`                                                                            | Idem                                                                                                                                               | `primaryLead` esiste                                        |
| Switch canale in composer              | Pill `Email`/`WhatsApp`     | `setChannel` locale, cambia placeholder e colore accent                                                      | Idem + validazione template WABA approvato se fuori finestra 24h                                                                                   | Canali abilitati in `org_integrations`                      |
| Invia messaggio                        | `Send` icon / Enter         | Push in `salesMessages`, aggiorna `conv.lastMessageAt` e `conv.channel`, `deliveryStatus = "delivered"` mock | Edge function `send-message` → provider (SendGrid/Postmark per email, 360dialog per WA) → INSERT in `communications.messages` → Realtime broadcast | draft non vuoto, channel abilitato, sender owner o director |
| Allega (paperclip)                     | Icon                        | **No-op** (nessun handler)                                                                                   | Upload su Storage, inserimento `attachments[]` nel message                                                                                         | MIME/size                                                   |
| Emoji                                  | Icon `Smile`                | **No-op**                                                                                                    | Apre emoji picker nativo/custom                                                                                                                    | —                                                           |
| Back (mobile)                          | `ChevronLeft`               | `navigate('/sales/conversazioni')`                                                                           | Idem                                                                                                                                               | mobile breakpoint                                           |

---

## 5. Componenti chiave

- [`ConversationListColumn`](../../../src/pages/sales/SalesConversazioni.tsx) — colonna sinistra: header con titolo+count+WABA banner, search, filter tabs, lista virtualizzabile
- [`ConversationPane`](../../../src/pages/sales/SalesConversazioni.tsx) — thread attivo: header cliente, messaggi con cluster & date divider, composer con channel picker
- [`EmptyConversation`](../../../src/pages/sales/SalesConversazioni.tsx) — illustrazione pane vuoto desktop
- [`Avatar`](../../../src/components/common/Avatar.tsx) — iniziali + colore deterministico da `companyId`
- [`EmptyState`](../../../src/components/sales/EmptyState.tsx) — stato vuoto lista con icona, titolo, subtitle, tone
- Helpers locali: `channelUI` (mapping canale → label/icon/accent), `chipCls`, `timeAgo`, `initialsFromName`, `avatarColor`

---

## 6. Integrazioni future

- **Supabase tables**: `communications.conversations`, `communications.messages`, `communications.message_attachments`, `communications.waba_config`, `core.org_integrations`
- **RLS policy**: `auth.uid() = assigned_to_user_id OR role IN ('director','admin','superadmin')`; POD director vede solo i thread del proprio POD (`pod_id = current_user.pod_id`)
- **Realtime channel**:
  - `conversations:user:<userId>` — update badge unread quando arriva nuovo inbound
  - `messages:conversation:<convId>` — append live dei nuovi messaggi + delivery status
  - `typing:conversation:<convId>` — broadcast "sta scrivendo" (ephemeral, 5s debounce)
- **Edge function**:
  - `send-message` → router multi-provider (email=Postmark/SendGrid, whatsapp=360dialog `/messages`, phone_note=INSERT diretto)
  - `webhook-waba-inbound` → riceve da 360dialog, matcha `from_phone` → contact → company → conversation (o crea), broadcast Realtime
  - `webhook-email-inbound` → parsing MIME (ricerca per `In-Reply-To` / `References`), threading per `subject` normalizzato
- **n8n workflow**: routing 360dialog inbound → lookup sales owner per lead → push a Supabase + notifica WhatsApp interno sales; retry delivery fallite
- **AI call**: Anthropic Claude Sonnet 4.5 su richiesta — "suggerisci risposta" partendo dal contesto ultimi 20 messaggi + scheda lead (prompt system: tono HireFast, max 80 parole, placeholder variabili {{first_name}}/{{company}})
- **Analytics event**: `conversation_opened`, `message_sent{channel,length,replyToId}`, `message_received{channel,latency_s}`, `conversation_filter_changed`, `quick_action_clicked{action}`

---

## 7. UX/UI review

### Funziona bene

- Layout 2 colonne desktop + fallback single-column mobile è idiomatico e familiare (pattern WhatsApp/iMessage)
- Cluster di messaggi con gap 5 min e border-radius dinamico per primo/ultimo bubble: polish alto
- Delivery status check/double-check con tint amber per "read": chiaro a colpo d'occhio
- Channel picker nel composer con colore accent (indigo email / emerald whatsapp) guida l'occhio
- Banner WABA contestuale sopra la lista spiega il numero condiviso senza interrompere
- Quick actions nell'header (call, wa.me, mailto, deep-link lead) riducono context switching
- Avatar deterministico (hash del seed) evita colori randomici a ogni render
- Empty state differenziato per lista vuota vs ricerca senza risultati
- Mark-as-read automatico all'apertura del thread (mutazione in-memory con `setTick` per rerender)

### Attriti UX attuali

- **Nessun typing indicator**, nessun "online", nessun timestamp ultimo accesso contatto
- **Nessuna indicazione streaming** per messaggi in invio (`sent` vs `delivered` vs `read` sono istantanei mock)
- **No read receipts inbound**: quando l'azienda legge un outbound non c'è transizione visiva (delivery hardcoded a `"delivered"`)
- Composer non supporta **attach/emoji reali**, bottoni no-op ingannano l'utente
- **Nessun threading email vero**: subject è flat, reply non incatena
- Filter tabs senza contatori oltre "unread"; un contatore sulle tab WhatsApp/Email aiuterebbe triage
- Lista non virtualizzata: con 200+ conversazioni React render di tutto (`scroll-thin` senza windowing)
- **Mutazione diretta** `conv.unreadCount = 0` è anti-pattern React (serve store)
- Nessuna ricerca full-text sul body (solo `subject`/company/contact)
- Mobile: back button navigate alla lista perdendo scroll position e filtro attivo
- Nessun **multi-select** per archiviare/assegnare conversazioni in bulk
- Nessuna **badge di canale aggregata** se un cliente ha sia email che whatsapp aperti
- Input textarea `maxHeight 140px` senza indicazione quando il testo supera il limite

### Coerenza UI

- Tipografia: ok (font-serif per titoli vuoti, sans per UI)
- Spacing: ok (padding coerente con altre pagine sales)
- Colore: ok — accent canali coerenti con `channelUI`; però la tint amber su "read" check è fuori palette standard
- Mobile-first: ok per la pagina base, ma la lista header ha 3 righe (titolo+banner+search+tabs) che su smartphone rubano real estate utile

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo | Proposta                                                                                                                                                                                      | Effort |
| --- | -------- | ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0       | UX   | **Typing indicator** inbound+outbound con animazione 3 puntini; broadcast via Realtime `typing:conversation:<id>` con debounce 1s ed autoreset a 5s                                           | M      |
| 2   | P0       | UX   | **Streaming delivery** reale: bubble outbound parte in stato `sending` (opacity 0.6 + spinner micro), transiziona a `sent` → `delivered` → `read` via Realtime; animazione check→double-check | M      |
| 3   | P0       | Arch | Sostituire mutazioni in-place (`conv.unreadCount = 0`, `salesMessages.push`) con uno **store Zustand / useReducer** e ottimistic update con rollback su error                                 | L      |
| 4   | P1       | UX   | **Composer unified multi-channel**: chip canale con validazione WABA 24h-window + template selector auto; warning se fuori finestra e nessun template approvato selezionato                   | M      |
| 5   | P1       | UX   | **Attach & Emoji reali**: file picker con preview, upload a Storage, emoji picker (emoji-mart o custom)                                                                                       | M      |
| 6   | P1       | UI   | **Virtualizzazione lista** (`@tanstack/react-virtual`) e windowing messaggi (scroll infinito caricando batch 50 messaggi)                                                                     | M      |
| 7   | P1       | UX   | **Quick reply AI** button sopra il composer: genera 2-3 bozze via Claude Sonnet basate su contesto thread + scheda lead; one-tap per inserire nel draft                                       | M      |
| 8   | P2       | UI   | **Presence & online indicator**: pallino verde su avatar se contact ha visto nell'ultima ora (da webhook WABA status)                                                                         | M      |
| 9   | P2       | UX   | **Threading email reale** via `In-Reply-To`/`References`; collapse dei reply quoted; pulsante "mostra history"                                                                                | L      |
| 10  | P2       | UI   | **Conteggi per filter tab** (WhatsApp 12 · Email 8); badge "assigned to me" per POD director                                                                                                  | S      |
| 11  | P3       | UX   | **Multi-select lista**: long-press (mobile) / shift-click (desktop) per archiviare, riassegnare owner, marcare letto in bulk                                                                  | L      |
| 12  | P3       | A11y | `aria-live="polite"` sui nuovi messaggi inbound, focus trap corretto su sheet mobile, screen-reader label per delivery status                                                                 | S      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- Qual è la finestra 24h WABA gestita lato prodotto? (blocca l'invio free-form fuori finestra o fallback automatico su template approvato?)
- Email multipla per azienda: il thread è per `contactId` singolo — cosa succede se lo stesso company ha CC multipli?
- `phone_note` è read-only (log interno) o il sales può crearne di nuove dal composer?
- RLS per POD director: vede i thread dei venditori del proprio POD o anche di altri POD della stessa region?
- Quote/media inline (PDF proposte, video walkthrough) devono renderizzarsi nel bubble o aprire preview lateral?
- Edge case: cambio owner del lead → le conversazioni esistenti seguono il nuovo owner (assigned_to) o restano storicizzate?
- Search sul body: dove indicizziamo (Postgres FTS vs Typesense vs Meilisearch)?
