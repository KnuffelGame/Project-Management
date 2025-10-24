# Datenflussdiagramme: Knuffel
## Multiplayer Kniffel Web-Anwendung

**Version:** 1.0  
**Datum:** 24.10.2025  
**Status:** ✅ Finalisiert

---

## Übersicht

Dieses Dokument beschreibt die **Hauptdatenflüsse** zwischen den Services der Knuffel-Anwendung. Die Diagramme zeigen die konzeptionellen Interaktionen ohne technische Details wie REST-Pfade, Payloads oder Datenbankabfragen.

### Service-Übersicht

| Service | Verantwortlichkeit |
|---------|-------------------|
| **Frontend** | React UI, Benutzerinteraktion |
| **API Gateway** | Routing, zentrale Authentifizierung |
| **Auth Service** | JWT-Verwaltung |
| **Lobby Service** | Lobby- und Spielerverwaltung |
| **Game Service** | Spiellogik und Punkteberechnung |
| **SSE Service** | Event-Broadcasting |
| **PostgreSQL** | Datenpersistenz |

---

## Flow 1: User-Onboarding & Lobby erstellen

### Szenario
Ein neuer User öffnet die App, gibt einen Username ein und erstellt eine neue Lobby.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Browser
    participant Frontend
    participant Gateway as API Gateway
    participant Auth as Auth Service
    participant Lobby as Lobby Service
    participant DB as PostgreSQL
    participant SSE as SSE Service

    Browser->>Frontend: App öffnen
    Frontend->>Browser: Startseite anzeigen
    
    Browser->>Frontend: Username eingeben + Lobby erstellen
    Frontend->>Gateway: Lobby-Erstellung anfordern
    
    Gateway->>Auth: JWT erstellen
    Auth->>Gateway: JWT zurückgeben
    
    Gateway->>Lobby: Lobby erstellen (User-Context)
    
    Lobby->>DB: User + Lobby + Spieler speichern
    DB->>Lobby: Bestätigung
    
    Lobby->>Lobby: Join-Code generieren
    
    Lobby->>SSE: Lobby registrieren
    SSE->>Lobby: Bestätigung
    
    Lobby->>Gateway: Lobby-Daten (ID, Join-Code, Leiter)
    Gateway->>Frontend: Lobby-Daten + JWT
    
    Frontend->>Browser: Lobby-Ansicht anzeigen
    
    Frontend->>SSE: SSE-Verbindung öffnen
    SSE-->>Frontend: Verbindung bestätigt
```

### Wichtige Datenflüsse
- Username → JWT-Erstellung
- Lobby-Daten → Persistierung
- Join-Code-Generierung
- SSE-Verbindung für Live-Updates

---

## Flow 2: Lobby beitreten

### Szenario
Ein zweiter User (Bob) möchte der Lobby von Alice beitreten. Er gibt seinen Username und den Join-Code ein.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Browser as Browser (Bob)
    participant Frontend as Frontend (Bob)
    participant Gateway as API Gateway
    participant Auth as Auth Service
    participant Lobby as Lobby Service
    participant DB as PostgreSQL
    participant SSE as SSE Service
    participant SSE_Alice as SSE (Alice)
    participant Frontend_Alice as Frontend (Alice)

    Browser->>Frontend: App öffnen
    Frontend->>Browser: Startseite anzeigen
    
    Browser->>Frontend: Username + Join-Code eingeben
    Frontend->>Gateway: Lobby beitreten anfordern
    
    Gateway->>Auth: JWT erstellen für Bob
    Auth->>Gateway: JWT zurückgeben
    
    Gateway->>Lobby: Lobby beitreten (User-Context)
    
    Lobby->>DB: Lobby finden
    DB->>Lobby: Lobby-Daten
    
    Note over Lobby: Validierung:<br/>Lobby existiert, Status OK,<br/>Nicht voll
    
    Lobby->>DB: Spieler hinzufügen
    DB->>Lobby: Bestätigung
    
    Lobby->>SSE: Event: Spieler beigetreten
    
    SSE->>SSE_Alice: Event weiterleiten
    SSE_Alice->>Frontend_Alice: Spielerliste aktualisieren
    
    SSE->>Lobby: Bestätigung
    
    Lobby->>Gateway: Lobby-Daten
    Gateway->>Frontend: Lobby-Daten + JWT
    
    Frontend->>Browser: Lobby-Ansicht anzeigen
    
    Frontend->>SSE: SSE-Verbindung öffnen
    SSE-->>Frontend: Verbindung bestätigt
```

