# Anforderungsspezifikation: Knuffel
## Multiplayer Kniffel Web-Anwendung

**Projektziel:** Entwicklung einer webbasierten Multiplayer-Variante des Würfelspiels Kniffel (Yahtzee) als Microservices-Architektur zur Demonstration moderner Web-Architekturkonzepte.

**Projektkontext:** Universitätskurs "Web Application Architecture", Wintersemester 2024/25

---

## 1. Stakeholder

| Rolle | Beschreibung |
|-------|--------------|
| **Spieler (Gast)** | Nutzer ohne Account, erstellt/joint Lobbies mit Custom Username |
| **Lobby-Leiter** | Spieler, der die Lobby erstellt hat oder zum Leiter ernannt wurde |
| **System** | Backend-Services zur Spielverwaltung und -logik |
| **Entwicklerteam** | Studierende, die die Anwendung entwickeln |

---

## 2. Functional Requirements (MVP)

### FR-1: Benutzer-Verwaltung (Gast-Modus)

#### FR-1.1: Username-Eingabe
- **Als** Gast **möchte ich** einen Custom Username eingeben **damit** ich im Spiel identifizierbar bin
- **Akzeptanzkriterien:**
  - Username wird auf der Startseite beim Erstellen/Beitreten einer Lobby eingegeben
  - Username bleibt aktiv, bis ein neues Spiel von der Startseite gestartet wird
  - Username muss 3-20 Zeichen lang sein
  - Keine Duplikate innerhalb einer Lobby

#### FR-1.2: Session-Verwaltung
- **Als** System **muss ich** Gast-Sessions per JWT-Token verwalten
- **Akzeptanzkriterien:**
  - JWT-Token wird als HTTP-Only Cookie gespeichert
  - Token enthält: User-ID, Username, Session-Timestamp
  - Token-Gültigkeit: Bis Spiel beendet oder Nutzer verlässt explizit

#### FR-1.3: Automatisches Rejoin
- **Als** Spieler **möchte ich** nach Verbindungsabbruch automatisch zurück ins Spiel **damit** ich weiterspielen kann
- **Akzeptanzkriterien:**
  - Bei Page-Refresh: Automatische Weiterleitung zur Spielseite
  - Spieler wird als "wieder verbunden" markiert
  - Spielzustand wird vom letzten Stand wiederhergestellt

---

### FR-2: Lobby-Verwaltung

#### FR-2.1: Lobby erstellen
- **Als** Spieler **möchte ich** eine neue Lobby erstellen **damit** andere Spieler beitreten können
- **Akzeptanzkriterien:**
  - Generierung eines eindeutigen 6-stelligen Join-Codes (z.B. "ABC123")
  - Lobby-Ersteller wird automatisch Lobby-Leiter
  - Lobby unterstützt 2-6 Spieler
  - Lobby-Status: "Wartend" (vor Spielstart)

#### FR-2.2: Lobby beitreten
- **Als** Spieler **möchte ich** über einen Join-Code einer Lobby beitreten
- **Akzeptanzkriterien:**
  - Eingabe des 6-stelligen Codes auf der Startseite
  - Fehlerbehandlung: "Lobby nicht gefunden" oder "Lobby voll"
  - Sofortige Weiterleitung zur Lobby-Ansicht
  - Alle Spieler in der Lobby werden über neuen Beitritt benachrichtigt

#### FR-2.3: Lobby-Übersicht
- **Als** Spieler **möchte ich** in der Lobby alle Teilnehmer sehen
- **Anzeige:**
  - Liste aller Spieler mit Username
  - Kennzeichnung des Lobby-Leiters (z.B. Stern-Icon)
  - Anzeige der Spieleranzahl (z.B. "3/6 Spieler")
  - Join-Code prominent angezeigt zum Teilen

#### FR-2.4: Spieler kicken
- **Als** Lobby-Leiter **möchte ich** Spieler aus der Lobby entfernen
- **Akzeptanzkriterien:**
  - "Kick"-Button nur für Lobby-Leiter sichtbar
  - Gekickter Spieler erhält Benachrichtigung und wird zur Startseite weitergeleitet
  - Nur während der Wartephase möglich (nicht im laufenden Spiel)

