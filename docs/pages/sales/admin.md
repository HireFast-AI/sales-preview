# Admin (Impostazioni)

**Route**: `/sales/admin`
**File**: [src/pages/sales/SalesAdmin.tsx](../../../src/pages/sales/SalesAdmin.tsx)
**Layout**: `SalesLayout` (desktop sidebar 272px + topbar 72px, mobile tab bar + `QuickNoteFAB`)
**Ruolo richiesto**: Admin (voce di menu visibile solo al ruolo `admin`; la rotta in sé non è gated lato router)

---

## 1. Overview

Pannello "Impostazioni" del portale Sales: mostra il profilo utente corrente, il suo POD Team, lo stato delle integrazioni (Notion, Brevo, Instantly, WABA Meta), i toggle per le notifiche personali e i link di supporto. È l'entry point per la gestione di identità, preferenze e connettori del venditore. In gating attuale è di fatto l'unica pagina riservata al ruolo `admin`, ma il contenuto è in gran parte un "account / preferences" per il singolo utente — la parte amministrativa vera (tenant, ruoli, audit) è ancora da costruire.

---

## 2. Viste & Stati

| ID vista           | Trigger                                                           | Descrizione                                                                                     | Componenti rilevanti                                                           |
| ------------------ | ----------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| default            | Utente autenticato trovato in `salesUsers`                        | Card profilo + 4 sezioni GroupedList (POD Team, Integrazioni, Notifiche, Supporto) + CTA logout | `IOSNavBar`, `IOSCard`, `Avatar`, `GroupedList`, `Chip`, `IOSButton`, `Toggle` |
| no-user            | `getSalesUserById(CURRENT_SALES_USER_ID)` restituisce `undefined` | La card profilo non viene renderizzata; il resto della pagina è visibile                        | `IOSNavBar`, `GroupedList`                                                     |
| toggles-on/off     | Click su uno switch Notifiche                                     | Aggiorna stato React locale (nessuna persistenza)                                               | `Toggle`                                                                       |
| integration-ok     | `status: "green"`                                                 | Chip verde "Connesso" accanto a Notion / Brevo / WABA Meta                                      | `Chip variant="green"`                                                         |
| integration-config | `status: "yellow"`                                                | Chip gialla "In config" accanto a Instantly                                                     | `Chip variant="yellow"`                                                        |
| supporto-docs      | Click su "Documentazione sales"                                   | `window.open("https://docs.hirefast.it")` in nuova tab                                          | `GroupedList` item                                                             |
| supporto-mail      | Click su "Richiedi supporto"                                      | `window.location.href = mailto:support@hirefast.it`                                             | `GroupedList` item                                                             |
| feedback           | Click su "Feedback"                                               | No-op (nessun handler collegato)                                                                | `GroupedList` item                                                             |

Stati mancanti (non implementati): `loading`, `error`, `empty` integrazioni, `unauthorized` (se non-admin arriva alla rotta), `saving toggle`.

---

## 3. Dati & Sorgenti

| Entità UI         | Mock (src/data/mock/sales.ts)                                                                   | Schema reale (Supabase v6.1)                                                               | Note                                                                                            |
| ----------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| Utente corrente   | `CURRENT_SALES_USER_ID = "u-gab-001"`, `salesUsers[]`, `getSalesUserById()`                     | `users` (auth) + `user_profiles` (fullName, avatarInitials, avatarColor, startedAt, phone) | Oggi hardcoded su Gabriele; in produzione → `supabase.auth.getUser()` + join su `user_profiles` |
| POD Team (membri) | `salesUsers.filter(u => u.podTeamName === user.podTeamName)`                                    | `pod_teams` + `user_pod_memberships`                                                       | Manca `pod_team_id` FK: oggi si usa match per stringa `podTeamName`                             |
| Ruolo             | `user.role` ∈ `pod_director \| sales \| closer \| admin`                                        | `user_roles` (role_id) + `permissions`                                                     | Nessuna enforce lato UI: solo label decorativa                                                  |
| Integrazioni      | Array `integrations` hardcoded in-file                                                          | `tenant_integrations` (tenantId, provider, status, credentials_vault_ref, last_sync_at)    | Nessuna di queste è letta da DB; statuses sono placeholder                                      |
| Config WABA       | `salesWabaConfig.sharedPhoneNumber` ("+39 02 9476 8810", provider "360dialog (WABA Cloud API)") | `waba_config` (phone_number_id, provider, webhook_secret, routing_rules)                   | Numero mostrato è mock condiviso HireFast                                                       |
| Toggle notifiche  | `useState` locale: `email_digest`, `push_mobile`, `alert_critical`                              | `user_notification_preferences` (channel, type, enabled, schedule)                         | Nessuna persistenza; reset a `true` al refresh                                                  |
| Versione app      | Stringa `"HireFast Sales · v6.1"` nel footer                                                    | Build metadata (`import.meta.env.VITE_APP_VERSION`)                                        | Hardcoded                                                                                       |

