# Performance POD

**Route**: `/sales/performance-pod`
**File**: [src/pages/sales/SalesPerformancePod.tsx](../../../src/pages/sales/SalesPerformancePod.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: POD Director (gating `isDirector(CURRENT_SALES_USER_ID)` → redirect `/sales` altrimenti)

---

## 1. Overview

Cruscotto mensile del POD Milano Alpha destinato al Director per monitorare pipeline aggregata e performance individuale dei sales. Combina KPI top-line (revenue, meeting, contratti, qualificati), breakdown per stato lead e ranking team con percentuale verso target. Punto di ingresso rapido per decisioni di riequilibrio carico, coaching e stimolo al closing.

---

## 2. Viste & Stati

Pagina a vista singola, sempre popolata (mock). Nessuna gestione esplicita di loading/error: a produzione vanno aggiunti skeleton + retry.

| ID vista         | Trigger                          | Descrizione                                                     | Componenti rilevanti                      |
| ---------------- | -------------------------------- | --------------------------------------------------------------- | ----------------------------------------- |
| default          | `isDirector === true`            | KPI grid 2x2/4x1 + breakdown stati + alert stale + ranking team | `IOSCard`, `GroupedList`, `Avatar`        |
| forbidden        | `isDirector === false`           | `<Navigate to="/sales" replace />`                              | —                                         |
| stale-alert      | `worstUser[1] >= 2`              | Banner rosso con CTA "Riequilibra"                              | `IOSCard` bordo rosso, `IOSButton` tinted |
| no-stale         | nessun sales ha ≥2 lead ferme    | Banner non renderizzato                                         | —                                         |
| empty (futuro)   | POD senza sales o senza pipeline | Mancante — mostrerebbe vuoto silente                            | `EmptyState` (da aggiungere)              |
| loading (futuro) | fetch Supabase                   | Mancante — render immediato da mock                             | skeleton card                             |
| error (futuro)   | errore RPC KPI                   | Mancante                                                        | toast + retry                             |

---

## 3. Dati & Sorgenti

| Entità UI                | Mock (src/data/mock/sales.ts)                                                                                 | Schema reale                                                                     | Note                                                                                |
| ------------------------ | ------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Lista POD                | `salesUsers.filter(podTeamName === "Milano Alpha")`                                                           | `core.users` + `sales.pod_teams` join                                            | In prod filtrare per `pod_team_id` dell'utente corrente, non string match           |
| Leads MTD                | `salesLeads.length`                                                                                           | `sales.leads` WHERE `created_at` in mese corrente                                | Mock prende TUTTI i lead senza filtro temporale — va aggiunto `date_trunc('month')` |
| Meeting MTD              | `salesAppointments.filter(status==='completed')`                                                              | `sales.appointments` WHERE `status='completed'` e `completed_at` in mese         | Stesso bug di perimetro temporale                                                   |
| Contratti                | `getLeadsByStatuses(['customer','awaiting_signature','contract_drafting'])`                                   | `sales.leads` stati chiusura                                                     | Conteggio cumulativo, non MTD                                                       |
| Revenue MTD              | `Σ salesQuarterlyCommissions.currentRevenue`                                                                  | `finance.commissions_mv` o `contracts.contracts SUM(amount)`                     | In prod usare vista MV aggregata per mese                                           |
| Qualificati MTD          | `getLeadsByStatuses(['qualified','market_investigation','closer_appointment','proposal_sent','negotiation'])` | `sales.leads.status`                                                             | Etichetta "MTD" fuorviante (è totale)                                               |
| Breakdown stato          | `getLeadsByStatuses([...])` × 7 buckets                                                                       | `sales.leads GROUP BY status`                                                    | Meglio single query con `COUNT(*) FILTER (WHERE status=...)`                        |
| Lead ferme               | `getStaleLeads()`                                                                                             | Vista `sales.v_stale_leads` (da creare: last_activity_at > threshold per status) | Soglia hardcoded lato client                                                        |
| Revenue/target per sales | `salesQuarterlyCommissions.find(userId)`                                                                      | `finance.quarterly_commissions`                                                  | `ratePercentage`/`earnedAmount` presenti ma non mostrati qui                        |

---

## 4. Pulsanti & Azioni

| Azione               | Trigger UI                                 | Effetto attuale (mock)                       | Effetto target (produzione)                                                                 | Gated da     |
| -------------------- | ------------------------------------------ | -------------------------------------------- | ------------------------------------------------------------------------------------------- | ------------ |
| Apri dettaglio sales | Tap riga team in `GroupedList`             | `navigate('/sales/pipeline?owner=<userId>')` | Stesso + preserve scroll + tracking event                                                   | `isDirector` |
| Riequilibra carico   | Tap bottone "Riequilibra" nel banner stale | Nessuno (placeholder)                        | Apre bottom sheet con suggerimento redistribuzione lead (algoritmo load-balance) + conferma | `isDirector` |
| KPI card             | Tap                                        | Nessuno                                      | Drill-down a pagina dedicata con trend storico e scomposizione                              | —            |
| Breakdown stato chip | Tap                                        | Nessuno                                      | Filtro rapido verso `/sales/pipeline?status=<stato>`                                        | —            |

---

## 5. Componenti chiave

- [IOSNavBar](../../../src/components/ios/IOSNavBar.tsx) — titolo + sottotitolo periodo
- [IOSCard](../../../src/components/ios/IOSPrimitives.tsx) — container KPI/breakdown/alert
- [IOSButton](../../../src/components/ios/IOSPrimitives.tsx) — CTA "Riequilibra"
- [GroupedList](../../../src/components/ios/GroupedList.tsx) — ranking team con trailing custom (progress bar + %)
- [Avatar](../../../src/components/common/Avatar.tsx) — iniziali + colore brand
- `AlertTriangle` (lucide) — icona banner stale

---

## 6. Integrazioni future

- **Supabase tables**: `sales.leads`, `sales.appointments`, `sales.pod_teams`, `finance.commissions`, `contracts.contracts`
- **Supabase views (da creare)**:
  - `sales.v_pod_kpis_mtd(pod_team_id, month)` — KPI top aggregati
  - `sales.v_pod_funnel(pod_team_id, month)` — breakdown stato
  - `sales.v_stale_leads(threshold_days)` — lead ferme per owner
  - `finance.v_team_commission_progress(quarter)` — revenue/target/tier per user
- **RLS policy**: POD Director può leggere solo `pod_team_id` di cui è `director_user_id`; Admin vede tutto
- **Realtime channel**: `pod:{pod_id}:kpis` → push counter su insert/update di `sales.leads`/`sales.appointments`
- **Edge function**: `compute-pod-kpis-mtd` (cache 5 min via Upstash)
- **n8n workflow**: notifica Slack al Director se un sales cade sotto 40% target a metà trimestre
- **AI call**: suggerimento redistribuzione lead per pulsante "Riequilibra" (input: carico corrente per sales + storico conversion rate)
- **Analytics event**: `pod_performance_viewed`, `pod_sales_drilldown`, `pod_rebalance_clicked`, `pod_kpi_drilldown`

---

## 7. UX/UI review

### Funziona bene

- Gerarchia visiva chiara: hero KPI → funnel → alert → team ranking
- Uso tabular-nums + font serif per i valori: leggibilità a colpo d'occhio
- Breakdown orizzontale scrollabile su mobile + griglia 7-col su desktop
- Banner stale condizionale (soglia ≥2 lead) evita rumore
- Progress bar trailing nel ranking comunica target a-vista

### Attriti UX attuali

- Sottotitolo "Aprile 2026" hardcoded → disallineato con selettore periodo (inesistente)
- "MTD" nei label ma dati sono in realtà totali/Q2: disonesto
- Nessuna delta vs periodo precedente (trend ↑/↓ mancante)
- Click su KPI card non fa nulla: aspettativa di drill-down tradita
- Banner stale nomina solo il "peggior sales", perde contesto sul perimetro totale (es. "5 lead ferme totali su 3 sales")
- Nessun sparkline/andamento giornaliero
- Nessun confronto vs altri POD (quando esisteranno)

### Coerenza UI

- Tipografia: ok (font-serif per numeri, eyebrow uppercase tracking)
- Spacing: ok (`space-y-5`, `gap-3`)
- Colore: ok — semantica ink/orange/green/teal coerente col design system
- Mobile-first: ok — grid 2→4 col responsive, scroll orizzontale per breakdown

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo | Proposta                                                                                                          | Effort |
| --- | -------- | ---- | ----------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0       | UX   | Aggiungere selettore periodo (MTD / Settimana / Q2 / custom) — oggi labels mentono sul perimetro temporale        | M      |
| 2   | P0       | Data | Filtrare correttamente `salesLeads`/`salesAppointments` per mese corrente; separare "revenue MTD" da "revenue Q2" | M      |
| 3   | P1       | UX   | Delta vs periodo precedente sulle 4 KPI card (↑12% vs marzo)                                                      | M      |
| 4   | P1       | UX   | Drill-down al tap su KPI card (apre grafico trend + scomposizione per sales)                                      | M      |
| 5   | P1       | UI   | Skeleton loading + error state con retry (sostituisce render immediato)                                           | S      |
| 6   | P2       | UX   | Sparkline mini-chart nelle KPI card (ultimi 14 giorni)                                                            | M      |
| 7   | P2       | UX   | Sezione "anomaly detection" — evidenzia KPI con variazione >2σ rispetto alla baseline POD                         | L      |
| 8   | P2       | UX   | Export CSV del ranking team (per review mensili offline)                                                          | S      |
| 9   | P3       | UI   | Confronto vs altri POD quando ne esisterà più di uno (tabella multi-pod per Admin)                                | L      |
| 10  | P3       | UX   | Suggerimento AI nel banner "Riequilibra" con redistribuzione proposta e impact stimato                            | XL     |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- Soglia stale-lead (ora `getStaleLeads()` implicita): va configurabile per status e per POD?
- Definizione ufficiale di "revenue MTD": fatturato, incassato, firmato? Serve allineamento con finance
- Criterio "target" per commission: è quarterly o mensile? La vista lo mescola
- Per l'azione "Riequilibra": decisione manuale del Director o proposta AI con approvazione? Quale audit log richiesto?
- Quando ci saranno più POD: questa pagina diventa per-pod o globale con filtro?
- "Qualificati MTD" include `proposal_sent`/`negotiation`: è intenzionale (pipeline calda) o errore di copy (va chiamato "Pipeline Attiva")?