#### FR-2.5: Spiel starten
- **Als** Lobby-Leiter **möchte ich** das Spiel starten **wenn** mindestens 2 Spieler bereit sind
- **Akzeptanzkriterien:**
  - "Spiel starten"-Button nur für Leiter sichtbar
  - Minimum: 2 Spieler, Maximum: 6 Spieler
  - Nach Start: Lobby wird geschlossen (kein Nachjoinen möglich)
  - Alle Spieler werden zur Spielseite weitergeleitet
  - Zufällige Zugreihenfolge wird bestimmt

#### FR-2.6: Lobby-Leiter-Wechsel
- **Als** System **muss ich** bei Disconnect des Leiters einen neuen Leiter bestimmen
- **Akzeptanzkriterien:**
  - Neuer Leiter: Ältester noch verbundener Spieler in der Lobby
  - Alle Spieler werden über Leiterwechsel benachrichtigt
  - Alter Leiter wird bei Rejoin NICHT wieder Leiter
  - Falls letzter Spieler die Lobby verlässt: Lobby wird gelöscht

---

### FR-3: Spielmechanik

#### FR-3.1: Spielstart und Zugreihenfolge
- **Als** System **muss ich** eine faire Zugreihenfolge festlegen
- **Akzeptanzkriterien:**
  - Zugreihenfolge wird bei Spielstart zufällig bestimmt
  - Reihenfolge bleibt für die gesamte Partie konstant
  - Aktueller Spieler wird visuell hervorgehoben

#### FR-3.2: Würfelmechanik
- **Als** aktiver Spieler **möchte ich** bis zu 3-mal würfeln
- **Würfel-Regeln:**
  - 5 Würfel mit Werten 1-6
  - Nach jedem Wurf: Würfel können fixiert/gelöst werden
  - Fixierte Würfel werden beim nächsten Wurf nicht neu gewürfelt
  - Spieler kann Zug vor dem 3. Wurf beenden (Feld wählen)

#### FR-3.3: Würfel fixieren/lösen
- **Als** aktiver Spieler **möchte ich** einzelne Würfel für den nächsten Wurf behalten
- **Akzeptanzkriterien:**
  - Klick auf Würfel: Toggle zwischen fixiert/nicht fixiert
  - Fixierte Würfel visuell gekennzeichnet (z.B. Rahmen)
  - Alle 5 Würfel können gleichzeitig fixiert sein
  - Fixierung wird nach Feldwahl zurückgesetzt

#### FR-3.4: Feld-Auswahl
- **Als** aktiver Spieler **möchte ich** nach maximal 3 Würfen ein Feld auf dem Block auswählen
- **Kniffel-Block (13 Felder):**
  
  **Oberer Teil:**
  - Einser (nur 1er zählen)
  - Zweier (nur 2er zählen)
  - Dreier (nur 3er zählen)
  - Vierer (nur 4er zählen)
  - Fünfer (nur 5er zählen)
  - Sechser (nur 6er zählen)
  - **Bonus:** +35 Punkte bei ≥63 Punkten im oberen Teil
  
  **Unterer Teil:**
  - Dreierpasch (3 gleiche, Summe aller Würfel)
  - Viererpasch (4 gleiche, Summe aller Würfel)
  - Full House (3+2 gleiche, 25 Punkte)
  - Kleine Straße (4 aufeinanderfolgende, 30 Punkte)
  - Große Straße (5 aufeinanderfolgende, 40 Punkte)
  - Kniffel (5 gleiche, 50 Punkte)
  - Chance (Summe aller Würfel)

- **Zusatzregel - Mehrfach-Kniffel:**
  - Wenn Spieler bereits Kniffel-Feld ausgefüllt hat (50 Punkte)
  - UND nochmal 5 gleiche Würfel hat
  - DANN: +50 Bonuspunkte + freie Feldwahl im oberen Teil (oder streichen)

- **Akzeptanzkriterien:**
  - Nur leere Felder sind auswählbar
  - Punkteberechnung erfolgt automatisch
  - **Streichen möglich:** Spieler kann Feld mit 0 Punkten füllen
  - Nach Feldwahl: Nächster Spieler ist dran
  - Würfel-Fixierungen werden zurückgesetzt

