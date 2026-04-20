# Provvigioni

**Route**: `/sales/provvigioni`
**File**: [src/pages/sales/SalesProvvigioni.tsx](../../../src/pages/sales/SalesProvvigioni.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: Tutti i sales (vista "Le mie") + POD Director (tab aggiuntiva "Team POD")

---

## 1. Overview

Pagina di motivazione e trasparenza economica: mostra al singolo sales quanto ha maturato nel mese corrente (breakdown per ruolo Setter/Closer/Referral/Gestione), la progressione verso i tier trimestrali (L1/L2/L3) con gap calcolato sul prossimo e la proiezione equity annuale. Al Director dà in più una vista "Team POD" con revenue/provvigione/equity per sales, utile per 1:1 e budget forecast. Recentemente spostata fuori dalla sezione Team: ora visibile a tutti i sales (vista "mine") ma con tab "team" gated per `pod_director`.

---

## 2. Viste & Stati

| ID vista               | Trigger                                | Descrizione                                                                                   | Componenti rilevanti                                                       |
| ---------------------- | -------------------------------------- | --------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| mine (default)         | Ogni ruolo, `tab === 'mine'`           | Hero maturato mese + card tier trimestrale + card equity annua + lista contratti contribuenti | `HeroAI`, `IOSCard`, `GroupedList`, `TierBar`, `NextTierHint`, `PacerHint` |
| team                   | `isDirector && tab === 'team'`         | Lista card per sales del POD con revenue/provvigione/equity/progress                          | `IOSCard`, `Avatar`, `Chip`, `TeamMetric`                                  |
| segmented-hidden       | `!isDirector`                          | Il `SegmentedControl` non è renderizzato, solo vista "mine"                                   | —                                                                          |
| no-commission          | `getCurrentUserCommission()` undefined | Viste tier/equity non renderizzano (render condizionale `comm && equity`)                     | — — fallback silente                                                       |
| no-next-tier           | Utente già a L3                        | `NextTierHint` non renderizza (return null)                                                   | —                                                                          |
| empty contracts        | `contributingQuotes.length === 0`      | Sezione "Contratti contribuenti MTD" con intestazione senza items                             | `GroupedList` (manca empty state dedicato)                                 |
| loading/error (futuri) | fetch Supabase                         | Non gestiti                                                                                   | skeleton + retry                                                           |

---

## 3. Dati & Sorgenti

| Entità UI                                 | Mock (src/data/mock/sales.ts)                                  | Schema reale                                                          | Note                                                                                                        |
| ----------------------------------------- | -------------------------------------------------------------- | --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Utente corrente                           | `getSalesUserById(CURRENT_SALES_USER_ID)`                      | `core.users`                                                          | In prod prendere da sessione Supabase auth                                                                  |
| Commission tier                           | `salesQuarterlyCommissions` (`getCurrentUserCommission`)       | `finance.quarterly_commissions`                                       | Struttura `{revenueTarget, currentRevenue, tierAchieved, tiers:{l1,l2,l3}}` mappa 1:1                       |
| Equity grant                              | `salesEquityGrants` (`getCurrentUserEquity`)                   | `finance.equity_grants`                                               | `grantYear`, `tierLevel`, `sharesValueEur`, `revenueAchieved`                                               |
| Breakdown Setter/Closer/Referral/Gestione | Array hardcoded `earnedBreakdown` in pagina                    | `finance.commission_line_items GROUP BY role_kind`                    | **Attualmente fake**: `[Setter 1200, Closer 3800, Referral 100, Gestione 1000]` non collegato al dato reale |
| Totale maturato                           | `Σ earnedBreakdown.value`                                      | Somma MV `finance.v_monthly_earned_by_user`                           | Non coincide con `commission.earnedAmount` (3212.5 per Gab vs 6100 hero)                                    |
| Contratti contribuenti                    | `salesQuotes.filter(status∈['accepted','sent']).slice(0,6)`    | `contracts.contracts WHERE owner_user_id = $me AND signed_at in mese` | Qui usa `quotes` non `contracts`; "role" dedotto da `status` (fake)                                         |
| Commissione per contratto                 | `Math.round(q.totalAmount * 0.03)` inline                      | Calcolo server-side su `finance.commissions`                          | 3% piatto hardcoded, ignora tier corrente                                                                   |
| Team POD                                  | `salesUsers.filter(podTeamName === currentUser.podTeamName)`   | `sales.pod_team_members`                                              | OK come perimetro                                                                                           |
| Team commission per sales                 | `salesQuarterlyCommissions.find(userId)`                       | `finance.quarterly_commissions`                                       | OK                                                                                                          |
| Pacer equity                              | Proiezione lineare `(revenueAchieved / 4) * 12` in `PacerHint` | Forecast lato server con stagionalità                                 | Assume aprile = mese 4/12, calcolo troppo ingenuo                                                           |

---

## 4. Pulsanti & Azioni

| Azione                      | Trigger UI             | Effetto attuale (mock)        | Effetto target (produzione)                                                      | Gated da                                     |
| --------------------------- | ---------------------- | ----------------------------- | -------------------------------------------------------------------------------- | -------------------------------------------- |
| Switch tab mine/team        | Tap `SegmentedControl` | `setTab(v)` locale            | Stesso + query param `?tab=` per deep-link                                       | `isDirector` (solo director vede il control) |
| Item contratto contribuente | Tap riga `GroupedList` | Nessuna azione (no `onClick`) | Apre dettaglio contratto/quote `/sales/preventivi/:id`                           | —                                            |
| Item team sales             | Tap card               | Nessuna azione                | Apre sheet dettaglio provvigione o deep-link `/sales/performance-pod?owner=<id>` | `isDirector`                                 |
| Chip tier                   | Visual                 | —                             | Tooltip con condizioni (rate%, soglia, periodo)                                  | —                                            |
| Export payout (futuro)      | Bottone assente        | —                             | Scarica CSV/PDF cedolino provvigione mensile                                     | —                                            |

---

## 5. Componenti chiave

- [IOSNavBar](../../../src/components/ios/IOSNavBar.tsx) — header
- [HeroAI](../../../src/components/ios/IOSPrimitives.tsx) — hero dark con maturato mese
- [IOSCard](../../../src/components/ios/IOSPrimitives.tsx) — card tier/equity/team
- [SegmentedControl](../../../src/components/ios/IOSPrimitives.tsx) — switch mine/team
- [Chip](../../../src/components/common/Chip.tsx) — tier achieved / ruolo
- [Avatar](../../../src/components/common/Avatar.tsx) — vista team
- [GroupedList](../../../src/components/ios/GroupedList.tsx) — contratti contribuenti
- Componenti locali: `TierBar`, `NextTierHint`, `PacerHint`, `TeamMetric` — utility in-file

---

## 6. Integrazioni future

- **Supabase tables**: `finance.quarterly_commissions`, `finance.equity_grants`, `finance.commission_line_items`, `contracts.contracts`, `sales.quotes`
- **Supabase views (da creare)**:
  - `finance.v_monthly_earned_by_user(user_id, month)` con scomposizione per `role_kind` (setter/closer/referral/gestione)
  - `finance.v_tier_projection(user_id, quarter)` con gap al next tier e ETA stimato
  - `finance.v_equity_pacer(user_id, year)` con proiezione forward stagionale
- **RLS policy**: `finance.*` policy: user vede solo proprie righe; `pod_director` vede tutti i sales del proprio `pod_team_id`; Admin vede tutto
- **Realtime channel**: `user:{id}:earnings` → update hero quando un contratto del mese viene firmato
- **Edge function**: `compute-monthly-earnings(user_id, month)` con cache 15 min
- **Webhook payout**: `payout.provvigioni.issued` da Stripe Connect/banca → aggiorna `finance.payouts` + notifica in-app "Provvigione accreditata"
- **n8n workflow**: mensile il 1° di ogni mese — snapshot earnings, genera PDF cedolino, invia email al sales + commercialista
- **AI call**: "Com'è andato il mio mese?" → riepilogo testuale contestualizzato (hit/miss tier, leva per arrivare al prossimo)
- **Analytics event**: `provvigioni_viewed`, `provvigioni_tab_switched`, `provvigioni_tier_achieved` (server-side), `provvigioni_payout_downloaded`

---

## 7. UX/UI review

### Funziona bene

- Hero scuro con totale grande: impatto motivazionale forte
- Tier bar con tick marks L1/L2/L3 comunica istantaneamente dove sei e quanto manca
- `NextTierHint` trasforma numeri astratti in azione ("mancano 51.500€ al tier L2")
- `PacerHint` proietta equity annuo mantenendo il ritmo attuale
- Vista team compatta: revenue + provvigione + equity in tre celle, progress bar in fondo
- Chip colorato tier achieved dà status signal a colpo d'occhio

### Attriti UX attuali

- **Disallineamento dati grave**: hero mostra 6.100€ (breakdown hardcoded) ma `commission.earnedAmount` per Gabriele è 3.212,50€ — due numeri in pagina non quadrano
- Commissione per contratto hardcoded al 3%: ignora tier effettivo (2.5/3.0/3.5)
- "Role" Setter/Closer derivato da `quote.status`: euristica errata, non è quello il campo che determina il ruolo
- Data di riferimento hero: "Maturato ad aprile" ma non c'è selettore mese
- Nessun link da contratto contribuente a dettaglio contratto
- `TierBar` labels "100K / 180K / 280K" duplicati (sopra progress + sotto labels tier card): ridondanza
- Tab "Team POD" non ha filtri (per ruolo, per tier raggiunto, per % target): diventa rumoroso con team grande
- Nessuna storia provvigioni passate (mesi precedenti, trimestri chiusi)
- `PacerHint` assume aprile = 4/12 sempre: rompe con cambio mese
- Empty state mancante se non ci sono contratti contribuenti
- `formatRelative` con `NOW` hardcoded al `2026-04-18T10:30:00+02:00`: tempo congelato

### Coerenza UI

- Tipografia: ok (font-mono per cifre, font-serif per numeri chiave)
- Spacing: ok (`space-y-5`, grid gap coerente)
- Colore: ok (orange = commission, green = equity, red non usato)
- Mobile-first: ok — hero + card stack, segmented sticky

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo | Proposta                                                                                                                                    | Effort |
| --- | -------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0       | Data | Sostituire `earnedBreakdown` hardcoded con query reale su `finance.commission_line_items`: gli importi in pagina devono coincidere tra loro | M      |
| 2   | P0       | Data | Calcolare commissione per contratto usando il tier corrente + ruolo dell'utente sul quote, non `totalAmount * 0.03`                         | M      |
| 3   | P0       | UX   | Aggiungere selettore mese/trimestre con deep-link + sincronizzare hero, tier card, contratti contribuenti                                   | M      |
| 4   | P1       | UX   | Storico provvigioni (ultimi 6 mesi + trimestri chiusi) con grafico barre + export PDF cedolino                                              | L      |
| 5   | P1       | UX   | Click su contratto contribuente → dettaglio `/sales/preventivi/:id` (oggi non cliccabile)                                                   | S      |
| 6   | P1       | UX   | Tab team: aggiungere filtri (tier, % target, ruolo) e sort (revenue desc, earned desc)                                                      | M      |
| 7   | P2       | UX   | Empty state per "nessun contratto contribuente questo mese" (ora lista vuota muta)                                                          | S      |
| 8   | P2       | UX   | Animazione confetti / haptic quando si scavalca un tier (mobile)                                                                            | S      |
| 9   | P2       | UI   | Pulsante "Esporta cedolino" PDF + CSV (anche team per Director)                                                                             | M      |
| 10  | P3       | UX   | Forecast AI "cosa mi serve per chiudere il prossimo tier" (lead in pipeline + probability-weighted revenue)                                 | L      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- Fonte di verità per scomposizione earned (Setter/Closer/Referral/Gestione): esiste già tabella `commission_line_items` con `role_kind`? Altrimenti schema da progettare
- Politica ruolo per contratto: il setter è chi ha creato il lead o chi ha fatto primo contatto? Il closer chi ha firmato o chi ha condotto il meeting?
- Tempistica payout: mensile o trimestrale? Se mensile anticipa il tier, chi riassorbe la differenza se il trimestre chiude sotto soglia?
- Equity: va mostrato anche il valuation implicito (sharesValueEur è già il controvalore €?) e il lockup/vesting?
- Visibilità tab "Team": il Director vede tutti o solo i propri diretti? Admin vede tutti i POD?
- Storicizzazione: quando cambia lo schema tier durante l'anno, come archiviamo quello vecchio?
- Provvigione su upsell / rinnovo contratti: stessa % del nuovo business o ridotta?