---

## 4. Pulsanti & Azioni

| Azione                            | Trigger UI                                            | Effetto attuale (mock)                              | Effetto target (produzione)                                            | Gated da                        |
| --------------------------------- | ----------------------------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------- |
| Apri profilo membro POD           | Click su riga POD Team                                | Nessun handler (solo chevron decorativo)            | Drawer/route `/sales/admin/users/:id` con permessi RBAC                | `admin` o `pod_director` (read) |
| Badge "Tu"                        | Render automatico se `u.id === CURRENT_SALES_USER_ID` | Chip arancione statica                              | Nessuna logica aggiuntiva                                              | —                               |
| Apri dettaglio integrazione       | Click riga Integrazioni                               | Nessun handler                                      | Modal OAuth/config + log `integration_events`                          | `admin`                         |
| Toggle "Email daily digest"       | `role="switch"`                                       | `setToggle("email_digest", v)` in state locale      | `PATCH /user/notification_preferences` + conferma toast                | Utente proprietario             |
| Toggle "Push notifiche mobile"    | `role="switch"`                                       | `setToggle("push_mobile", v)`                       | Registrazione/revoca device token (FCM/APNs)                           | Utente proprietario             |
| Toggle "Alert critical real-time" | `role="switch"`                                       | `setToggle("alert_critical", v)`                    | Subscribe/unsubscribe channel Supabase Realtime `alerts:critical`      | `pod_director` o `admin`        |
| Documentazione sales              | Click riga Supporto                                   | `window.open("https://docs.hirefast.it", "_blank")` | Link stabile versione-corrente + SSO se doc è privato                  | Tutti                           |
| Richiedi supporto                 | Click riga Supporto                                   | `mailto:support@hirefast.it`                        | Apertura ticket in-app (tabella `support_tickets`) + pre-fill contesto | Tutti                           |
| Feedback                          | Click riga Supporto                                   | **No-op** (bottone senza handler)                   | Modal form → `POST /feedback` (category, message, screenshot)          | Tutti                           |
| Esci dall'account                 | Click `IOSButton` destructive                         | **No-op** (nessun `onClick`)                        | `supabase.auth.signOut()` + redirect `/login` + clear cache            | Tutti                           |

---

## 5. Componenti chiave