#### FR-3.5: Gemeinsamer Kniffel-Block
- **Als** Spieler **möchte ich** alle Punktestände in einem gemeinsamen Block sehen
- **Layout:**
  ```
  ┌────────────────┬─────────┬─────────┬─────────┬─────────┐
  │ Feld           │ Alice   │ Bob     │ Charlie │ Diana   │
  ├────────────────┼─────────┼─────────┼─────────┼─────────┤
  │ Einser         │ 3       │ -       │ 5       │ -       │
  │ Zweier         │ 8       │ -       │ -       │ 10      │
  │ ...            │ ...     │ ...     │ ...     │ ...     │
  ├────────────────┼─────────┼─────────┼─────────┼─────────┤
  │ Summe Oben     │ 11      │ 0       │ 5       │ 10      │
  │ Bonus          │ -       │ -       │ -       │ -       │
  ├────────────────┼─────────┼─────────┼─────────┼─────────┤
  │ Gesamtsumme    │ 11      │ 0       │ 5       │ 10      │
  └────────────────┴─────────┴─────────┴─────────┴─────────┘
  ```
- **Akzeptanzkriterien:**
  - Jeder Spieler hat eine Spalte
  - Ausgefüllte Felder zeigen Punkte
  - Leere Felder zeigen "-"
  - Summen werden automatisch berechnet
  - Aktuelle Spieler-Spalte wird hervorgehoben

#### FR-3.6: Timeout-Mechanismus
- **Als** System **muss ich** inaktive Spieler automatisch überspringen
- **Akzeptanzkriterien:**
  - **Timeout:** 40 Sekunden pro Zug
  - **Timer resettet bei jeder Interaktion** (Würfeln, Würfel fixieren)
  - Countdown wird allen Spielern angezeigt
  - Bei Timeout: Zug wird komplett übersprungen
  - Spieler wird als "inaktiv" markiert (visuell gekennzeichnet)
  - Bei Reconnect: Spieler wird wieder als "aktiv" markiert
  - Inaktive Spieler werden in nächsten Runden übersprungen, bis sie sich verbinden

#### FR-3.7: Spieler-Disconnect während des Spiels
- **Als** System **muss ich** Disconnects behandeln
- **Akzeptanzkriterien:**
  - Disconnect → Spieler als "inaktiv" markiert
  - Spiel läuft weiter (andere Spieler unbeeinträchtigt)
  - Rejoin möglich: Automatische Rückkehr zum aktuellen Spielstand
  - Inaktive Spieler werden in Zugreihenfolge übersprungen

#### FR-3.8: Spieler verlässt Spiel freiwillig
- **Als** Spieler **möchte ich** eine laufende Runde verlassen können
- **Akzeptanzkriterien:**
  - "Spiel verlassen"-Button auf Spielseite
  - Bestätigungs-Dialog: "Möchtest du wirklich verlassen?"
  - Nach Verlassen: Spieler wird als "ausgeschieden" markiert
  - Spieler kann nicht mehr rejoinen
  - Spiel läuft weiter, Spieler-Spalte bleibt im Block (grau ausgegraut)

#### FR-3.9: Spiel vorzeitig beenden (Lobby-Leiter)
- **Als** Lobby-Leiter **möchte ich** das Spiel vorzeitig beenden
- **Akzeptanzkriterien:**
  - "Spiel beenden"-Button nur für Leiter sichtbar (während des Spiels)
  - Bestätigungs-Dialog: "Spiel wirklich beenden?"
  - Alle Spieler werden zur Endeseite mit aktuellem Zwischenstand weitergeleitet

---

### FR-4: Spielende

#### FR-4.1: Spielende erkennen
- **Als** System **muss ich** das Spielende feststellen
- **Bedingung:** Alle Spieler haben alle 13 Felder ausgefüllt ODER Leiter beendet Spiel vorzeitig
- **Akzeptanzkriterien:**
  - Nach letzter Feld-Auswahl: Automatische Berechnung der Endpunkte
  - Automatische Weiterleitung aller Spieler zur Endeseite (max. 2 Sekunden Verzögerung)

#### FR-4.2: Endeseite - Rangliste
- **Als** Spieler **möchte ich** das Spielergebnis sehen
- **Anzeige:**
  - Sortierte Liste aller Spieler nach Punktzahl (absteigend)
  - Platzierung (1., 2., 3., ...)
  - Username und Gesamtpunktzahl
  - Hervorhebung des Gewinners (z.B. Gold-Badge)
  - Detaillierte Punkteverteilung (oberer Teil, Bonus, unterer Teil)

