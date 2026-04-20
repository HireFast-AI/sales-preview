# AI Copilot

**Route**: `/sales/ai-copilot`
**File**: [src/pages/sales/SalesAICopilot.tsx](../../../src/pages/sales/SalesAICopilot.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: tutti i ruoli sales (contesto filtrato per owner; director/admin vedono contesto POD)

---

## 1. Overview

Assistente conversazionale per il sales che combina **il contesto operativo del venditore** (pipeline, lead, storico conversazioni, pattern POD) con un'interfaccia chat familiare. La pagina parte da un `HeroAI` con quick-prompts ("Qualifica un lead caldo", "Riassumi la settimana"), propone una griglia di 6 **prompt tile** preconfezionati (qualifica, follow-up, market research, next action, closer script, riassunto) e mantiene un thread con bubble user/bot più composer sticky in fondo. Oggi la risposta è una stringa canned mock; in produzione ogni prompt innesca una chiamata strutturata ad Anthropic Claude con tool-calling sullo stato Supabase dell'utente. È l'ingresso principale a qualsiasi automazione "AI-assisted" del portale sales.

---

## 2. Viste & Stati

| ID vista            | Trigger                   | Descrizione                                                                                             | Componenti rilevanti             |
| ------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------- | -------------------------------- |
| `default`           | Default mount             | HeroAI + 6 prompt tile + thread con 3 messaggi iniziali (2 user + 1 bot)                                | `HeroAI`, `IOSCard`, bubble chat |
| `typing-user`       | Utente scrive in textarea | Auto-resize fino 140px, send button si abilita                                                          | textarea                         |
| `after-send`        | User invia                | Push messaggio user + messaggio bot canned immediato                                                    | `setMessages` state              |
| `after-prompt-tile` | Click prompt tile         | Invia il `p.prompt` come se l'utente l'avesse scritto                                                   | stesso handler `send()`          |
| `after-quick-chip`  | Click chip hero           | Stessa logica dei prompt tile, prompt "Qualifica Logistica Veloce" o "Riassumi la settimana di vendite" | `HeroAI`                         |
| empty               | —                         | Non esiste "thread vuoto": `initialMessages` ha sempre 3 messaggi hardcoded                             | —                                |
| loading             | —                         | Non implementato (risposta sincrona canned)                                                             | —                                |
| error               | —                         | Non implementato                                                                                        | —                                |
| streaming           | —                         | **Non implementato**: bot risponde istantaneamente con stringa fissa                                    | —                                |

---

## 3. Dati & Sorgenti

| Entità UI            | Mock (src/data/mock/sales.ts)                            | Schema reale                                                                           | Note                                         |
| -------------------- | -------------------------------------------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------- |
| Messaggi chat        | `initialMessages` const locale + `useState<Message[]>`   | `ai.copilot_threads` + `ai.copilot_messages`                                           | Persistere thread per utente con `thread_id` |
| Prompt tile          | `prompts` array locale (6 entry)                         | `ai.copilot_prompt_library`                                                            | Tile riutilizzabili, scoped a ruolo          |
| Quick chip hero      | Array inline in JSX                                      | Stesso schema dei prompt tile                                                          | Hardcoded oggi                               |
| Contesto lead (demo) | "Logistica Veloce", "l-014" citati nella canned response | Join runtime con `sales.leads`, `sales.conversations`, `sales.quotes`, `sales.pod_kpi` | Oggi hardcoded nella stringa                 |
| Utente corrente      | Implicito (nessun `CURRENT_SALES_USER_ID` usato qui)     | `core.users`                                                                           | Usato in prod per scoping RLS e tool-calling |

---

## 4. Pulsanti & Azioni

| Azione                   | Trigger UI           | Effetto attuale (mock)                             | Effetto target (produzione)                                                  | Gated da          |
| ------------------------ | -------------------- | -------------------------------------------------- | ---------------------------------------------------------------------------- | ----------------- |
| Invia messaggio          | Click `Send` / Enter | `setMessages([...m, user, bot-canned])` istantaneo | POST `ai/copilot/stream` → SSE con tokens → append incrementale              | input non vuoto   |
| Shift+Enter              | Keyboard             | Nuova riga (browser default)                       | Idem                                                                         | —                 |
| Prompt tile              | Click card           | `send(p.prompt)` — stesso flusso canned            | Invia prompt strutturato con context hint (tile id)                          | —                 |
| Quick chip hero          | Click button         | `send(hardcoded string)`                           | Idem                                                                         | —                 |
| Auto-resize textarea     | Input change         | `handleAutoResize` fino 140px                      | Idem                                                                         | —                 |
| Stop generazione         | —                    | **Non esistente**                                  | Cancel SSE stream, marca ultimo bot message come "stopped"                   | Durante streaming |
| Copia risposta bot       | —                    | **Non esistente**                                  | `navigator.clipboard` sul text del bubble                                    | Hover bubble bot  |
| Feedback (thumb up/down) | —                    | **Non esistente**                                  | INSERT `ai.copilot_feedback{messageId, rating, reason}` per fine-tuning RLHF | —                 |
| Nuovo thread             | —                    | **Non esistente**                                  | Reset state, crea nuovo `thread_id`, persiste su Supabase                    | —                 |

---

## 5. Componenti chiave

- [`IOSNavBar`](../../../src/components/ios/IOSNavBar.tsx) — titolo "AI Copilot" + subtitle "Sales"
- [`HeroAI`](../../../src/components/ios/IOSPrimitives.tsx) — blocco scuro con slogan + quick chips
- [`IOSCard`](../../../src/components/ios/IOSPrimitives.tsx) — prompt tile con icon + title + desc
- Bubble chat — implementati inline nel file (non estratti): user = teal trasparente right-aligned, bot = paper bordered left-aligned con avatar `Sparkles`
- Composer sticky bottom — textarea + button invio circolare ink-filled
- Icons lucide: `Sparkles`, `Send`, `Mail`, `ArrowRight`, `Phone`, `MessageSquare`, `FileText`, `ClipboardCheck`, `Bot`

---

## 6. Integrazioni future

- **Supabase tables**: `ai.copilot_threads`, `ai.copilot_messages`, `ai.copilot_prompt_library`, `ai.copilot_tool_calls`, `ai.copilot_feedback`, `ai.copilot_tokens_usage` (billing)
- **RLS policy**: `thread.user_id = auth.uid()` (thread privato); `prompt_library` readable per tutti, writable admin; access esteso a director sul proprio POD con `can_view_subordinate_threads=true`
- **Realtime channel**: `copilot:thread:<threadId>` per streaming tokens (alternativa a SSE se serve multi-device sync) e `copilot:user:<userId>:notification` per alert async ("AI ha completato l'analisi in background")
- **Edge function**:
  - `copilot-stream` → riceve prompt + threadId, carica contesto (lead, pipeline, storico), chiama Anthropic Claude con tool-use e streaming, forwarda SSE al client, persiste message+tool-calls al completion
  - `copilot-summarize-week` → cron ogni lunedì mattina, pre-genera riassunto settimana per ogni sales, cached in `ai.copilot_cached_insights`
  - `copilot-qualify-lead` → tool chiamabile sia dal copilot sia da LeadDetail: carica scheda + storico e restituisce score 0-10 + razionali
- **n8n workflow**: `copilot-context-builder` — on-demand data loader che aggrega snapshot JSON (lead+storico+quote+kpi) richiamato via webhook dalla edge function prima del prompt finale
- **AI call**:
  - **Provider**: Anthropic — modello `claude-sonnet-4-5` come default (balance costo/qualità); escalation a `claude-opus-4-6` per richieste di analisi profonda ("analizza ricerca mercato", "script closer call complesso")
  - **Tipo di prompt**: **system prompt** + **tool-use** strutturato:
    - System: persona HireFast sales coach, tono diretto italiano, max 150 parole per risposta, cita sempre `[lead:id]` quando menziona entità
    - Tools esposti: `get_lead(leadId)`, `list_recent_conversations(userId, limit)`, `get_pod_kpi(podId, period)`, `search_email_templates(category)`, `get_quote(quoteId)`, `write_followup_email(leadId, tone)`, `suggest_next_action(leadId)`
    - Prompt caching: metti system prompt + tool definitions in cache (5-min TTL) per abbattere costi su iterazioni rapide
    - Extended thinking (budget 4000 token) per prompt complessi tipo "qualifica" e "script"
- **Analytics event**: `copilot_message_sent{promptSource:tile|chip|free}`, `copilot_stream_start`, `copilot_stream_complete{tokensIn,tokensOut,latencyMs}`, `copilot_tool_called{toolName}`, `copilot_feedback{rating,messageId}`, `copilot_stopped_by_user`

---

## 7. UX/UI review

### Funziona bene

- **HeroAI scuro** con chip rapide crea gerarchia visiva netta e invita all'azione
- Griglia di **6 prompt tile** abbassa il "blank page syndrome": il sales sa subito cosa può chiedere
- Layout flex-col + composer sticky in fondo è idiomatico per chat mobile-first
- Avatar `Sparkles` circolare sul lato bot dà identità visiva al Copilot
- Initial messages 3-message sample simula una conversazione in corso: ottimo per demo
- Auto-resize textarea (max 140px) con Enter-to-send / Shift+Enter-newline: convenzione standard chat
- Sample di risposta canned (`messages[1]`) è ricco di dettagli concreti (lead id, stato, urgenza, budget, azione suggerita): comunica chiaramente il valore atteso dell'AI

### Attriti UX attuali

- **Nessuno streaming di token**: bot risponde istantaneamente con stringa fissa che cita il prompt dell'utente. Feeling di "mock" evidente in demo live
- **Nessun loading state** tra invio e risposta (no "Copilot sta pensando..." o placeholder skeleton)
- **Nessun pulsante Stop**: se in prod la risposta è lunga (streaming) l'utente non può interromperla
- **Nessun messaggio di sistema visibile**: non si sa di quale modello/versione si parla né i tool a disposizione
- **Thread non persistito**: refresh perde tutto (oggi accettabile, ma manca qualsiasi sketch di persistenza)
- Nessun **auto-scroll** al fondo quando arrivano nuovi messaggi (i messaggi possono finire off-screen)
- Nessun **feedback thumbs up/down** sulle risposte del bot (bloccante per iterazione qualità)
- Nessuna **clear/nuova conversazione**: la chat cresce indefinitamente senza modo di resettare
- Prompt tile statici: non si adattano a ciò su cui l'utente sta lavorando (es. se è su /sales/lead/:id dovrebbero offrire "qualifica questo lead")
- **No context indicator**: l'utente non sa cosa il copilot "sa" di lui (pipeline? lead aperto? ricerca?)
- Bubble user è su sfondo teal trasparente — contrasto basso su light mode, accessibilità dubbia
- Composer non ha **placeholder dinamico** contestuale (es. "Chiedi sul lead XYZ che stai visualizzando…")
- Nessun support a **slash commands** (`/qualify l-014`, `/write-email formal`) per utenti power
- Nessuna gestione **markdown** nel body del bot (bold, liste, link cliccabili verso lead)
- Nessun link cliccabile `[lead:l-014]` → non si naviga alla scheda dal bubble

### Coerenza UI

- Tipografia: ok — font-serif per slogan hero, sans-body standard nei bubble
- Spacing: ok — `space-y-5`, `gap-3` uniforme con il resto
- Colore: ok — accent teal coerente, però il bubble user in `rgba(21,81,95,0.12)` è quasi identico al paper in certi contesti
- Mobile-first: ok — composer sticky, safe-area-inset rispettata, tap target 44px sul send button

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo | Proposta                                                                                                                                                                                      | Effort |
| --- | -------- | ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0       | UX   | **Streaming token reale** via SSE: bubble bot appare immediatamente con caret blinking, i token si appendono in batch 16 char per evitare jank; cursor a fine ultima parola durante streaming | L      |
| 2   | P0       | UX   | **Loading skeleton** pre-streaming ("Sta pensando..." con 3 dots animati) + pulsante `Stop` che cancella SSE e marca messaggio come "interrotto"                                              | M      |
| 3   | P0       | Arch | **Persistenza thread** su Supabase (`ai.copilot_threads` + `messages`); nuovo thread button, ripresa conversazione al reload, history paginata                                                | L      |
| 4   | P1       | UX   | **Auto-scroll-to-bottom** durante streaming (con flag "utente ha scrollato su" che disabilita l'auto-scroll e mostra chip "Nuovi messaggi ↓")                                                 | S      |
| 5   | P1       | UX   | **Markdown rendering nel bubble bot** (bold, italic, liste numerate, link cliccabili `[lead:l-014]` che fanno navigate a `/sales/lead/l-014`)                                                 | M      |
| 6   | P1       | AI   | **Tool-use visibile**: quando il copilot chiama `get_lead(l-014)` mostra un chip "Consulto scheda lead..." che diventa "Consultato ✓" quando completato — trasparenza e confidence            | M      |
| 7   | P1       | UX   | **Context chip** in alto al thread che mostra cosa il copilot sa di te ("Vedendo: pipeline Gabriele · 24 lead attivi · POD Milano Alpha")                                                     | S      |
| 8   | P1       | UX   | **Feedback thumbs up/down** sotto ogni risposta bot, con modale opzionale "cosa non andava?" → `ai.copilot_feedback`                                                                          | S      |
| 9   | P2       | UX   | **Prompt tile contestuali**: se provi da `/sales/lead/:id`, i 6 tile cambiano a "Qualifica questo lead", "Scrivi follow-up per XYZ", "Script call per oggi 14:30"                             | M      |
| 10  | P2       | UX   | **Slash commands** (`/qualify`, `/write`, `/script`, `/summary`) con autocomplete dropdown                                                                                                    | M      |
| 11  | P2       | UI   | **Delivery status** (sent/streaming/complete/stopped/error) inline in piccolo sotto il bubble utente                                                                                          | S      |
| 12  | P3       | UX   | **Session export**: download thread come markdown/PDF (utile per handoff in meeting)                                                                                                          | S      |
| 13  | P3       | A11y | `aria-live="polite"` su nuovo bubble bot, focus trap composer, screen-reader annotation per streaming                                                                                         | S      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- Budget token/utente/mese e policy di throttling quando si esauriscono (degradazione a Haiku? blocco hard?)
- Se il copilot può **eseguire azioni** (write email, update lead status) serve confirmation step esplicito o modalità "auto" per utenti trusted?
- Cross-pagina: ha senso mantenere **un solo thread globale** o thread separati per contesto (lead/campagna/pipeline)?
- Retention thread: 30gg? 90gg? Diritto utente a cancellare (GDPR)?
- Modello di default: `claude-sonnet-4-5` o `claude-opus-4-6`? Trade-off costo vs qualità per sales use-case
- Integrare **vector search** su knowledge base interna HireFast (playbook sales, case studies) come tool aggiuntivo?
- Copilot in **Conversazioni**: serve una modalità "assist" laterale al thread cliente (sidepanel) oltre alla pagina dedicata?
- Logging prompt/response per compliance: dove (Langfuse, Helicone, Supabase JSONB)? RLS per nascondere PII nei log?
- Offline/fallback: se Anthropic API down, mostriamo messaggio esplicito o cached answer?
