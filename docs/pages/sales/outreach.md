# Outreach

**Route**: `/sales/outreach`
**File**: [src/pages/sales/SalesOutreach.tsx](../../../src/pages/sales/SalesOutreach.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: tutti (ownerUserId per le azioni mutate; director/admin per vedere campagne di altri)

---

## 1. Overview

Cockpit delle **campagne cold-outreach multi-step W1→W4** (scraping → cold drip → follow-up → qualified): ogni sales può attivare workflow di prospecting automatizzato filtrato per settore/regione/size e monitorarne il funnel (scraped → sent → open → reply → qualified → meeting). La pagina espone un KPI aggregato in testa e una lista per campagna con status chip (attiva/pausa/bozza/completata); il tap apre un BottomSheet con mini-funnel e tre azioni di governance (pausa/modifica/duplica). È il complemento strategico alle Conversazioni: le campagne generano i lead che poi confluiscono nell'inbox.

---

## 2. Viste & Stati

| ID vista        | Trigger                        | Descrizione                                                                                                                     | Componenti rilevanti                                         |
| --------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `default`       | `salesCampaigns.length > 0`    | Funnel aggregato (4 step) + lista campagne con chip status e dot di colore                                                      | `IOSCard`, `GroupedList`, `Chip`                             |
| `empty`         | `salesCampaigns.length === 0`  | EmptyState tone=positive "Nessuna campagna attiva" + CTA "Nuova campagna"                                                       | [`EmptyState`](../../../src/components/sales/EmptyState.tsx) |
| `detail-sheet`  | `selectedId !== null`          | BottomSheet con mini-funnel (6 metriche: scraped/sent/open%/reply%/qualified/meeting) + 3 CTA (pausa-riprendi/modifica/duplica) | `BottomSheet`, `IOSButton`                                   |
| `detail-paused` | `campaign.status === "paused"` | CTA primaria diventa "Riprendi" con icon `Play`                                                                                 | `IOSButton` tone=primary                                     |
| `detail-active` | `campaign.status === "active"` | CTA primaria "Pausa" con icon `Pause` e tone=orange                                                                             | `IOSButton` tone=orange                                      |
| loading         | —                              | Non implementato (sync mock)                                                                                                    | —                                                            |
| error           | —                              | Non implementato                                                                                                                | —                                                            |

---

## 3. Dati & Sorgenti

| Entità UI                 | Mock (src/data/mock/sales.ts)                                     | Schema reale                                                            | Note                                                                     |
| ------------------------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Lista campagne            | `salesCampaigns` (4 seed, `SalesCampaign`)                        | `campaigns.outreach_campaigns`                                          | Stato: draft/active/paused/completed                                     |
| Stats per campagna        | `campaign.stats` (`SalesCampaignStats`)                           | `campaigns.outreach_stats` (materialized view o tabella denormalizzata) | 6 counter: scraped, sent, opens, replies, qualifiedLeads, meetingsBooked |
| Target sector/region/size | `targetSector`, `targetRegion`, `targetSize`, `targetPositions[]` | `campaigns.outreach_targets` (JSONB filters)                            | Base per lookup scraping                                                 |
| Volume/finestra           | `dailyVolume`, `timeWindow`                                       | `campaigns.send_config`                                                 | Deliverability: rispetto orari 09-11/14-16                               |
| Template email            | `emailSubjectTemplate`, `emailBodyTemplate`                       | `campaigns.email_templates` (o FK a `sales.email_templates`)            | Placeholder `{{first_name}}`, `{{business_name}}`, `{{region}}`          |
| Owner                     | `ownerUserId`                                                     | `auth.users.id` / `core.users.id`                                       | Per RLS                                                                  |
| Funnel aggregato          | `reduce` locale su `salesCampaigns`                               | Vista aggregata lato server (`campaigns.funnel_by_owner`)               | Riduce payload, consente cache                                           |

---

## 4. Pulsanti & Azioni

| Azione         | Trigger UI                           | Effetto attuale (mock)              | Effetto target (produzione)                                                                | Gated da         |
| -------------- | ------------------------------------ | ----------------------------------- | ------------------------------------------------------------------------------------------ | ---------------- |
| Nuova campagna | `IOSButton` "Nuova" nell'`IOSNavBar` | **No-op** (nessun handler)          | Apre wizard multi-step (target → audience size preview → template → schedule → review)     | Ruolo            |
| Apri detail    | Tap riga lista                       | `setSelectedId(c.id)` → BottomSheet | Idem + prefetch messaggi invio ultime 24h                                                  | —                |
| Pausa/Riprendi | CTA sheet                            | **No-op**                           | PATCH `campaigns/:id` status=paused/active → edge function sospende cron n8n scraping/send | Owner o director |
| Modifica       | CTA sheet                            | **No-op**                           | Apre editor full-screen (target filters, template, schedule)                               | Owner o director |
| Duplica        | CTA sheet                            | **No-op**                           | POST clone con `status=draft` e `startedAt=null`, copia template                           | Owner o director |
| Chiudi sheet   | Tap fuori / handle                   | `setSelectedId(null)`               | Idem                                                                                       | —                |

---

## 5. Componenti chiave

- [`IOSNavBar`](../../../src/components/ios/IOSNavBar.tsx) — titolo "Outreach" + subtitle funnel + CTA
- [`IOSCard`](../../../src/components/ios/IOSPrimitives.tsx) — card funnel aggregato e mini-funnel
- [`GroupedList`](../../../src/components/ios/GroupedList.tsx) — lista campagne con leading icon, chevron, onClick
- [`BottomSheet`](../../../src/components/ios/IOSPrimitives.tsx) — detail campagna
- [`IOSButton`](../../../src/components/ios/IOSPrimitives.tsx) — CTA nav + sheet (variant `plain`/`tinted`/`filled`)
- [`Chip`](../../../src/components/common/Chip.tsx) — status chip (green/yellow/soft/ink)
- [`EmptyState`](../../../src/components/sales/EmptyState.tsx) — stato vuoto con action
- Helpers locali: `statusMeta` (mapping CampaignStatus → label/chip/dot), `funnelSteps` derivati dal reduce

---

## 6. Integrazioni future

- **Supabase tables**: `campaigns.outreach_campaigns`, `campaigns.outreach_leads_discovered`, `campaigns.outreach_messages`, `campaigns.outreach_events`, `campaigns.outreach_stats` (mview)
- **RLS policy**: `owner_user_id = auth.uid() OR role IN ('director','admin')`; POD director scope filtrato per `pod_id`
- **Realtime channel**: `campaign:owner:<userId>` per aggiornare counter live durante invii massivi; `campaign:<id>:events` per stream open/reply istantaneo (toast "Reply da ACME")
- **Edge function**:
  - `campaign-start` → valida config, crea job n8n schedule
  - `campaign-pause` / `campaign-resume` → flip status + freeze send queue
  - `campaign-send-next-batch` → cron ogni ora dentro `timeWindow`, legge coda, manda via provider email (SendGrid/Postmark/Mailgun), rispetta daily volume cap e warm-up
  - `campaign-track-event` → webhook provider (open/click/reply/bounce) → UPDATE stats
- **n8n workflow**:
  - `outreach-scraper` — query Apollo/Lusha/Clearbit per target filters, dedupe vs `core.companies`
  - `outreach-sender` — consuma coda, personalizza template, invia, logga event
  - `outreach-reply-router` — parse reply inbound, se positivo crea `sales.lead` e apre `communications.conversation` (handoff ad Conversazioni)
- **AI call**: Anthropic Claude Sonnet per (a) generazione variant A/B del `emailSubjectTemplate`/`emailBodyTemplate` partendo da brief, (b) classificazione sentiment delle reply (positive/negative/ooo/unsubscribe), (c) suggerimento target lookalike basato sui lead convertiti dalla campagna
- **Analytics event**: `campaign_created`, `campaign_started`, `campaign_paused`, `campaign_duplicated`, `campaign_opened_detail`, `funnel_metric_viewed{step}`

---

## 7. UX/UI review

### Funziona bene

- Gerarchia chiara: KPI card in top → lista scrollabile → detail non intrusivo (sheet invece di page nav)
- Funnel aggregato con **barre proporzionali al max** (normalizzate) dà colpo d'occhio su dove cala il funnel
- Status chip + dot ring su icon leading replica il pattern iOS (coerente col resto del portale)
- `tabular-nums` su metriche evita jitter di allineamento
- Mini-funnel nel sheet mostra **open% e reply%** calcolati (non solo counter assoluti): utile per deliverability check
- Subtitle navbar "W1 → W2 → W3 → W4" racconta la metodologia a colpo d'occhio

### Attriti UX attuali

- **CTA "Nuova" è no-op**: l'utente ci clicca e non succede niente (disappointing UX)
- **Pausa/Modifica/Duplica nel sheet sono tutte no-op**: manca l'azione core della pagina
- Nessun filtro per **stato** sulla lista (tutte le 4 campagne visibili in ogni momento)
- Nessuna **timeline/calendar heatmap** per vedere quando arrivano open/reply nel giorno
- Funnel aggregato usa `max(values, 1)` come denominatore: se scraped=420 e qualified=9 le barre più basse sono quasi invisibili (2% vs 100%)
- Mini-funnel 6 metriche in 3 colonne su mobile: layout OK ma perde gerarchia (tutte uguali di peso visivo)
- Nessun **benchmark** (es. "open 44% · target 35%") per capire se la campagna performa
- `ownerUserId` non visibile: POD director non sa chi gestisce la campagna
- Nessun preview del template email (subject/body) nel sheet
- Nessuna indicazione del **prossimo invio schedulato** ("Prossimo batch oggi alle 14:00")
- `startedAt` presente nei dati ma non esposto in UI
- Chip "draft"/"completed" usa variant ink (scuro) che visivamente pesa come "active"

### Coerenza UI

- Tipografia: ok — font-serif sulla metrica grande (28px) coerente col linguaggio iOS premium
- Spacing: ok — `space-y-5` e `gap-4` uniformi
- Colore: ok — status palette standard (green/yellow/soft/ink); però la chip "completed" con variant=ink è poco distinguibile da "paused" alla sintesi
- Mobile-first: ok — grid 2 colonne su mobile, 4 su md+

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo     | Proposta                                                                                                                                                                        | Effort |
| --- | -------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0       | UX       | **Wiring CTA Nuova campagna**: apri wizard multi-step (target filters con audience size live, template editor con variabili, schedule+cap, review); mockup acceptable in fase 1 | L      |
| 2   | P0       | UX       | **Azioni sheet funzionanti** (pausa/modifica/duplica) anche solo con mutation mock + toast, così il flusso demo diventa credibile                                               | M      |
| 3   | P1       | UX       | **Filtro status** sulla lista (SegmentedControl: Tutte/Attive/In pausa/Bozze/Completate) e ricerca per nome campagna                                                            | S      |
| 4   | P1       | UI       | **Preview template** nel BottomSheet (subject + body troncato con variabili evidenziate in teal come in TemplateEmail)                                                          | S      |
| 5   | P1       | UX       | **Benchmark/target** accanto alle metriche (open% vs baseline 30%, reply% vs 5%) con dot verde/rosso                                                                            | M      |
| 6   | P1       | UX       | **Timeline prossimi invii** ("Oggi 14:00 · batch 40/80 · 40 rimanenti") + stop se fuori finestra                                                                                | M      |
| 7   | P2       | UX       | **Funnel normalizzato con conversion rate tra step** (scraped → sent 90%, sent → open 44%, open → reply 17%), non solo barre assolute                                           | M      |
| 8   | P2       | UX       | **Owner chip** con avatar nelle righe lista per director view multi-owner                                                                                                       | S      |
| 9   | P2       | UX       | **A/B variants** nel detail: due versioni subject/body con performance comparate                                                                                                | L      |
| 10  | P3       | UI       | **Mini sparkline** 7gg su ogni riga (open/reply trend)                                                                                                                          | M      |
| 11  | P3       | Realtime | Toast live quando arriva una **nuova reply positiva** ("Luca di XYZ ha risposto — apri")                                                                                        | M      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- Provider email scelto per invio cold (SendGrid vs Postmark vs Mailgun vs Amazon SES)? Implica gestione dedicated IP e warm-up
- Cold outreach legale: come gestiamo **unsubscribe / GDPR / CAN-SPAM** link e suppression list globale?
- Scraping: usiamo Apollo.io / Lusha / Clearbit o scraping proprietario? Costo per lead condiziona `dailyVolume` max
- `timeWindow` è stringa libera ("09:00-11:00 / 14:00-16:00"): formato strutturato o parser custom?
- Handoff a Conversazioni: reply positiva crea automaticamente lead+conversation o richiede approvazione sales?
- RLS director: può **modificare** campagne di altri owner del suo POD o solo visualizzarle?
- Cap su invii giornalieri a livello org (es. 500/giorno su tutto il sales team) per proteggere domain reputation?
