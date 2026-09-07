# msg.Hello — Produkt-Intake-Dossier

> **Zweck dieser Datei:** Vollständige Wissensaufnahme über das Produkt **msg.Hello**, erstellt in einer Grilling-Session (Code-Exploration aller vier Repos + geklärte Produktentscheidungen). Diese Datei ist die **Übergabegrundlage für den Doku-Skill**, der daraus die Mintlify-Kundendoku generiert. Sie ist kein Kundendokument, sondern das interne Briefing.
>
> **Stand:** 2026-08-11 · Quelle: Ist-Zustand des Codes (nicht die veralteten `CLAUDE.md` / `neap-doku/`).

---

## 0. Verbindliche Vorgaben für den Doku-Skill

1. **Terminologie:** Durchgängig **msg.Hello** (Produkt), Hersteller **msg**. Die Namen **„NEAP" und „Neohelden" dürfen NICHT vorkommen** — es gab ein Rebranding, beide existieren nicht mehr. Interne Repo-/Registry-Namen (`neap-*`, `neap-authz-service`, `registry.neohelden.com`) sind Alt-Bestand und gehören nicht in die Kundendoku.
2. **Reihenfolge:** **Phase 1 = Kundendoku** (Mandanten/Anwender). **Phase 2 = Entwicklerdoku** (wie ein Kunde einen Connector implementiert). Dieses Dossier deckt beides ab, aber die *aktuelle* Aufgabe ist Phase 1.
3. **Reifegrad-Badges (Q5):** Jede Kernfunktion wird mit einem Badge gekennzeichnet:
   - ✅ **live** — funktioniert real und ist bedienbar
   - 🚧 **in Arbeit** — Gerüst existiert, Verhalten noch unvollständig/nicht verdrahtet
   - 📋 **geplant** — Roadmap, noch nicht gebaut
4. **Grundhaltung:** „gebaut ≠ verdrahtet". Viele Konzepte sind als Code-Gerüst vorhanden, aber zur Laufzeit nicht aktiv (siehe §5). Ehrlichkeit vor Marketing — die Kundendoku darf nichts als „live" verkaufen, was beim ersten Test auffällt.
5. **Positionierung (Q2):** **Branchenübergreifende Voice-Agent-Plattform.** Terminbuchung im Gesundheitswesen ist das *erste Beispiel*, nicht der Produktkern. Weitere Connector-Use-Cases folgen (Forderungsmanagement, Recruiting-Screening, Vertragsverifizierung, Handover an menschliche Agenten).
6. **Zielformat:** Mintlify-Doku-Site (`docs-mintlify/`, aktuell noch Starter-Kit). Zweisprachig de/en (beide Portale sind de/en).
7. **Alt-Doku:** `neap-doku/` wird **gelöscht, sobald die neue Doku steht** — bis dahin Faktenquelle. `CLAUDE.md` ist im selben Zug auf msg.Hello umzustellen.

---

## 1. Produktüberblick

**msg.Hello** ist eine B2B-SaaS-Plattform für **telefonische KI-Voice-Agents**. Ein Anrufer spricht mit einem Bot, der ein Anliegen versteht, einen konfigurierten Gesprächs-Flow durchläuft, bei Bedarf externe Systeme über **Connectoren** anbindet und den Anruf abschließt (Auflegen oder Weiterleiten an einen Menschen).

- **Hersteller/Betreiber:** msg
- **Mandanten (Kunden):** Organisationen, die Anrufe automatisiert entgegennehmen wollen — erster Referenz-Vertikal: Arztpraxen/MVZ/Fachzentren (z. B. Endokrinologikum Hamburg), perspektivisch branchenübergreifend.
- **Kernnutzen:** Entlastung der Telefonannahme, 24/7-Erreichbarkeit, automatisierte fachliche Prozesse via Connectoren.
- **Kernidee der E2E-Fähigkeit (Q12):** Durch Integration mit Connectoren löst die Plattform vollständige Fachprozesse Ende-zu-Ende (z. B. Terminbuchung inkl. Patienten-Identifikation, Verfügbarkeitssuche, Buchung). ➜ Als Kern-Feature positiv beschreiben, **Connector-Ausführung mit 🚧 markieren** (siehe §5).

