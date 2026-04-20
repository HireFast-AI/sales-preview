# Geo-Vendite

**Route**: `/sales/geo`
**File**: [src/pages/sales/SalesGeoVendite.tsx](../../../src/pages/sales/SalesGeoVendite.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: POD Director (gating `isDirector(CURRENT_SALES_USER_ID)` → redirect `/sales` altrimenti)

---

## 1. Overview

Navigatore geografico della performance commerciale: esplora l'Italia in tre livelli (Regioni → Città → Aziende) con metrica selezionabile (Revenue / Spend / Lead / Clienti / ROAS). Pensato per Director e future figure di Management per capire dove concentrare spend outreach, quali regioni hanno ROAS migliore, dove replicare un case study esistente. Include link diretto ai case study regionali per citare prove sociali in conversazione commerciale.

---

## 2. Viste & Stati

| ID vista               | Trigger                                 | Descrizione                                                  | Componenti rilevanti                 |
| ---------------------- | --------------------------------------- | ------------------------------------------------------------ | ------------------------------------ |
| regions (default)      | `level === 'regions'`                   | Lista 11 regioni ranked by metrica selezionata               | `GroupedList`, `SegmentedControl`    |
| cities                 | Tap regione                             | Header regione + lista città filtrate per `regionId`         | `IOSCard`, `GroupedList`             |
| companies              | Tap città                               | Header città + lista aziende di quella città con chip status | `IOSCard`, `GroupedList`, `Chip`     |
| empty-companies        | `getCompaniesByCity(city).length === 0` | "Nessuna azienda a X" con icona MapPin                       | `EmptyState`                         |
| breadcrumb             | sempre (sticky)                         | Italia › Regione › Città con back-link cliccabili            | `<button>` custom con `ChevronRight` |
| forbidden              | `!isDirector`                           | `<Navigate to="/sales" replace />`                           | —                                    |
| loading/error (futuri) | fetch Supabase                          | Non gestiti                                                  | skeleton + retry                     |

Nota: nessuna mappa grafica presente oggi — il nome "Geo" è suggestivo ma la UI è una navigazione a lista gerarchica.

---

## 3. Dati & Sorgenti

| Entità UI         | Mock (src/data/mock/sales.ts)                                         | Schema reale                                                        | Note                                                                                                                                          |
| ----------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| Regioni           | `salesGeoRegions` (11 item)                                           | Vista MV `sales.v_geo_by_region` aggregando `core.companies.region` | Campi: `name`, `isoCode`, `citiesCount`, `clientsCount`, `leadsCount`, `spendTotal`, `revenueTotal`, `roas`, `caseStudiesCount`, `lat`, `lng` |
| Città             | `salesGeoCities` (20 item), `getCitiesByRegion(regionId)`             | Vista MV `sales.v_geo_by_city`                                      | Campi extra: `cac`, `cpl` (non usati oggi in UI!)                                                                                             |
| Aziende per città | `getCompaniesByCity(cityName)` → `salesGeoCompanies`                  | `core.companies` + aggregate `sales.leads`/`contracts.contracts`    | Oggi match su stringa city (case sensitive)                                                                                                   |
| Case studies      | `salesCaseStudies.find(region === co.region && sector === co.sector)` | `marketing.case_studies`                                            | Link esterno `externalUrl` — apre hirefast.it                                                                                                 |
| Metric            | `metricOptions: [revenue, spend, leads, clients, roas]`               | colonne precalcolate nelle MV                                       | ROAS hardcoded nel mock ma facile da calcolare server side (revenue/spend)                                                                    |
| Status azienda    | `co.status ∈ {'lead','prospect','client'}`                            | `core.companies.status`                                             | Coerente con `CompanyStatus` del dominio                                                                                                      |
| Breadcrumb        | derivato da `region` + `city` via find                                | parametri di route                                                  | Meglio URL-driven (`/sales/geo/:region/:city`)                                                                                                |

---

## 4. Pulsanti & Azioni

| Azione                | Trigger UI                                                                              | Effetto attuale (mock)                                 | Effetto target (produzione)                                  | Gated da     |
| --------------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------ | ------------ |
| Cambia metrica        | Tap chip in `SegmentedControl`                                                          | `setMetric(v)` ricomputa `sortedRegions`               | + query param `?metric=` + analytics                         | `isDirector` |
| Drill regione         | Tap riga regione                                                                        | `setRegionId(r.id); setLevel('cities')`                | + push su router `/sales/geo/:region`                        | `isDirector` |
| Drill città           | Tap riga città                                                                          | `setCityName(c.city); setLevel('companies')`           | + push `/sales/geo/:region/:city`                            | `isDirector` |
| Back "Italia"         | Tap "Italia" breadcrumb                                                                 | `goToRegions()` reset a livello root                   | `navigate(-N)` o `/sales/geo`                                | —            |
| Back regione          | Tap nome regione in breadcrumb (solo se `level === 'companies'` o disabled in `cities`) | `goToCities()`                                         | idem                                                         | —            |
| Apri case study       | Tap icona `ExternalLink` in riga azienda                                                | `window.open(externalUrl)` + `stopPropagation`         | Stesso + tracking `case_study_opened`                        | —            |
| Apri azienda (futuro) | Tap riga azienda                                                                        | Nessun `onClick` definito — chevron presente ma inerte | Apre dettaglio `/sales/lead?companyId=<id>` o scheda azienda | —            |
| Export CSV (futuro)   | Bottone assente                                                                         | —                                                      | Scarica lista corrente con metrica                           | —            |
| Toggle mappa (futuro) | Bottone assente                                                                         | —                                                      | Passa da lista a mappa Italia choropleth                     | —            |

---

## 5. Componenti chiave

- [IOSNavBar](../../../src/components/ios/IOSNavBar.tsx) — titolo + sottotitolo dinamico su livello
- [IOSCard](../../../src/components/ios/IOSPrimitives.tsx) — header regione/città
- [SegmentedControl](../../../src/components/ios/IOSPrimitives.tsx) — switch metrica
- [GroupedList](../../../src/components/ios/GroupedList.tsx) — liste con ranking/chip/trailing numerico
- [Chip](../../../src/components/common/Chip.tsx) — status azienda (cliente/prospect/lead)
- [EmptyState](../../../src/components/sales/EmptyState.tsx) — città senza aziende
- Icone lucide: `MapPin`, `ChevronRight`, `ExternalLink`
- Breadcrumb sticky custom (non component: inline `<button>` con backdrop-blur)

---

## 6. Integrazioni future

- **Supabase tables**: `core.companies`, `sales.leads`, `contracts.contracts`, `campaigns.campaigns`, `campaigns.spend_daily`, `marketing.case_studies`
- **Supabase views (da creare)**:
  - `sales.v_geo_by_region(region, metric_window)` — aggregate revenue/spend/lead/client/ROAS per regione
  - `sales.v_geo_by_city(city, province, region, metric_window)` — stesso per città + CAC/CPL
  - `sales.v_geo_companies_by_city(city)` — aziende con ultimo touch
- **RLS policy**: Director vede tutto geo aggregato del proprio POD; Admin tutto; Sales può vedere solo geo del proprio portafoglio (modalità personal future)
- **Realtime channel**: meno critico (dati aggregati non richiedono realtime); refresh scheduled ogni ora via edge function + invalidate cache React Query
- **Edge function**: `refresh-geo-mv(window)` chiamata da cron ogni ora
- **Mapbox / Leaflet**: choropleth Italia con `isoCode` ISO-3166-2 (es. IT-25 Lombardia). Libreria consigliata: MapLibre GL JS (OSS, no API key) o Mapbox GL con token. GeoJSON regioni: `https://github.com/openpolis/geojson-italy`
- **Analytics**: dashboard Looker/Grafana con dettaglio spend/ROAS per campagna geo-targeted
- **n8n workflow**: alert al Director quando ROAS regione scende sotto 1.5x per >7 giorni
- **AI call**: "Dove conviene investire il prossimo budget?" — suggestion con logica LTV/CAC per regione
- **Webhook**: `campaigns.spend_updated` → invalida MV geo
- **Analytics event**: `geo_viewed`, `geo_metric_changed`, `geo_region_drilldown`, `geo_city_drilldown`, `case_study_opened_from_geo`

---

## 7. UX/UI review

### Funziona bene

- Navigazione gerarchica a 3 livelli semplice da capire
- Breadcrumb sticky con back-link cliccabili su ogni livello
- Metrica switchabile con re-sort immediato delle regioni
- Ranking 1-11 in prima posizione riga regioni: orientamento visivo rapido
- Chip status (client/prospect/lead) colorato per identificazione rapida
- Link case study inline sull'azienda: prova sociale in contesto
- Empty state dedicato per città senza aziende
- Sottotitolo navbar aggiornato in base al livello ("Italia › Lombardia › Milano")

### Attriti UX attuali

- **Non è una mappa**: il nome "Geo-Vendite" promette cartografia ma è una lista. Aspettativa tradita
- Metrica applicata alle regioni ma **non re-sorta** città/aziende (hanno ordine array)
- CAC/CPL presenti nel dato città ma non mostrati in UI
- Riga azienda ha `chevron: true` ma nessun `onClick`: tap senza effetto, anti-pattern
- Nessun confronto temporale (metrica ultimi 30gg? trimestre? anno?) — perimetro implicito
- Breadcrumb `level === 'cities'` disabilita il bottone "regione" ma resta col colore teal: visualmente inconsistente
- Stato URL non riflette livello/regione/città: il refresh torna a root
- Ricerca testuale assente: se devi trovare "Verona" nella lista città lombarde, non la troverai (è Veneto)
- Liguria e Sardegna con 0 città/clienti ma presenti in lista: rumore (serve filtro "Attive" o ordinamento con zero in coda)
- Match case study basato su `region + sector` stringa: fragile e spesso restituisce il primo (solo 1 `match`)
- Mobile: breadcrumb in bianco semi-trasparente su scroll può faticare su hero chiaro
- Nessuna heatmap/intensità visiva tra regioni (tutte uguali visivamente tranne il numero)

### Coerenza UI

- Tipografia: ok (serif numeri, mono per valori, uppercase tracking eyebrow)
- Spacing: ok (`space-y-4`, gap coerente)
- Colore: ok (teal per link, ink per attivo) ma manca scala chromatic per metrica
- Mobile-first: ok — ma il vero valore geo esploderebbe su desktop con mappa

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo | Proposta                                                                                          | Effort |
| --- | -------- | ---- | ------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0       | UX   | URL-driven nav (`/sales/geo/:region/:city`) + query `?metric=` — deep-link e refresh-safe         | M      |
| 2   | P0       | UX   | Sort metrica anche su città e aziende (non solo regioni)                                          | S      |
| 3   | P0       | UI   | Rendere cliccabile la riga azienda (deep-link scheda azienda) o togliere il chevron               | S      |
| 4   | P1       | UI   | Aggiungere vista mappa (choropleth Italia) con MapLibre/Mapbox + toggle lista/mappa               | XL     |
| 5   | P1       | UX   | Selettore periodo (30gg / trimestre / anno / custom) per tutte le metriche                        | M      |
| 6   | P1       | UX   | Data density: mostrare CAC/CPL su città (oggi nel mock ma non in UI), delta vs periodo precedente | S      |
| 7   | P2       | UX   | Ricerca testuale città/azienda globale                                                            | M      |
| 8   | P2       | UX   | Heatmap coloring righe regione in base a metrica (gradient bg) — scan visivo più veloce           | S      |
| 9   | P2       | UX   | Export CSV della vista corrente (regioni/città/aziende)                                           | S      |
| 10  | P2       | UX   | Anomaly detection: highlight regioni con ROAS < soglia o calo >20% vs periodo precedente          | M      |
| 11  | P3       | UX   | Multi-metric view: piccolo sparkline per metrica sulla riga (revenue + spend stacked)             | L      |
| 12  | P3       | UX   | AI suggestion "Dove investire il prossimo €" con ranking cost-weighted                            | L      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- Perimetro temporale delle metriche: oggi è "tutto lo storico"? Da confermare coi stakeholder quale finestra di default (30gg / Q / YTD)
- Fonte per `spendTotal`: include spend piattaforme Meta/Google o solo budget interno? Come normalizziamo quando una campagna copre più regioni?
- Match case study: meglio tabella esplicita `case_study_companies` con `company_id` piuttosto che euristica `region + sector`
- Mappa choropleth: scegliere tra Mapbox (serve token, ricco) vs MapLibre (OSS, zero costo) vs Leaflet (semplice, stile datato)
- Privacy aziende: i nomi azienda sono visibili a tutti i Director o solo quelli con contratto attivo? Caso "prospect" / "lead" è sensibile
- Quando ci saranno più POD: la metrica "revenue regione" è aggregato totale o del solo POD dell'utente? Impatta su RLS e interpretazione
- Città con `clientsCount = 0` ma `leadsCount > 0`: sono utili (pipeline early)? Mostrare con chip "solo lead"?
- ROAS "0" per Liguria/Sardegna per via di revenue = 0: va mostrato come "—" invece di 0.00x per evitare confusione