### Wichtige Datenflüsse
- Join-Code → Lobby-Identifikation
- Spieler-Hinzufügen → Persistierung
- Event → Alle Lobby-Mitglieder
- Neue SSE-Verbindung

---

## Flow 3: Spiel starten

### Szenario
Alice (Lobby-Leiter) klickt auf "Spiel starten". Das System erstellt ein Game, legt die Zugreihenfolge fest und leitet alle Spieler zur Spielseite weiter.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Frontend_Alice as Frontend (Alice)
    participant Gateway as API Gateway
    participant Lobby as Lobby Service
    participant Game as Game Service
    participant DB_Lobby as PostgreSQL (lobby_db)
    participant DB_Game as PostgreSQL (game_db)
    participant SSE as SSE Service
    participant SSE_All as SSE (alle Spieler)
    participant Frontend_All as Frontend (alle Spieler)

    Frontend_Alice->>Gateway: Spiel starten
    
    Gateway->>Lobby: Spielstart anfordern (User-Context)
    
    Lobby->>DB_Lobby: Lobby + Spieler laden
    DB_Lobby->>Lobby: Daten zurückgeben
    
    Note over Lobby: Validierung:<br/>Leiter-Berechtigung,<br/>Mindestanzahl Spieler
    
    Lobby->>Lobby: Zugreihenfolge generieren
    
    Lobby->>Game: Game erstellen (Spieler-Liste)
    
    Game->>DB_Game: Game + Scores initialisieren
    DB_Game->>Game: Game-ID zurückgeben
    
    Game->>Game: Spielzustand initialisieren
    
    Game->>Lobby: Game-Daten (ID, aktueller Spieler)
    
    Lobby->>DB_Lobby: Lobby-Status aktualisieren (running)
    DB_Lobby->>Lobby: Bestätigung
    
    Lobby->>SSE: Event: Spiel gestartet
    
    SSE->>SSE_All: Event weiterleiten
    SSE_All->>Frontend_All: Zur Spielseite navigieren
    Frontend_All->>Frontend_All: Spielansicht laden
    
    SSE->>Lobby: Bestätigung
    Lobby->>Gateway: Game-Daten
    Gateway->>Frontend_Alice: Bestätigung
```

### Wichtige Datenflüsse
- Leiter-Berechtigung → Validierung
- Zufällige Zugreihenfolge → Generierung
- Game-Initialisierung → Persistierung
- Event → Alle Spieler gleichzeitig
- Lobby-Status-Änderung (waiting → running)

---

## Flow 4: Spieler kicken

### Szenario
Alice (Lobby-Leiter) kickt Bob aus der Lobby im "waiting"-Status. Bob wird benachrichtigt und zur Startseite weitergeleitet.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Frontend_Alice as Frontend (Alice)
    participant Gateway as API Gateway
    participant Lobby as Lobby Service
    participant DB as PostgreSQL (lobby_db)
    participant SSE as SSE Service
    participant SSE_Bob as SSE (Bob)
    participant Frontend_Bob as Frontend (Bob)
    participant SSE_Others as SSE (andere Spieler)
    participant Frontend_Others as Frontend (andere Spieler)

    Frontend_Alice->>Gateway: Spieler kicken anfordern
    
    Gateway->>Lobby: Kick-Befehl (User-Context + Target)
    
    Lobby->>DB: Lobby + Spieler laden
    DB->>Lobby: Daten zurückgeben
    
    Note over Lobby: Validierung:<br/>Leiter-Berechtigung,<br/>Status = waiting,<br/>Target in Lobby
    
    Lobby->>DB: Spieler entfernen
    DB->>Lobby: Bestätigung
    
    Lobby->>SSE: Event: Spieler gekickt
    
    SSE->>SSE_Bob: Event: Du wurdest gekickt
    SSE_Bob->>Frontend_Bob: Benachrichtigung anzeigen
    Frontend_Bob->>Frontend_Bob: Zur Startseite navigieren
    Frontend_Bob->>SSE_Bob: Verbindung schließen
    
    SSE->>SSE_Others: Event: Spieler verlassen
    SSE_Others->>Frontend_Others: Spielerliste aktualisieren
    
    SSE->>SSE: Bob's Connection entfernen
    SSE->>Lobby: Bestätigung
    
    Lobby->>Gateway: Bestätigung
    Gateway->>Frontend_Alice: Bestätigung
```