- [IOSNavBar](../../../src/components/ios/IOSNavBar.tsx) — barra titolo/sottotitolo ("Impostazioni" · "Profilo e configurazioni")
- [IOSCard](../../../src/components/ios/IOSPrimitives.tsx) — card profilo utente (padding `lg`)
- [IOSButton](../../../src/components/ios/IOSPrimitives.tsx) — CTA logout (`variant="tinted"` `tone="destructive"`)
- [GroupedList](../../../src/components/ios/GroupedList.tsx) — 4 sezioni (POD Team, Integrazioni, Notifiche, Supporto)
- [Avatar](../../../src/components/common/Avatar.tsx) — avatar iniziali + colore, taglie `xl` (profilo) e `md` (lista POD)
- [Chip](../../../src/components/common/Chip.tsx) — varianti `orange` ("Tu"), `green` ("Connesso"), `yellow` ("In config")
- `Toggle` (locale al file) — switch custom accessibile (`role="switch"`, `aria-checked`, `aria-label`)
- Icone [lucide-react](https://lucide.dev): `LifeBuoy`, `FileText`, `Mail`, `Zap`, `MessageCircle`, `LogOut`, `BookOpen`, `MessagesSquare`

---

## 6. Integrazioni future

- **Supabase tables**: `users`, `user_profiles`, `user_roles`, `pod_teams`, `user_pod_memberships`, `tenant_integrations`, `waba_config`, `user_notification_preferences`, `user_devices` (FCM/APNs tokens), `audit_log`
- **RLS policy**:
  - `user_profiles`: SELECT self; UPDATE self; SELECT same-tenant per `admin` e `pod_director`
  - `tenant_integrations`: SELECT/UPDATE solo `role = admin` su tenant corrente
  - `waba_config`: SELECT `admin` + `pod_director`; UPDATE solo `admin`
  - `user_notification_preferences`: SELECT/UPDATE self
  - `audit_log`: INSERT server-side only, SELECT `admin`
- **Tenant isolation**: ogni query filtrata da `tenant_id = auth.jwt() ->> 'tenant_id'`; enforce via RLS policy helper `is_same_tenant(row.tenant_id)`
- **Realtime channel**:
  - `tenant:{tenantId}:integrations` → refresh automatico dello stato connettori
  - `user:{userId}:prefs` → sync preferenze cross-device
- **Edge function**:
  - `auth-signout` → revoca sessione + invalida refresh token
  - `integration-oauth-callback` → handshake OAuth Notion/Brevo/Instantly e scrittura cifrata in `tenant_integrations.credentials_vault_ref`
  - `user-invite` (admin only) → invita membri POD
- **n8n workflow**:
  - `sync-integrations-status` (cron 5m) → ping health di Notion/Brevo/Instantly/WABA → aggiorna `last_sync_at` e `status`
  - `user-offboarding` → su `auth.signOut` finale, cleanup token device
- **AI call**: nessuna in-page; eventuale "summary preferences" via Copilot è out-of-scope
- **Analytics event**:
  - `settings_viewed`, `settings_toggle_changed {toggle_id, value}`
  - `integration_clicked {name}`, `integration_status_changed {name, from, to}`
  - `support_link_clicked {target}`, `logout_clicked`
- **Audit log**: ogni cambio di integrazione, ruolo, permesso deve scrivere una riga in `audit_log` con `actor_id`, `action`, `entity`, `before`, `after`, `ip`, `user_agent`

### Sezioni Admin mancanti (da costruire)

Oggi la pagina è "Impostazioni utente". Per coprire il ruolo `admin` servono sotto-sezioni:

- Gestione utenti tenant (invite, role assign, deactivate)
- Gestione POD Teams (create/rename, spostamento membri)
- Gestione ruoli & permessi (matrice RBAC)
- Audit log (filtrabile per utente/entità/data)
- Billing / piano tenant
- Configurazione WABA (phone numbers, routing rules, webhook secret rotation)
- Data retention / GDPR (export, erase request)

---

## 7. UX/UI review

### Funziona bene

- Pattern iOS-like coerente con il resto del portale (NavBar + GroupedList + Chip)
- Card profilo leggibile, gerarchia chiara (nome serif 24px, ruolo, email mono, data iscrizione)
- Badge "Tu" sul proprio record nel POD Team è un tocco di UX gradevole
- Toggle accessibili (`role="switch"`, `aria-checked`, `aria-label` dedicate, `stopPropagation` sul click per non scattenare row click)
- `max-w-[840px]` mantiene leggibilità su desktop wide

### Attriti UX attuali

- CTA **"Esci dall'account" è un bottone fantasma**: nessun `onClick`. Si clicca e non succede nulla → grave violazione di aspettativa
- CTA **"Feedback" senza handler**: stesso problema
- Righe POD Team con `chevron: true` ma **nessun target di navigazione** — chevron suggerisce click ma l'interazione è muta
- **Nessuna persistenza toggle**: al refresh torna tutto a `true`. Se il dev clicca per disattivare alert critical e ricarica, il toggle mente
- Mancano feedback visivi: nessun toast "preferenza salvata", nessun loading, nessun error
- Stato integrazione "In config" (Instantly) senza CTA per completare la configurazione → vicolo cieco
- Nessun link/azione su WABA (il numero è read-only) — difficile capire se c'è un problema di routing
- Versione app `v6.1` è hardcoded: se in futuro si aggiorna lo schema, questa riga mentirà
- Tag title "Impostazioni" vs. route `/admin` vs. voce menu "Admin": **inconsistenza nomenclatura** (3 nomi diversi per la stessa pagina)
- Gating di ruolo è solo visuale in `SalesLayout`: chi conosce l'URL `/sales/admin` ci entra comunque

### Coerenza UI

- Tipografia: ok (serif per nomi, sans per body, mono per email — scelta premium coerente)
- Spacing: ok (`space-y-5` + padding card `lg`)
- Colore: ok; Chip verde/gialla/arancione usate nel canon del design system
- Mobile-first: ok, il `GroupedList` collassa bene su small screen; `max-w-[840px]` centra su wide; l'header 72px e la tab bar sono gestiti dal `SalesLayout`

---

## 8. Miglioramenti proposti (prioritizzati)

| #   | Priorità | Tipo    | Proposta                                                                                                                            | Effort |
| --- | -------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | P0       | UX      | Collegare handler a "Esci dall'account" (`supabase.auth.signOut()` + redirect `/login`)                                             | S      |
| 2   | P0       | SEC     | Enforce gating rotta `/sales/admin` lato router (guard che legge `user.role`) + RLS server-side: oggi chi conosce l'URL entra       | M      |
| 3   | P0       | UX      | Persistere i toggle su `user_notification_preferences` + toast di conferma                                                          | M      |
| 4   | P1       | UX      | Rimuovere chevron sulle righe POD Team finché non è cliccabile (o implementare il drawer membro)                                    | S      |
| 5   | P1       | UX      | CTA "Configura" sulla riga Instantly (status yellow) → porta al flusso OAuth                                                        | M      |
| 6   | P1       | UX      | Implementare form Feedback (modal con textarea + category + optional screenshot) → `POST /feedback`                                 | M      |
| 7   | P1       | INFO    | Normalizzare nomenclatura: scegliere tra "Impostazioni" / "Admin" / "Account" e usare lo stesso nome su route, menu e titolo        | S      |
| 8   | P2       | FEATURE | Sezione "Gestione utenti" visibile solo ad `admin`: lista tenant users + invite + role assign + deactivate                          | XL     |
| 9   | P2       | FEATURE | Sezione "Audit log" filtrabile (actor, action, entity, range data)                                                                  | L      |
| 10  | P2       | FEATURE | Sezione "POD Teams" con CRUD team e spostamento membri                                                                              | L      |
| 11  | P2       | UI      | Stato "ultimo sync" su ogni integrazione (`last_sync_at` human-readable) + pulsante "Riprova ora"                                   | M      |
| 12  | P2       | UX      | Sostituire `window.open` doc sales con navigazione interna se docs vivrà in-app; altrimenti aggiungere icona external-link visibile | S      |
| 13  | P3       | UI      | Sostituire footer hardcoded "v6.1" con `import.meta.env.VITE_APP_VERSION` + commit SHA                                              | S      |
| 14  | P3       | UX      | Aggiungere sezione "Theme" (light/dark/system) — coerente con direzione UI                                                          | M      |
| 15  | P3       | A11Y    | Focus ring visibile su tutte le righe `GroupedList` cliccabili; trap focus in modal quando introdotti                               | S      |
| 16  | P3       | FEATURE | Export dati personali (GDPR) + "cancella account" con conferma a doppio step                                                        | L      |

Legenda priorità: P0 bloccante · P1 alto · P2 medio · P3 nice-to-have
Legenda effort: S <1h · M 1-4h · L 1-2gg · XL >2gg

---

## 9. Aperti / Da chiarire

- **Scope del ruolo `admin`**: è admin di tenant (HR agency HireFast) o super-admin di piattaforma? La distinzione cambia quali sezioni servono.
- Il numero WABA è condiviso tra tutti i sales HireFast — la config è read-only per i POD Director o è modificabile da Admin?
- L'invito nuovi utenti passa da questa pagina o da un pannello separato (es. `/org-admin`)?
- Audit log: retention e chi può leggerlo (solo admin tenant o anche pod_director del POD coinvolto)?
- Logout: serve logout "da tutti i dispositivi" oltre al logout sessione corrente?
- Le preferenze notifiche sono per-device (solo push) o globali per utente?
- Le integrazioni (Notion/Brevo/Instantly) sono per-tenant o per-utente? Se per-tenant, un sales non-admin le vede ma non può modificarle: serve stato disabled/read-only?
- Policy per "Feedback": va in Slack #product, in un tool di ticketing, o in una tabella `feedback`?
- Come gestiamo multi-tenant per un utente che appartiene a più tenant (es. consulente)?
