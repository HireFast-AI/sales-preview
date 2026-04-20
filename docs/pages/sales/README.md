# Sales Portal — Page Documentation

Documentazione architetturale pagina-per-pagina del portale HireFast Sales.
Ciascun doc descrive le viste, i pulsanti, i dati, le integrazioni future e i miglioramenti UX/UI proposti.

## Deploy

- Preview pubblica: https://hirefast-ai.github.io/sales-preview/
- Repo preview: https://github.com/HireFast-AI/sales-preview
- Repo applicativo: https://github.com/HireFast-AI/webapp-hirefast-main (branch `add-sales-react-portal`)

## Gating ruoli

| Ruolo        | Vede                                                                                                                  |
| ------------ | --------------------------------------------------------------------------------------------------------------------- |
| Sales        | Home, Pipeline, Lead, Ricerca Mercato, Appuntamenti, Conversazioni, Outreach, Provvigioni, AI Copilot, Template Email |
| POD Director | Tutto quanto Sales + Performance POD, Alert Team, Geo-Vendite                                                         |
| Admin        | Tutto quanto POD Director + Admin                                                                                     |

## Pagine documentate

| #   | Pagina                 | Route                        | Ruolo        | Doc                                                    |
| --- | ---------------------- | ---------------------------- | ------------ | ------------------------------------------------------ |
| 1   | Home                   | `/sales`                     | Tutti        | [home.md](home.md)                                     |
| 2   | Pipeline               | `/sales/pipeline`            | Tutti        | [pipeline.md](pipeline.md)                             |
| 3   | Lead                   | `/sales/lead`                | Tutti        | [lead.md](lead.md)                                     |
| 4   | Lead Detail            | `/sales/lead/:id`            | Tutti        | [lead-detail.md](lead-detail.md)                       |
| 5   | Ricerca Mercato        | `/sales/ricerca-mercato`     | Tutti        | [ricerca-mercato.md](ricerca-mercato.md)               |
| 6   | Ricerca Mercato Detail | `/sales/ricerca-mercato/:id` | Tutti        | [ricerca-mercato-detail.md](ricerca-mercato-detail.md) |
| 7   | Appuntamenti           | `/sales/appuntamenti`        | Tutti        | [appuntamenti.md](appuntamenti.md)                     |
| 8   | Preventivi             | `/sales/preventivi`          | Tutti        | [preventivi.md](preventivi.md)                         |
| 9   | Conversazioni          | `/sales/conversazioni`       | Tutti        | [conversazioni.md](conversazioni.md)                   |
| 10  | Outreach               | `/sales/outreach`            | Tutti        | [outreach.md](outreach.md)                             |
| 11  | Template Email         | `/sales/template-email`      | Tutti        | [template-email.md](template-email.md)                 |
| 12  | AI Copilot             | `/sales/ai-copilot`          | Tutti        | [ai-copilot.md](ai-copilot.md)                         |
| 13  | Performance POD        | `/sales/performance-pod`     | POD Director | [performance-pod.md](performance-pod.md)               |
| 14  | Provvigioni            | `/sales/provvigioni`         | Tutti        | [provvigioni.md](provvigioni.md)                       |
| 15  | Alert Team             | `/sales/alert-team`          | POD Director | [alert-team.md](alert-team.md)                         |
| 16  | Geo-Vendite            | `/sales/geo`                 | POD Director | [geo-vendite.md](geo-vendite.md)                       |
| 17  | Admin                  | `/sales/admin`               | Admin        | [admin.md](admin.md)                                   |

## Legenda priorità

- **P0** bloccante, da fare subito
- **P1** alto, ciclo corrente
- **P2** medio, prossimo ciclo
- **P3** nice-to-have

## Legenda effort

- **S** < 1h
- **M** 1-4h
- **L** 1-2 giorni
- **XL** > 2 giorni

## Come usare questo archivio

1. Base architetturale per implementation plan
2. Onboarding nuovi dev/designer
3. Source of truth per scope produzione

## Stato aggiornamento

Generato il 2026-04-20 da Claude Opus 4.7.