### So läuft ein Anruf (laienverständlich, für Kapitel 1)
1. Ein Anrufer wählt die Rufnummer des Bots.
2. Der Anruf läuft über den Telefonie-Provider (**Twilio**) direkt zum Sprachmodell (**OpenAI Realtime**).
3. Der Bot begrüßt, versteht das Anliegen und folgt dem vom Mandanten gebauten **Flow**.
4. Für fachliche Schritte ruft der Bot **Tools** auf, die über **Connectoren** an die Systeme des Kunden angebunden sind.
5. Am Ende wird aufgelegt oder an einen Menschen weitergeleitet. Anruf, Transkript und Auswertung stehen danach im Portal.

---

## 2. Architektur-Realität (wichtig — korrigiert die Alt-Doku)

**Telefonie/Medien (Q6, bestätigt):** **Twilio zeigt per SIP direkt auf OpenAI.** OpenAI hostet Anruf **und** Medien. Der Call-Service ist reine **Control-Plane**:
- OpenAI schickt Webhook `realtime.call.incoming` → Call-Service `POST /sip-connection/webhook`.
- Call-Service akzeptiert via `POST {callsBaseURL}/{callId}/accept` (Webhook-Signatur mit `OPENAI_WEBHOOK_SECRET` geprüft).
- Steuerung/Events laufen über einen **WebSocket-Control-Kanal**.
- Aktionen: **FORWARD** → `POST /calls/{callId}/refer` (`tel:…`), **HANGUP** → `POST /calls/{callId}/hangup`.
- **Das Audio läuft NICHT durch den Call-Service.** ⇒ Das Sequenzdiagramm in `neap-doku/03` (mit „SIP→CallService→Audio") ist **veraltet** und darf nicht übernommen werden.

**Modell-Konfiguration (Ist):** Modell `gpt-realtime-1.5`, Stimme `marin` (wählbar: alloy/ash/ballad/cedar/coral/echo/marin/sage/shimmer/verse), Eingabe-Transkription `gpt-4o-transcribe` (Sprache `de`), Rauschunterdrückung `near_field`, Server-VAD/Turn-Detection, Barge-in pro Bot schaltbar. Separate Anruf-Zusammenfassung via `gpt-4.1-mini`.

**Komponentenlandschaft:** zwei Domänen — **Voice/Call** (Portal + Service + Postgres) und **IAM** (Portal + Service + Postgres). Call-Service ↔ IAM-Service via **gRPC** (`EntityService.GetEntity`, Auflösung von Benutzer-UUIDs zu Klarnamen). Auth über eigenen OIDC-Provider (IAM), Autorisierung über **OPA-Sidecar** je Service. **Redis ist dokumentiert, aber im Call-Service ungenutzt** — Session-Sync läuft über ein **Postgres-Outbox-Pattern**.

---

## 3. Rebranding-/Umbenennungs-Map (alt → neu)

| Alt (im Code/Doku) | Neu (in der Kundendoku) |
|---|---|
| NEAP / Neohelden Agentic AI Platform | **msg.Hello** |
| Neohelden (Betreiber) | **msg** |
| „Neo Call Insights" (internes FE-Label) | msg.Hello (Portal) |
| „Neo Auth Portal" / neap-authz-portal | Anmeldung & Benutzerverwaltung (integrierter Bereich, siehe Q11) |
| Rechte-Präfix `hello_…` | intern, nicht kundenseitig zeigen |

---

## 4. Vollständiges Feature-Inventar (nach Komponente, mit Reifegrad)

### 4.1 Voice-/Call-Produkt — Mandanten-Portal (`neap-call-portal`, Next.js/React)
Zentrale Kunden-Oberfläche. Alle Seiten mandanten-scoped unter `/tenants/[tid]/…`.

- **Home/Onboarding** ✅ — Willkommens-Hero, Onboarding-Checkliste, Bot-Auswahl-Dialog.
- **Bots verwalten** ✅ — Liste (Suche, Live-Version-Status), **Create-Bot-Wizard** (Grundeinstellungen, Base Instructions, Anrufparameter, Modell-Defaults), **Bot-Konfiguration** (Tabs: Grundeinstellungen inkl. Rufnummer, Instructions, Anrufparameter, KI-Modell-Parameter, Connectoren).
- **Flow-Editor** ✅ — echter **Node-Graph** (React Flow / @xyflow). State-Machine mit States **REGULAR / FORWARD / HANGUP**; pro State: Instructions, erlaubte Tools, benannte **Outputs** mit **Bedingungen** (Tool + responseCode + isNegated) → transitionTo. Import/Export, **Validierung** (Erreichbarkeit, gültige Übergänge), **Versionierung** (Draft → publish → Version, aktivieren, „als Draft öffnen"), read-only Version-Viewer.
- **Connector-Setup** (Konzept ✅ / Ausführung 🚧) — Liste verfügbarer Connector-Typen (Anzeigename, Beschreibung, Anzahl unterstützter Tools, Auth-Methoden-Chips) + aktive Connectoren. Dialog: **Basis-URL + Authentifizierung** (Felder aus `AuthenticationMethodSchema`, je Feld `isRequired`/`isSecret`), Tools aktivieren.
- **Test-Anruf im Browser** ✅ — WebRTC (Audio) + socket.io (Signaling) gegen `/flow-simulation/connect-frontend`. Läuft **live gegen den aktuellen Draft**, kein persistierter Call. 3-Pane-UI (Flow-State-Sidebar, Live-Chat, Event-Log).
- **Anrufe auswerten** ✅ — Liste (Filter: Datum, Rufnummer, Status), Detail mit **Transkript** (Dialog USER/SYSTEM), **Zusammenfassung**, **Kommentare** (Thread), **Zuweisung** (Assignee), **Tags**, Bearbeitungsstatus (OPEN/IN_PROGRESS/COMPLETED).
- **KPI-Dashboard** ✅ — Overview-Cards + Zeitreihen-Chart, Datumsbereich.
- **Einstellungen** ✅ — **Datenschutz/Retention** (Aufbewahrung Call-/Dialog-Tage), **Erscheinungsbild** (Primärfarbe), **Sprache**.
- **Auth** ✅ — NextAuth v4, eigener OIDC-Provider, PKCE/State/Nonce, Server-Sessions in Redis; Scopes `openid profile email tenant_permission tenant_custom_permission`.
- **Stub im FE** 🚧 — `hello_consumption-read` und `hello_retention_command-execute` als System-Scope angelegt, aber **nicht verdrahtet** (OIDC-Request fordert `permission`-Scope nicht an).

### 4.2 Voice-/Call-Produkt — Backend (`neap-call-service`, NestJS)
- **Anruf-Lifecycle** ✅ (nur **INBOUND**; OUTBOUND ist Daten-Platzhalter 🚧). SIP-Webhook → Bot-Lookup per Rufnummer → Runtime-Session (Start-State) → OpenAI-Realtime-Verbindung. Timer für max. Anrufdauer und max. Wartezeit auf Nutzereingabe.
- **Flow-Engine** ✅ — Draft (relational) → Version (immutable `jsonb` `VersionGraph`, `graphSchemaVersion=1`). Interpreter `VersionFlowEngineService.resolveNextState` (pur): Output matcht bei `tool === lastTool && responseCode === lastResponseCode` (negierbar); **ohne responseCode kein Übergang**; Zyklen erlaubt (nur Warnung); FORWARD/HANGUP terminal.
- **Prompt-Assembly** ✅ — Markdown-Sektionen: SYSTEM ROLE, GLOBAL RULES, CRITICAL RULES, STATE INSTRUCTIONS, PREVIOUS SLOT VALUES (Runtime-Slots als Variablen), ALLOWED TOOLS. Pro-Bot-Felder in `BotInstructionsEntity`.
- **Tools** — builtin **`classify`** ✅ (einziges funktionierendes Tool). 14 **Connector-Tools** (`lookup_patient`, `create_patient`, `get_specialties`, `get_appointment_types`, `get_available_dates`, `get_available_times`, `book_appointment`, `lookup_appointments`, `cancel_appointment`, …) 🚧 — deklarieren Schema + Response-Codes, **werfen aber `NotImplementedException`** (keine Ausführung).
- **Connector-Runtime** (Konfig ✅ / Ausführung 🚧) — System-Katalog `CONNECTOR_TYPES`: **nur `PVS_DEFAULT`** existiert (APIKey-Auth). Pro-Bot: `BotConnectorConfigEntity` (baseUrl), `BotConnectorAuthEntity` (AES-256-GCM-verschlüsselt), `BotToolConnectorEntity` (Tool→Connector-Mapping). **Kein echter HTTP-Call an ein externes System heute.** Standard-OpenAPI-Connector + Doctolib/ClickDoc/Principa existieren im Code **nicht**.
- **Retention** (🚧 Cron kommt) — `RetentionService.enforceRetention` löscht Calls/Dialoge älter als Mandanten-Policy (Default 90 Tage); heute nur via `POST /commands/retention-delete`, **automatischer Cronjob ist beschlossen** (Q10).
- **Billing/Consumption** — `GET /consumption` (Minuten + Anrufanzahl je Mandant, CSV). **Nicht in der Kundendoku** (Q9: Pricing raus).
- **Datenmodell** — `CallEntity` (Richtung, Dauer, Status, Assignee, Summary, 1:1 Dialog, M:N Tags, 1:N Comments) → `DialogEntity` → `DialogMessageEntity` (USER/SYSTEM). Runtime: `RuntimeSessionEntity`/`…SlotEntity`/`…SyncOutboxEntity`. Bot-Aggregat: `BotEntity` (+ Instructions, ModelDefaults, CallParameters, Draft, Versions).

### 4.3 IAM — Portal (`neap-iam-portal`, Next.js/React)
Für die Kundendoku (Q11) als **integrierter Bereich „Anmeldung & Benutzerverwaltung"** führen, nicht als eigenes Produkt. Zwei Scopes: **System** (Plattform-Admin, msg-intern) und **Tenant** (Kunde selbst).
- **Kunden-relevant (Tenant-Scope)** ✅ — Benutzer einladen (`Invitations`), Rollen zuweisen, **Tenant-Custom-Rollen/-Rechte**, Mandanten-Wechsel.
- **msg-intern (System-Scope)** — Mandanten anlegen/löschen, System-Benutzer, OIDC-Clients, System-/Tenant-Rollen-Templates. *(Nicht Kundendoku, ggf. Ops.)*

### 4.4 IAM — Backend (`neap-iam-service`, NestJS; intern noch `neap-authz-service`)
- **OIDC** — ist **gleichzeitig OIDC-Provider und Broker** zu einem Upstream-IdP. Login-Formular nimmt E-Mail → leitet an Upstream-IdP → Callback `…/consume` legt Benutzer bei Erstlogin automatisch an → eigener Authorization-Code. Grants: authorization_code, refresh_token, client_credentials (letztere zwei implementiert, aber im Discovery nicht beworben). PKCE S256.
- **Upstream-IdP** — generisch-OIDC via `DEFAULT_OIDC_*`. **Azure CIAM ist Ziel/Prod** (Login-Hint-Plumbing passt dazu), **Keycloak in Dev**. **Multi-IdP pro Domain ist NICHT verdrahtet** 📋 (Resolver liefert immer den einen Default-IdP, obwohl DB-Struktur existiert).
- **RBAC** — vier Ebenen: System, Tenant (Standard), Tenant-Custom (frei definierbar), **Bot** 🚧. Rollen-Vererbung (`extendedRoles`, Zyklen verhindert). **Bot-Level-RBAC ist Stub** 📋: `bots`-Claim existiert, wird aber nie befüllt.
- **Tenancy** — `TenantEntity` + Assoziationstabellen; Mitgliedschaft **lazy** (Benutzer wird bei Login mit passender Einladung dem Mandanten hinzugefügt). **Einladungen** ✅ (`POST /tenants/{tid}/invitations`, E-Mail via SMTP/SES, async).

---

## 5. „Gebaut ≠ verdrahtet" — explizite Reifegrad-Liste

| Feature | Status | Für Kundendoku |
|---|---|---|
| Bot begrüßen / klassifizieren (`classify`) / weiterleiten / auflegen | ✅ live | normal beschreiben |
| Flow-Editor, Versionierung, Test-Anruf, Anruf-Auswertung, KPIs, Einstellungen | ✅ live | normal beschreiben |
| Benutzer/Team-Verwaltung (Einladung, Rollen) auf eigenem Tenant | ✅ live | eigenes Kapitel (Q13) |
| **Connector-Ausführung (Tools rufen externes System)** | 🚧 in Arbeit | E2E-Konzept positiv, Ausführung mit 🚧 |
| Standard-Connector-API (Kunde implementiert Spec) | Spec vorhanden, Runtime 🚧 | Konzept nennen, Details → Phase 2 |
| Fertige Direkt-Integrationen (Doctolib/ClickDoc/Principa …) | 📋 geplant | als kommend nennen |
| Automatische Retention-Löschung (Cron) | 🚧 kommt | als kommend/planned |
| Outbound-Calls | 📋 geplant | nicht/als Roadmap |
| Bot-Level-Rechte | 📋 geplant | Roadmap-Anhang |
| Multi-IdP-Föderation pro Domain | 📋 geplant | Roadmap-Anhang |
| Consumption/Billing | vorhanden, FE-Stub | **raus** (Q9) |

---

## 6. Eingefrorenes Gerüst — Kundendoku (Phase 1)

1. **Was ist msg.Hello?** — Nutzen, branchenübergreifend, „so läuft ein Anruf" (§1). ✅
2. **Erste Schritte** — Onboarding-Realität (Q13.2): *msg legt den Mandanten an und onboardet den Mandanten-Admin, danach übernimmt der Kunde selbst.* Anmeldung/SSO, erster Bot. ✅
3. **Bots verwalten** — Grundeinstellungen, Instructions, Anruf- & Modell-Parameter, Rufnummer. ✅
4. **Gesprächs-Flows bauen** — Node-Editor, State-Typen, Tools, Bedingungen, Validierung, Draft→Version→Aktivieren. ✅
5. **Connectoren einrichten** — nur **Bedien-Ebene** (Q14): Typ wählen, Basis-URL + Zugangsdaten, Tools aktivieren; Standard-Connector *und* Direkt-Integrationen erwähnen. Konzept ✅ / Ausführung 🚧. *Implementierung → Phase 2.*
6. **Bot testen** — Test-Anruf im Browser (WebRTC). ✅
7. **Anrufe auswerten** — Liste, Transkript, Zusammenfassung, Kommentare, Zuweisung, Tags, KPI-Dashboard. ✅
8. **Einstellungen** — Retention/Datenschutz (Auto-Löschung 🚧), Erscheinungsbild, Sprache. ✅
9. **Benutzer & Team** — Kunde verwaltet selbst auf eigenem Tenant: einladen, Rollen/Custom-Rollen. ✅
10. **Glossar** — siehe §7.
- **Roadmap-Anhang** (📋): Bot-Level-Rechte, Multi-IdP, Outbound, weitere Direkt-Connectoren.
- **Nicht enthalten:** Pricing/Billing, Infrastruktur/Betrieb, Connector-Implementierung.

**Phase 2 (Entwicklerdoku, später):** Standard-Connector-API-Referenz (`connector-standard-api.yaml`, 10 Endpunkte, 2-stufige Fachgebiet→Terminart-Hierarchie, 2 Availability-Endpunkte dates/times, Cursor-Paginierung, InsuranceType GKV/PKV/Selbstzahler), Auth-Methoden, Response-Codes/Fehlermeldungen, Test-Tool.

---

## 7. Glossar (für Kapitel 10)

| Begriff | Bedeutung |
|---|---|
| **Mandant / Tenant** | Kunde der Plattform; zentrale Isolationsgrenze. Wird von msg bereitgestellt. |
| **Bot** | Konfigurierter Voice-Agent, an eine Rufnummer gebunden. |
| **Flow** | Gesprächsablauf als Zustandsgraph (States, Übergänge, Tools). |
| **State** | Gesprächszustand: REGULAR (mit Tools/Übergängen), FORWARD (Weiterleitung), HANGUP (Ende). |
| **Draft / Version** | Bearbeitbarer Entwurf vs. veröffentlichter, unveränderlicher Snapshot; „Live-Version" ist die aktive. |
| **Tool** | Vom Modell aufrufbare Aktion (z. B. `classify`, `book_appointment`), meist über einen Connector. |
| **Connector** | Adapter zwischen Bot und externem System; Standard-Connector (Kunde implementiert) oder Direkt-Integration (von msg). |
| **Response-Code** | Ergebnis eines Tools, das den Flow-Übergang steuert. |
| **Test-Anruf** | Browser-basierter Voice-Test gegen den aktuellen Draft (WebRTC). |
| **Retention** | Pro Mandant konfigurierbare Aufbewahrungsfrist für Anruf-/Dialogdaten. |

---

## 8. Hinweise an den Doku-Skill

- Schreibe **Phase 1 (Kundendoku)** nach §6. Bediener-Perspektive, keine internen Architekturdetails außer dem laienverständlichen „so läuft ein Anruf".
- Setze **Reifegrad-Badges** konsequent (§0.3, §5).
- **Keine** Alt-Namen (§0.1, §3).
- Quellen für Detailtreue: `neap-call-portal/src/app/api/internal/…` (BFF spiegelt den vollständigen Feature-Umfang), `neap-call-service/openapi.yaml`, `neap-iam-service/openapi.yaml`, `connector-standard-api.yaml` (nur Phase 2).
- Nach Fertigstellung der neuen Doku: **`neap-doku/` löschen** und **`CLAUDE.md`** auf msg.Hello umstellen.