#### FR-4.3: Nochmal spielen
- **Als** Spieler **möchte ich** mit den gleichen Spielern eine neue Runde starten
- **Akzeptanzkriterien:**
  - "Nochmal spielen"-Button auf Endeseite
  - Alle Spieler werden in neue Lobby verschoben
  - Neue Lobby-ID wird generiert
  - Alter Lobby-Leiter bleibt Leiter
  - Status zurück auf "Wartend"
  - Spieler können aus neuer Lobby auch verlassen/neue können joinen

#### FR-4.4: Spiel verlassen (von Endeseite)
- **Als** Spieler **möchte ich** zur Startseite zurückkehren
- **Akzeptanzkriterien:**
  - "Zur Startseite"-Button
  - Session wird beendet
  - Username wird zurückgesetzt (bei nächstem Spiel neu eingeben)

---

### FR-5: Real-Time Kommunikation

#### FR-5.1: Live-Updates
- **Als** Spieler **möchte ich** Änderungen in Echtzeit sehen
- **Events die gesendet werden müssen:**
  - Spieler joint/verlässt Lobby
  - Lobby-Leiter wechselt
  - Spiel startet
  - Spieler würfelt
  - Würfel werden fixiert/gelöst
  - Spieler wählt Feld
  - Punkte werden aktualisiert
  - Spieler wird inaktiv/aktiv
  - Timeout-Countdown
  - Spiel endet

- **Technologie:** Server-Sent Events (SSE) oder WebSockets
- **Akzeptanzkriterien:**
  - Latenz < 500ms für kritische Events (Würfeln, Feld wählen)
  - Automatisches Reconnect bei Verbindungsabbruch
  - Graceful Degradation bei Netzwerkproblemen

---

### FR-6: Datenpersistenz

#### FR-6.1: Spielstand speichern
- **Als** System **muss ich** aktive Spiele in der Datenbank speichern
- **Gespeichert werden:**
  - Lobby-Status (Wartend, Laufend, Beendet)
  - Spieler (User-IDs, Usernames, Status: aktiv/inaktiv/ausgeschieden)
  - Kniffel-Block (alle ausgefüllten Felder pro Spieler)
  - Aktueller Spieler
  - Würfelzustand (aktuelle Würfel, Fixierungen, Wurf-Count)
  - Timestamps (Spielstart, letzter Zug)

- **Akzeptanzkriterien:**
  - Spielstand wird nach jedem Zug gespeichert
  - Bei Server-Neustart: Aktive Spiele können weitergespielt werden
  - Abgeschlossene Spiele werden archiviert (für spätere Stretch-Goals)

---

## 3. Functional Requirements (Stretch Goals)

### FR-S1: Account-System (Google SSO via OIDC)
- Login mit Google-Account über Okta/OIDC
- Persistente User-Profile
- Spielstatistiken über Sessions hinweg

### FR-S2: Freunde-System
- Freunde hinzufügen/entfernen
- Private Lobbies (nur Freunde können joinen)
- Freunde zu Lobby einladen (Push-Benachrichtigung)

### FR-S3: Matchmaking
- **Random Matchmaking:** Automatisches Zusammenführen von suchenden Spielern
- **ELO-basiertes Matchmaking:** Spieler mit ähnlichem Skill-Level
- Matchmaking-Queue mit geschätzter Wartezeit

### FR-S4: Globale Rangliste
- Persistente Speicherung aller Spielergebnisse
- Leaderboard (Top 100 Spieler nach Durchschnittspunkten)
- Persönliche Statistiken (Spiele gespielt, Win-Rate, Kniffel-Count)

### FR-S5: Bot-Teilnehmer
- KI-Spieler mit konfigurierbarer Schwierigkeit
- Bot kann Lobby auffüllen, wenn zu wenig menschliche Spieler

---

## 4. Non-Functional Requirements

### NFR-1: Performance
- **NFR-1.1:** API-Response-Zeit < 200ms (95. Perzentil)
- **NFR-1.2:** Seitenladezeit < 2 Sekunden
- **NFR-1.3:** Gleichzeitige Spiele: Mindestens 100 parallele Lobbies
- **NFR-1.4:** Würfel-Animation < 1 Sekunde

### NFR-2: Skalierbarkeit
- **NFR-2.1:** Microservices können horizontal skaliert werden
- **NFR-2.2:** Datenbank unterstützt Partitionierung (für Stretch-Goals)
- **NFR-2.3:** Stateless Services (außer Game-State)