### Wichtige Datenflüsse
- Kick-Berechtigung → Validierung
- Spieler-Entfernung → Persistierung
- Unterschiedliche Events für Gekickten vs. Andere
- SSE-Verbindung schließen

---

## Flow 5: Spielzug komplett

### Szenario
Charlie ist am Zug. Er würfelt dreimal, fixiert Würfel zwischen den Würfen, wählt dann ein Feld aus. Danach ist Alice an der Reihe.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Frontend_Charlie as Frontend (Charlie)
    participant Gateway as API Gateway
    participant Game as Game Service
    participant DB as PostgreSQL (game_db)
    participant SSE as SSE Service
    participant SSE_All as SSE (alle)
    participant Frontend_All as Frontend (alle)

    Note over Frontend_Charlie: Charlie am Zug, Wurf 0

    %% Erster Wurf
    Frontend_Charlie->>Gateway: Würfeln
    Gateway->>Game: Würfel-Befehl (User-Context)
    
    Game->>DB: Spielzustand laden
    DB->>Game: Zustand zurückgeben
    
    Note over Game: Validierung:<br/>Richtiger Spieler,<br/>Wurf-Limit nicht erreicht
    
    Game->>Game: Würfel generieren
    
    Game->>DB: Würfel + Wurf-Count speichern
    DB->>Game: Bestätigung
    
    Game->>SSE: Event: Würfel geworfen
    SSE->>SSE_All: Event weiterleiten
    SSE_All->>Frontend_All: Würfel anzeigen (Animation)
    
    SSE->>Game: Bestätigung
    Game->>Gateway: Würfel-Daten
    Gateway->>Frontend_Charlie: Bestätigung

    %% Würfel fixieren
    Frontend_Charlie->>Gateway: Würfel fixieren
    Gateway->>Game: Fixier-Befehl (Würfel-Indices)
    
    Game->>DB: Spielzustand laden
    DB->>Game: Zustand zurückgeben
    
    Game->>Game: Fixierungen aktualisieren
    
    Game->>DB: Fixierungen speichern
    DB->>Game: Bestätigung
    
    Game->>SSE: Event: Würfel fixiert
    SSE->>SSE_All: Event weiterleiten
    SSE_All->>Frontend_All: Fixierung anzeigen
    
    SSE->>Game: Bestätigung
    Game->>Gateway: Bestätigung
    Gateway->>Frontend_Charlie: Bestätigung

    %% Zweiter Wurf
    Frontend_Charlie->>Gateway: Nochmal würfeln
    Gateway->>Game: Würfel-Befehl
    
    Game->>DB: Spielzustand laden
    DB->>Game: Zustand zurückgeben
    
    Game->>Game: Nicht-fixierte Würfel neu würfeln
    
    Game->>DB: Würfel + Wurf-Count speichern
    DB->>Game: Bestätigung
    
    Game->>SSE: Event: Würfel geworfen
    SSE->>SSE_All: Event weiterleiten
    SSE_All->>Frontend_All: Würfel aktualisieren
    
    SSE->>Game: Bestätigung
    Game->>Gateway: Würfel-Daten
    Gateway->>Frontend_Charlie: Bestätigung

    %% Feld wählen
    Frontend_Charlie->>Gateway: Feld auswählen
    Gateway->>Game: Feld-Auswahl (User-Context, Feld)
    
    Game->>DB: Spielzustand + Scores laden
    DB->>Game: Daten zurückgeben
    
    Note over Game: Validierung:<br/>Richtiger Spieler,<br/>Feld noch frei
    
    Game->>Game: Punkte berechnen
    
    Game->>DB: Punkte speichern
    DB->>Game: Bestätigung
    
    Game->>Game: Nächsten Spieler bestimmen
    
    Game->>DB: Spielzustand zurücksetzen
    DB->>Game: Bestätigung
    
    Game->>SSE: Event: Feld gewählt + Zugwechsel
    SSE->>SSE_All: Events weiterleiten
    SSE_All->>Frontend_All: Block aktualisieren + Spieler wechseln
    
    SSE->>Game: Bestätigung
    Game->>Gateway: Bestätigung
    Gateway->>Frontend_Charlie: Bestätigung

    Note over Frontend_All: Alice ist jetzt am Zug
