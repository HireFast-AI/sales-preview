# Alert Team

**Route**: `/sales/alert-team`
**File**: [src/pages/sales/SalesAlertTeam.tsx](../../../src/pages/sales/SalesAlertTeam.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: POD Director (gating `isDirector(CURRENT_SALES_USER_ID)` → redirect `/sales` altrimenti)

---

## 1. Overview

Centrale di controllo rischio del POD: aggrega alert operativi (lead non lavorate, appuntamenti scaduti, proposte ferme, follow-up mancati) con severity codificata (critical/high/medium/low) e gestione workflow aperti→ack→risolti. Permette al Director di intervenire prima che una pipeline marcisca, assegnare un segnale al sales responsabile e chiudere il loop con "Riconosci" o "Risolto". Pensata per essere la seconda schermata che il Director apre al mattino dopo la Home.

---

## 2. Viste & Stati

| ID vista               | Trigger                 | Descrizione                                                  | Componenti rilevanti               |
| ---------------------- | ----------------------- | ------------------------------------------------------------ | ---------------------------------- |
| open (default)         | `tab === 'open'`        | Alert non ack e non risolti                                  | `SegmentedControl`, card alert     |
| ack                    | `tab === 'ack'`         | Alert riconosciuti ma non ancora risolti                     | idem                               |
| resolved               | `tab === 'resolved'`    | Alert chiusi                                                 | idem                               |
| all                    | `tab === 'all'`         | Tutti gli alert senza filtro stato                           | idem                               |
| filters-open           | `showFilters === true`  | Card espandibile con select severity/type/sales              | `IOSCard`, `<select>` native       |
| empty                  | `filtered.length === 0` | "Tutto sotto controllo" con icona ShieldCheck, tone positive | `EmptyState`                       |
| detail                 | `detail != null`        | Bottom sheet con dettaglio alert + azioni Ack/Risolvi        | `BottomSheet`, `IOSButton`, `Chip` |
| forbidden              | `!isDirector`           | `<Navigate to="/sales" replace />`                           | —                                  |
| loading/error (futuri) | fetch realtime          | Non gestiti                                                  | skeleton + retry                   |

---

## 3. Dati & Sorgenti

| Entità UI          | Mock (src/data/mock/sales.ts)                       | Schema reale                                                                              | Note                                                                                           |
| ------------------ | --------------------------------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Lista alert        | `salesTeamAlerts`                                   | Nuova tabella `sales.team_alerts` (da creare) o view materializzata `sales.v_team_alerts` | Mock ha 12 alert, 3 critical + 4 high + 3 medium + 2 low                                       |
| Severity counters  | `counts.{critical,high,medium,low}` via `useMemo`   | `COUNT(*) FILTER (WHERE severity=...)` lato server                                        | Calcolato client-side su array intero                                                          |
| Stato counters     | `counts.{open,ack,resolved}`                        | idem                                                                                      | derivato da `acknowledgedAt`/`resolvedAt` null                                                 |
| Target user        | `targetUserId` + `targetUserName`                   | `sales.users` lookup                                                                      | Duplicato denormalizzato nel mock                                                              |
| Target company     | `companyName` (string denorm)                       | `core.companies.business_name` join via `leadId → lead.companyId`                         | Oggi è stringa, non id                                                                         |
| Alert type         | `AlertType` enum                                    | `sales.alert_types` lookup o enum colonna                                                 | `lead_unworked`/`lead_not_contacted`/`overdue_appointment`/`stale_proposal`/`missed_follow_up` |
| Severity           | `AlertSeverity` enum                                | enum colonna                                                                              | `critical`/`high`/`medium`/`low`                                                               |
| Threshold          | `thresholdValue` + `thresholdUnit` (giorni)         | jsonb `threshold_config` per regola                                                       | Metadata della regola che ha sparato l'alert                                                   |
| Timestamp          | `triggeredAt`, `acknowledgedAt`, `resolvedAt`       | colonne timestamptz                                                                       | OK                                                                                             |
| POD users (filter) | `salesUsers.filter(podTeamName === 'Milano Alpha')` | `sales.pod_team_members`                                                                  | Filter dropdown hardcoded su "Milano Alpha"                                                    |

---

## 4. Pulsanti & Azioni

| Azione                       | Trigger UI                      | Effetto attuale (mock)              | Effetto target (produzione)                                                                | Gated da     |
| ---------------------------- | ------------------------------- | ----------------------------------- | ------------------------------------------------------------------------------------------ | ------------ |
| Cambia tab stato             | Tap `SegmentedControl`          | `setTab(v)`                         | + query param `?tab=` per deep-link + analytics                                            | `isDirector` |
| Toggle filtri avanzati       | Tap bottone "Filtri avanzati"   | `setShowFilters(v)`                 | Stesso + persist in localStorage                                                           | —            |
| Cambia severity filter       | `<select>`                      | `setSeverityFilter`                 | + query param + tracking                                                                   | —            |
| Cambia type filter           | `<select>`                      | `setTypeFilter`                     | + query param                                                                              | —            |
| Cambia sales filter          | `<select>`                      | `setSalesFilter`                    | + query param                                                                              | —            |
| Refresh                      | Tap icona `RefreshCw` in navbar | Nessuno (bottone vuoto)             | Invalida query + refetch da `sales.team_alerts`                                            | `isDirector` |
| Apri dettaglio               | Tap card alert                  | `setDetail(a)` → apre `BottomSheet` | Stesso + preload entità correlate (lead, contatto, ultimo touch)                           | —            |
| Riconosci                    | Tap "Riconosci" in sheet        | `setDetail(null)` (chiude e basta)  | `UPDATE team_alerts SET acknowledged_at=now(), acknowledged_by=$me` + Realtime push        | `isDirector` |
| Risolto                      | Tap "Risolto" in sheet          | `setDetail(null)`                   | `UPDATE team_alerts SET resolved_at=now(), resolved_by=$me` + log in `sales.alert_actions` | `isDirector` |
| Assegna / riassegna (futuro) | Azione mancante                 | —                                   | Cambia `targetUserId` (riequilibrio)                                                       | —            |
| Apri lead correlato (futuro) | Azione mancante nel sheet       | —                                   | Deep-link `/sales/lead/<leadId>`                                                           | —            |
| Snooze (futuro)              | Azione mancante                 | —                                   | Rimanda alert di N ore/giorni                                                              | —            |

---

## 5. Componenti chiave

- [IOSNavBar](../../../src/components/ios/IOSNavBar.tsx) — titolo + action refresh
- [IOSCard](../../../src/components/ios/IOSPrimitives.tsx) — container counter + filtri
- [SegmentedControl](../../../src/components/ios/IOSPrimitives.tsx) — tab open/ack/resolved/all con badge counter
- [BottomSheet](../../../src/components/ios/IOSPrimitives.tsx) — dettaglio alert
- [IOSButton](../../../src/components/ios/IOSPrimitives.tsx) — CTA Riconosci/Risolto
- [Chip](../../../src/components/common/Chip.tsx) — severity
- [EmptyState](../../../src/components/sales/EmptyState.tsx) — stato "tutto ok"
- Card alert custom inline (bordo sinistro colorato per severity + icona `AlertTriangle` tinted)
- Icone lucide: `AlertTriangle`, `ChevronDown`/`ChevronUp`, `RefreshCw`, `CheckCircle2`, `ShieldCheck`

---

## 6. Integrazioni future

- **Supabase tables**: `sales.team_alerts` (id, pod_team_id, target_user_id, lead_id, alert_type, severity, title, message, threshold_config, triggered_at, acknowledged_at, acknowledged_by, resolved_at, resolved_by); `sales.alert_rules` (regole attive per POD); `sales.alert_actions` (audit log)
- **RLS policy**: Director legge/scrive solo alert del proprio `pod_team_id`; Admin tutto; sales legge solo propri alert (pagina Home)
- **Realtime channel**: `pod:{pod_id}:alerts` → push su INSERT/UPDATE, aggiorna counter + lista senza refresh manuale
- **Edge function**: `evaluate-alert-rules` (scheduled ogni 15 min) — scansiona `sales.leads`/`appointments`/`quotes` applicando regole da `alert_rules`, INSERT nuovo alert se soglia superata e non già aperto
- **n8n workflow**:
  - invia notifica Slack/email al Director quando appare nuovo critical
  - notifica push mobile al sales target quando severity = high
  - escalation: se un critical resta aperto >24h, notifica Admin
- **AI call**: sintesi naturalized del message + suggested action ("Questo lead è fermo perché cliente ha chiesto 2 settimane. Suggerisco follow-up il 24/04 con template X")
- **Analytics event**: `alert_viewed`, `alert_acknowledged`, `alert_resolved`, `alert_drilldown`, `alert_filter_applied`, `alert_snoozed`

---

## 7. UX/UI review

### Funziona bene

- Hierarchy visiva: counter severity → tab stato → filtri collassabili → lista → sheet
- Bordo sinistro colorato per severity + icona tinted: pattern-match rapido
- Empty state positivo con ShieldCheck e tono rassicurante
- Counter tab con badge numerico nel segmented control
- Filtri avanzati collassati di default: non rubano spazio
- Bottom sheet con azione primaria "Risolto" a destra (gerarchia filled > tinted)

### Attriti UX attuali

- **Azioni Ack/Risolto non persistono**: la sheet chiude ma il dato non cambia (mock)
- Bottone refresh è decorativo: non fa nulla
- Nessuna indicazione in card del tempo trascorso da "triggered" (solo nel line 3 in formato relativo) — mancano timestamp ack/resolved quando rilevanti
- Nessun deep-link a lead/azienda correlati dal sheet — il Director deve navigare a mano
- Filtri usano `<select>` native: sul mobile iOS ok, su desktop l'UX è meno moderna (no multi-select, no search)
- "Tutti" in tab: sovrapposto a filtri — confonde quando anche un filter è attivo
- Hardcoded `podTeamName === 'Milano Alpha'` nei filter sales: rompe con POD multipli
- `formatRelative` con NOW congelato al 2026-04-18: testi tipo "ora" o "5 min fa" saranno sbagliati appena cambia la data
- Nessuna azione bulk (es. "ack tutti i medium")
- Nessun sort (oggi ordine = array originale)
- Severity counter non cliccabili: sembrerebbero filtri ma non lo sono

### Coerenza UI

- Tipografia: ok (numeri in font-mono tabular-nums, uppercase tracking per eyebrow)
- Spacing: ok (`space-y-4`, `gap-2`)
- Colore: ok — mappa severity→colore consistente (red/orange/yellow/muted)
- Mobile-first: ok — card full-width con left border, touch target ≥44px

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo | Proposta                                                                                                | Effort |
| --- | -------- | ---- | ------------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0       | UX   | Implementare realmente Riconosci/Risolto (persistenza + ottimistic UI + audit log) — oggi è placeholder | M      |
| 2   | P0       | UX   | Collegare il bottone refresh alla refetch (realtime subscribe + manual trigger)                         | S      |
| 3   | P0       | UX   | Deep-link "Apri lead" nel bottom sheet (button secondario) — oggi il Director resta fermo               | S      |
| 4   | P1       | UX   | Severity counter cliccabili → filter rapido (tap "High" applica `severityFilter='high'`)                | S      |
| 5   | P1       | UX   | Azioni bulk: seleziona multipla + ack/risolvi/assegna in massa                                          | M      |
| 6   | P1       | UX   | Snooze alert (1h / 4h / domani / lunedì) con ripristino automatico                                      | M      |
| 7   | P2       | UX   | Sort delle card: Priorità (severity+tempo) / Più recenti / Più vecchi / Per sales                       | S      |
| 8   | P2       | UI   | Sostituire `<select>` con componente brand (cercabile, multi-value)                                     | M      |
| 9   | P2       | UX   | Grafico "alert open nel tempo" (trend settimanale) per Director — capire se la pipeline peggiora        | M      |
| 10  | P3       | UX   | Suggerimento AI nel sheet: "Cosa fare?" con 3 opzioni attuabili                                         | L      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- Regole di generazione alert: sono hardcoded lato edge function o configurabili dal Director tramite UI?
- Soglie per severity: stesso tipo alert può essere medium→critical in base a contesto (budget lead, urgenza cliente)? Quale input?
- Sales può riconoscere un proprio alert o solo il Director? Se sì, le due pagine (Home sales vs questa) vanno allineate
- Escalation automatica dopo N ore: chi decide le regole?
- Cosa succede quando un alert è "resolved" ma la causa torna? Si genera nuovo alert o si riapre il vecchio (deduplicazione)?
- Auditing: serve log "chi ha riconosciuto/risolto quando" visibile in UI (per 1:1)?
- Storicità: dopo quanto tempo un alert resolved viene archiviato e sparisce dalla tab "all"?
- Multi-POD: quando ci saranno più POD, Director di POD-A può vedere alert di POD-B? (RLS dice no, ma la UI potrebbe voler mostrare aggregato al Management)