### NFR-3: Verfügbarkeit
- **NFR-3.1:** Uptime > 95% (für MVP akzeptabel, da Uni-Projekt)
- **NFR-3.2:** Graceful Degradation bei Teilausfall von Services
- **NFR-3.3:** Automatisches Reconnect bei Netzwerkproblemen

### NFR-4: Sicherheit
- **NFR-4.1:** JWT-Tokens mit HMAC-SHA256 signiert
- **NFR-4.2:** HTTP-Only Cookies (kein XSS)
- **NFR-4.3:** HTTPS in Produktion (TLS 1.3)
- **NFR-4.4:** Input-Validierung (z.B. Username-Length, Join-Code-Format)
- **NFR-4.5:** Rate-Limiting (z.B. max 10 API-Calls/Sekunde pro User)

### NFR-5: Usability
- **NFR-5.1:** Responsive Design (Desktop, Tablet, Mobile)
- **NFR-5.2:** Browser-Support: Chrome, Firefox, Safari, Edge (letzte 2 Versionen)
- **NFR-5.3:** Intuitive UI ohne Tutorial notwendig
- **NFR-5.4:** Fehler-Meldungen verständlich und hilfreich

### NFR-6: Wartbarkeit
- **NFR-6.1:** Code-Dokumentation (Swagger für APIs)
- **NFR-6.2:** Unit-Test-Coverage > 70%
- **NFR-6.3:** Logging aller kritischen Events (strukturiertes Logging)
- **NFR-6.4:** Service-Health-Checks für Monitoring

### NFR-7: Technologie-Stack
- **Backend:** Go (Microservices)
- **Frontend:** React
- **Datenbank:** PostgreSQL
- **Real-Time:** **SSE (Server-Sent Events) + REST API**
- **Auth:** JWT in HTTP-Only Cookies (MVP), OIDC/Okta (Stretch)
- **Deployment:** Docker + Docker Compose
- **CI/CD:** GitHub Actions

---

## 5. Geklärte Fragen & Entscheidungen

### 5.1 Team & Organisation

**✅ F1: Teamgröße und Rollenverteilung**
- **3 Personen** ohne feste Rollen
- Flexible Aufgabenverteilung nach Bedarf

**✅ F2: Zeitrahmen**
- **4 Monate** bis Ende der Vorlesungszeit
- Zwischenmeilensteine nach Bedarf

**✅ F3: Arbeitsweise**
- **Kanban** als Basis
- Bei Bedarf Wechsel zu **Scrum** für mehr Struktur
- Code-Review nach Bedarf

---

### 5.2 Technische Infrastruktur

**✅ F4: Deployment & Hosting**
- **Linux-Server an der Uni** (2 CPU Cores, 4GB RAM)
- Deployment via **Docker**
- **CI/CD** über GitHub Actions

**✅ F5: Datenbank**
- **PostgreSQL** selbst gehostet (Docker)
- Schema-Design wird später erstellt

**✅ F6: CI/CD**
- **Automatisierte CI/CD Pipeline** mit **GitHub Actions**
- Automated Testing + Deployment auf Uni-Server

**✅ F7: Monitoring & Logging**
- **Console-Logging** ausreichend für MVP
- Strukturiertes Logging optional später

---

### 5.3 Technische Detailentscheidungen

**✅ F8: Real-Time Kommunikation → ENTSCHEIDUNG: SSE + REST**

**Gewählt:** Server-Sent Events (SSE) + REST API

**Begründung:**
- ✅ Passt besser zu den Architekturmustern der Vorlesungsfolien (REST, SOA)
- ✅ HTTP-Only Cookie Auth einfach umsetzbar (bei jedem Request)
- ✅ Klare Trennung: REST für Actions, SSE für Live-Updates
- ✅ Gut dokumentierbar mit OpenAPI/Swagger
- ✅ Demonstriert RESTful API Design explizit

**Architektur:**
```
React Frontend
    ├─► REST API (POST /api/game/:id/roll, etc.)
    │   └─► JWT-Cookie bei jedem Request
    │
    └─► SSE Stream (GET /api/game/:id/events)
        └─► JWT-Cookie beim Connection-Aufbau
        
Backend Broadcasting:
  1. REST-Handler verarbeitet Action
  2. Speichert in DB
  3. Published Event zu SSE-Broker
  4. SSE-Broker sendet an alle Clients
```