```

### Wichtige Datenflüsse
- Würfeln: Zufallswerte → Persistierung → Broadcasting
- Fixieren: Würfel-Auswahl → Persistierung → Broadcasting
- Punkteberechnung: Würfel + Feld → Punkte
- Zugwechsel: Spieler-Index → Nächster Spieler
- Timer-Reset bei jeder Interaktion

---

## Flow 6: Timeout-Handling

### Szenario
Alice ist am Zug, interagiert aber 40 Sekunden lang nicht. Der Game Service erkennt das Timeout und überspringt Alice's Zug.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Timer as Timer (Game Service)
    participant Game as Game Service
    participant DB as PostgreSQL (game_db)
    participant SSE as SSE Service
    participant SSE_All as SSE (alle)
    participant Frontend_All as Frontend (alle)
    participant Frontend_Alice as Frontend (Alice)

    Note over Game: Alice am Zug,<br/>keine Interaktion

    Timer->>Game: Periodischer Check
    
    Game->>DB: Laufende Spiele prüfen
    DB->>Game: Spielzustände zurückgeben
    
    Game->>Game: Zeitdifferenz berechnen
    
    Note over Game: Timeout erkannt:<br/>Mehr als 40 Sekunden
    
    Game->>DB: Spieler-Status: inactive
    DB->>Game: Bestätigung
    
    Game->>Game: Nächsten aktiven Spieler finden
    
    Game->>DB: Spielzustand zurücksetzen
    DB->>Game: Bestätigung
    
    Game->>SSE: Events: Timeout + Status + Zugwechsel
    
    SSE->>SSE_All: Events weiterleiten
    SSE_All->>Frontend_All: Benachrichtigung + Inaktiv-Markierung
    
    SSE_All->>Frontend_Alice: Spezielle Warnung anzeigen
    
    SSE->>Game: Bestätigung

    Note over Frontend_All: Nächster Spieler am Zug

    %% Reaktivierung
    Note over Frontend_Alice: Alice kehrt zurück

    Frontend_Alice->>Gateway: Reaktivierung anfordern
    Gateway->>Game: Reaktivierungs-Befehl
    
    Game->>DB: Spieler-Status: active
    DB->>Game: Bestätigung
    
    Game->>SSE: Event: Spieler wieder aktiv
    SSE->>SSE_All: Event weiterleiten
    SSE_All->>Frontend_All: Inaktiv-Markierung entfernen
    
    SSE->>Game: Bestätigung
    Game->>Gateway: Bestätigung
    Gateway->>Frontend_Alice: Bestätigung
```

### Wichtige Datenflüsse
- Periodischer Timer → Timeout-Erkennung
- Spieler-Status: active ↔ inactive
- Automatisches Überspringen inaktiver Spieler
- Manuelle Reaktivierung möglich

---

## Flow 7: Spieler Disconnect/Reconnect

### Szenario
Bob verliert die Internetverbindung. Seine SSE-Verbindung bricht ab. Das Spiel läuft weiter. Bob stellt die Verbindung wieder her und wird automatisch zurück ins Spiel gebracht.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Browser_Bob as Browser (Bob)
    participant Frontend_Bob as Frontend (Bob)
    participant SSE_Bob as SSE (Bob)
    participant SSE as SSE Service
    participant Gateway as API Gateway
    participant Game as Game Service
    participant DB as PostgreSQL (game_db)
    participant SSE_Others as SSE (andere)
    participant Frontend_Others as Frontend (andere)

    Note over Browser_Bob: Verbindung unterbrochen

    %% Disconnect
    Browser_Bob->>Frontend_Bob: Verbindung verloren
    Frontend_Bob->>SSE_Bob: Verbindung bricht ab
    
    SSE_Bob->>SSE: Connection-Close erkannt
    SSE->>SSE: Bob's Connection entfernen
    
    SSE->>Game: Spieler disconnected
    Game->>DB: Spieler-Status: disconnected
    DB->>Game: Bestätigung
    
    Game->>SSE: Event: Spieler disconnected
    SSE->>SSE_Others: Event weiterleiten
    SSE_Others->>Frontend_Others: Disconnect-Markierung
    
    SSE->>Game: Bestätigung

    Note over Game: Spiel läuft weiter,<br/>Bob wird übersprungen

    %% Reconnect
    Note over Browser_Bob: Verbindung wiederhergestellt

    Browser_Bob->>Frontend_Bob: App neu laden
    Frontend_Bob->>Frontend_Bob: Session prüfen (JWT + game_id)
    
    Frontend_Bob->>Gateway: Spielzustand abrufen
    Gateway->>Game: State-Anfrage (User-Context)
    
    Game->>DB: Kompletten Spielstand laden
    DB->>Game: Spielstand zurückgeben
    
    Game->>Gateway: Spielstand
    Gateway->>Frontend_Bob: Spielstand
    
    Frontend_Bob->>Frontend_Bob: Spielansicht rendern
    
    Frontend_Bob->>SSE: SSE-Verbindung neu öffnen
    SSE->>SSE: Bob's Connection registrieren
    SSE-->>Frontend_Bob: Verbindung bestätigt
    
    SSE->>Game: Spieler reconnected
    Game->>DB: Spieler-Status: active
    DB->>Game: Bestätigung
    
    Game->>SSE: Event: Spieler reconnected
    SSE->>SSE_Others: Event weiterleiten
    SSE_Others->>Frontend_Others: Disconnect-Markierung entfernen
    
    SSE->>SSE_Bob: Event weiterleiten
    SSE_Bob->>Frontend_Bob: Bestätigung anzeigen
    
    SSE->>Game: Bestätigung

    Note over Frontend_Bob: Bob wieder vollständig im Spiel
```

### Wichtige Datenflüsse
- Connection-Loss → Status-Update (disconnected)
- Spieler wird automatisch übersprungen
- Reconnect → Vollständiger Spielstand-Abruf
- Neue SSE-Verbindung aufbauen
- Status-Update (active)

---

## Flow 8: Spielende erkennen & Rangliste

### Szenario
Alice füllt ihr letztes (13.) Feld aus. Der Game Service erkennt, dass alle Spieler fertig sind, berechnet die Endpunkte und erstellt eine Rangliste.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Frontend_Alice as Frontend (Alice)
    participant Gateway as API Gateway
    participant Game as Game Service
    participant DB as PostgreSQL (game_db)
    participant SSE as SSE Service
    participant SSE_All as SSE (alle)
    participant Frontend_All as Frontend (alle)

    Note over Frontend_Alice: Alice füllt letztes Feld

    Frontend_Alice->>Gateway: Feld auswählen
    Gateway->>Game: Feld-Auswahl (User-Context)
    
    Game->>DB: Spielzustand + Scores laden
    DB->>Game: Daten zurückgeben
    
    Game->>Game: Punkte berechnen
    
    Game->>DB: Punkte speichern
    DB->>Game: Bestätigung
    
    Note over Game: Prüfung: Alle fertig?
    
    Game->>DB: Alle Spieler fertig prüfen
    DB->>Game: Alle fertig!
    
    Game->>DB: Alle Scores laden
    DB->>Game: Score-Daten
    
    Game->>Game: Endpunkte berechnen:<br/>- Oberer Teil + Bonus<br/>- Unterer Teil<br/>- Gesamtsumme
    
    Game->>Game: Rangliste erstellen (sortiert)
    
    Game->>DB: Game-Status: finished<br/>Rangliste speichern
    DB->>Game: Bestätigung
    
    Game->>SSE: Events: Feld gewählt + Spielende
    SSE->>SSE_All: Events weiterleiten
    SSE_All->>Frontend_All: Block aktualisieren
    
    Note over Frontend_All: Kurze Pause (2s)
    
    Frontend_All->>Frontend_All: Zur Endeseite navigieren
    Frontend_All->>Frontend_All: Rangliste anzeigen
    
    SSE->>Game: Bestätigung
    Game->>Gateway: Bestätigung
    Gateway->>Frontend_Alice: Bestätigung

    Note over Frontend_All: Endeseite mit Rangliste
```