**REST-Endpunkte:**
- `POST /api/lobby/create` - Lobby erstellen
- `POST /api/lobby/:id/join` - Lobby beitreten
- `POST /api/lobby/:id/start` - Spiel starten
- `POST /api/game/:id/roll` - Würfeln
- `POST /api/game/:id/toggle-dice/:index` - Würfel fixieren
- `POST /api/game/:id/select-field` - Feld auswählen
- `GET /api/game/:id/state` - Spielzustand abrufen

**SSE-Events:**
- `GET /api/game/:id/events` - Event-Stream
  - Events: `player_joined`, `dice_rolled`, `field_selected`, `turn_changed`, `game_ended`, etc.

---

**✅ F9: JWT-Implementierung**
- Library: **`golang-jwt/jwt`**
- Token-Expiry: **24h** für Gast-Sessions
- Refresh-Token: Nur für **OIDC-Accounts** (Stretch-Goal)
- Single-Token ausreichend für MVP

**✅ F10: Join-Code Format**
- **6-stellig alphanumerisch** (z.B. `ABC123`)
- Groß-/Kleinschreibung egal (intern uppercase)

---

### 5.4 UI/UX Details

**✅ F11: Responsive Design Prioritäten**
- **Desktop** = Hauptfokus
- Tablet/Mobile = Nice-to-have (später)

**✅ F12: UI-Framework**
- **Tailwind CSS** (geplant, noch nicht final)
- Plain React ansonsten

**✅ F13: Würfel-Animation**
- **Slot-Machine Style** (keine klassischen Würfel)
- Animation mit CSS/React Transitions

**✅ F14: Sound-Effekte**
- **Nicht für MVP** (kann später ergänzt werden)

---

### 5.5 Spielregeln-Klärungen

**✅ F15: Kniffel-Zusatzregel (Mehrfach-Kniffel)**
- **JA, implementieren!**
- Wenn Spieler bereits Kniffel (50 Punkte) hat und nochmal 5 gleiche würfelt:
  - +50 Bonuspunkte
  - Freie Feldwahl im oberen Teil (oder streichen)

**✅ F16: Inaktivitäts-Timeout Details**
- **40 Sekunden** pro Zug (nicht 60!)
- Timer **resettet bei jeder Interaktion** (Würfeln, Würfel fixieren)
- Zählt ab Zug-Beginn
- Bei Timeout: Zug wird übersprungen, Spieler als "inaktiv" markiert

---

### 5.6 Admin & Testing

**✅ F17: Admin-Interface für Debugging**
- **Nicht benötigt für MVP**
- Backend-Logging ausreichend

**✅ F18: Test-Accounts/Lobbies**
- **Manuelles Testen** mit mehreren Browser-Tabs
- Keine Bot-Spieler für Testing nötig

---

### 5.7 Dokumentation

**✅ F19: API-Dokumentation**
- **OpenAPI/Swagger** gewünscht
- Automatische Generierung aus Code (z.B. `swag` für Go)

**✅ F20: Architektur-Diagramme**
- **PlantUML** oder **Mermaid** (Markdown-integriert)
- C4-Model für Architektur-Übersicht optional

---

## 6. Zusammenfassung der Entscheidungen

### 🎯 MVP-Scope (finale Version)

**Team & Zeitrahmen:**
- 3 Personen, 4 Monate
- Kanban, bei Bedarf Scrum
- Keine festen Rollen

**Infrastruktur:**
- Linux-Server Uni (2 Cores, 4GB RAM)
- Docker + Docker Compose
- GitHub Actions CI/CD
- PostgreSQL in Docker

**Tech-Stack:**
- Backend: Go + SSE + REST
- Frontend: React + (Tailwind CSS)
- Auth: JWT in HTTP-Only Cookies
- Datenbank: PostgreSQL
- Docs: OpenAPI/Swagger

**Spielregeln-Anpassungen:**
- Timeout: 40 Sekunden (resettet bei Interaktion)
- Mehrfach-Kniffel: Ja, implementieren
- Würfel: Slot-Machine Style

**Nicht im MVP:**
- Sound-Effekte
- Admin-Interface
- Mobile-Optimierung
- Bot-Spieler