### Wichtige Datenflüsse
- Letzte Feldwahl → Punkteberechnung
- Prüfung: Alle Spieler fertig?
- Endpunkte-Berechnung (Bonus-Logik)
- Rangliste erstellen (sortiert)
- Game-Status: running → finished
- Navigation aller Spieler zur Endeseite

---

## Flow 9: Nochmal spielen / Neue Runde

### Szenario
Die Spieler sind auf der Endeseite. Alice (Lobby-Leiter) klickt auf "Nochmal spielen". Eine neue Lobby wird mit denselben Spielern erstellt.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Frontend_Alice as Frontend (Alice)
    participant Gateway as API Gateway
    participant Lobby as Lobby Service
    participant Game as Game Service
    participant DB_Lobby as PostgreSQL (lobby_db)
    participant DB_Game as PostgreSQL (game_db)
    participant SSE as SSE Service
    participant SSE_All as SSE (alle)
    participant Frontend_All as Frontend (alle)

    Note over Frontend_All: Alle auf Endeseite

    Frontend_Alice->>Gateway: Rematch anfordern
    Gateway->>Lobby: Rematch-Befehl (User-Context)
    
    Lobby->>DB_Lobby: Alte Lobby + Spieler laden
    DB_Lobby->>Lobby: Daten zurückgeben
    
    Note over Lobby: Validierung:<br/>Leiter-Berechtigung,<br/>Spiel beendet
    
    Lobby->>Lobby: Neue Lobby erstellen:<br/>- Neuer Join-Code<br/>- Status: waiting<br/>- Gleicher Leiter
    
    Lobby->>DB_Lobby: Neue Lobby + Spieler speichern
    DB_Lobby->>Lobby: Neue Lobby-ID
    
    Lobby->>Game: Altes Game archivieren
    Game->>DB_Game: Game archivieren + Referenz speichern
    DB_Game->>Game: Bestätigung
    Game->>Lobby: Bestätigung
    
    Lobby->>SSE: Event: Rematch erstellt
    
    SSE->>SSE: Connections für altes Game finden
    SSE->>SSE_All: Event weiterleiten
    SSE_All->>Frontend_All: Benachrichtigung anzeigen
    
    Frontend_All->>Frontend_All: Zur neuen Lobby navigieren
    Frontend_All->>Frontend_All: Lobby-Ansicht anzeigen
    
    SSE->>Lobby: Bestätigung
    Lobby->>Gateway: Neue Lobby-Daten
    Gateway->>Frontend_Alice: Bestätigung

    Note over Frontend_All: Alle in neuer Lobby

    Frontend_All->>SSE_All: Alte SSE-Verbindung schließen
    Frontend_All->>SSE: Neue SSE-Verbindung öffnen
    SSE->>SSE: Neue Connections registrieren
    SSE-->>Frontend_All: Verbindung bestätigt

    Note over Frontend_All: Bereit für nächstes Spiel
```

### Wichtige Datenflüsse
- Rematch-Berechtigung → Validierung (nur original Leiter)
- Neue Lobby-Erstellung (neuer Join-Code)
- Spieler-Übernahme (automatisch)
- Altes Game → Archivierung
- Event → Alle Spieler
- SSE-Verbindungen wechseln (alt → neu)

---

## Flow 10: SSE-Connection-Management & Event-Broadcasting

### Szenario
Technische Infrastruktur des SSE Service. Zeigt den kompletten Lifecycle: Connection aufbauen, Events empfangen, Broadcasting, Connection schließen.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Frontend as Frontend (Client)
    participant SSE as SSE Service
    participant Auth as Auth Service
    participant Service as Service (Lobby/Game)
    participant Connections as In-Memory Map

    %% Connection aufbauen
    Note over Frontend: User in Lobby/Game

    Frontend->>SSE: SSE-Verbindung öffnen (JWT)
    
    SSE->>Auth: JWT validieren
    Auth->>SSE: User-Context zurückgeben
    
    Note over SSE: Connection-Objekt erstellen
    
    SSE->>Connections: Connection registrieren
    Connections->>SSE: Bestätigung
    
    SSE-->>Frontend: Verbindung bestätigt
    
    Note over Frontend,SSE: Stream bleibt offen

    %% Event Broadcasting
    Note over Service: Event tritt ein

    Service->>SSE: Event veröffentlichen (Lobby/Game-ID)
    
    SSE->>Connections: Alle Connections für ID abrufen
    Connections->>SSE: Connection-Liste
    
    Note over SSE: Broadcasting an alle

    loop Für jede Connection
        SSE->>Frontend: Event senden
        
        alt Senden erfolgreich
            Frontend->>Frontend: Event verarbeiten
        else Senden fehlgeschlagen
            SSE->>SSE: Connection als "broken" markieren
            SSE->>Connections: Connection entfernen
        end
    end
    
    SSE->>Service: Broadcasting-Bestätigung

    %% Keep-Alive
    Note over SSE: Periodisch (30s)

    SSE->>Frontend: Keep-Alive senden
    Note over SSE,Frontend: Verhindert Timeout

    %% Connection schließen (Client)
    Note over Frontend: User navigiert weg

    Frontend->>Frontend: Verbindung schließen
    Frontend->>SSE: Connection-Close
    
    SSE->>Connections: Connection entfernen
    SSE->>SSE: Ressourcen freigeben

    %% Connection schließen (Server)
    Note over Service: Alternative: Server-Close

    Service->>SSE: Lobby/Game schließen
    SSE->>Connections: Alle Connections abrufen
    
    loop Für jede Connection
        SSE->>Frontend: Close-Event senden
        SSE->>Frontend: Verbindung schließen
    end
    
    SSE->>Connections: Lobby/Game aus Map löschen
    SSE->>Service: Bestätigung

    %% Cleanup
    Note over SSE: Background-Task (5 Min)

    SSE->>Connections: Alle Lobbies/Games prüfen
    
    loop Für jede Lobby/Game
        alt Connection tot
            SSE->>Connections: Zombie entfernen
        end
    end
```

### Wichtige Datenflüsse
- JWT → User-Authentifizierung
- Connection-Registrierung in Map (lobby_id/game_id → Connections)
- Event → Alle Connections parallel
- Keep-Alive → Verbindung aktiv halten
- Connection-Cleanup (Client/Server-initiiert)
- Periodischer Zombie-Cleanup

## Zusammenfassung

### Phasen-Übersicht

**Phase 1: Lobby-Management**
- Flow 1: User-Onboarding & Lobby erstellen
- Flow 2: Lobby beitreten
- Flow 3: Spiel starten
- Flow 4: Spieler kicken

**Phase 2: Spiel-Durchführung**
- Flow 5: Spielzug komplett
- Flow 6: Timeout-Handling
- Flow 7: Spieler Disconnect/Reconnect

**Phase 3: Spielende**
- Flow 8: Spielende erkennen & Rangliste
- Flow 9: Nochmal spielen / Neue Runde

**Querschnitt**
- Flow 10: SSE-Connection-Management & Event-Broadcasting

### Zentrale Datenfluss-Patterns

| Pattern | Verwendung |
|---------|------------|
| **Request → Validate → Persist → Broadcast** | Würfeln, Feldwahl, Beitreten |
| **Periodischer Check → Status-Update → Event** | Timeout-Handling |
| **Connection-Loss → Status-Update → Reconnect** | Disconnect/Reconnect |
| **Check-Complete → Calculate → Persist → Notify** | Spielende |
| **Event → Lookup-Connections → Parallel-Send** | SSE-Broadcasting |
