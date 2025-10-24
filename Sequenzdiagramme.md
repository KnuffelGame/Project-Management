# Sequenzdiagramme

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

    Browser->>Frontend: Öffnet App
    Frontend->>Browser: Zeigt Startseite (Username-Eingabe)
    
    Browser->>Frontend: Gibt Username ein + klickt "Lobby erstellen"
    
    Frontend->>Gateway: POST /api/lobby/create<br/>{username: "Alice"}
    
    Note over Gateway: Routing + Validierung
    
    Gateway->>Auth: POST /internal/create<br/>{username: "Alice"}
    Auth->>Auth: JWT generieren
    Auth->>Gateway: {token: "eyJhbG..."}
    
    Note over Gateway: Setzt JWT als Cookie
    
    Gateway->>Lobby: POST /internal/lobby/create<br/>Header: X-Username: Alice<br/>Header: X-User-ID: user-123
    
    Lobby->>DB: INSERT INTO users<br/>INSERT INTO lobbies<br/>INSERT INTO players
    DB->>Lobby: OK
    
    Lobby->>Lobby: Join-Code generieren (z.B. "ABC123")
    
    Lobby->>SSE: POST /internal/broadcast<br/>{event: "lobby_created", lobby_id, join_code}
    SSE->>SSE: Registriert Lobby (noch keine Connections)
    SSE->>Lobby: OK
    
    Lobby->>Gateway: {lobby_id, join_code: "ABC123", leader: "Alice"}
    Gateway->>Frontend: Set-Cookie: jwt=...<br/>{lobby_id, join_code: "ABC123"}
    
    Frontend->>Browser: Zeigt Lobby-Ansicht<br/>Join-Code: ABC123
    
    Frontend->>SSE: GET /events/lobby/{lobby_id}<br/>Cookie: jwt=...
    Note over SSE: Validiert JWT,<br/>registriert Connection
    SSE-->>Frontend: SSE Stream opened
    
    Note over Frontend,SSE: Verbindung bleibt offen<br/>für Live-Updates
```

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
    participant SSE_Alice as SSE Connection (Alice)
    participant Frontend_Alice as Frontend (Alice)

    Browser->>Frontend: Öffnet App
    Frontend->>Browser: Zeigt Startseite
    
    Browser->>Frontend: Gibt Username "Bob" + Join-Code "ABC123" ein
    
    Frontend->>Gateway: POST /api/lobby/join<br/>{username: "Bob", join_code: "ABC123"}
    
    Gateway->>Auth: POST /internal/create<br/>{username: "Bob"}
    Auth->>Auth: JWT generieren für Bob
    Auth->>Gateway: {token: "eyJhbG..."}
    
    Note over Gateway: Setzt JWT als Cookie
    
    Gateway->>Lobby: POST /internal/lobby/join<br/>{join_code: "ABC123"}<br/>Header: X-User-ID: user-456<br/>Header: X-Username: Bob
    
    Lobby->>DB: SELECT lobby WHERE join_code = "ABC123"
    DB->>Lobby: {lobby_id, status: "waiting", player_count: 1}
    
    Note over Lobby: Validiert:<br/>- Lobby existiert<br/>- Status = "waiting"<br/>- Nicht voll (< 6 Spieler)
    
    Lobby->>DB: INSERT INTO players<br/>(lobby_id, user_id, username: "Bob")
    DB->>Lobby: OK
    
    Lobby->>SSE: POST /internal/broadcast<br/>{event: "player_joined",<br/>lobby_id, username: "Bob"}
    
    SSE->>SSE: Findet alle Connections für lobby_id
    SSE->>SSE_Alice: event: player_joined<br/>data: {username: "Bob", player_count: 2}
    SSE_Alice->>Frontend_Alice: Event empfangen
    Frontend_Alice->>Frontend_Alice: Aktualisiert Spielerliste
    
    SSE->>Lobby: OK (Broadcast sent)
    
    Lobby->>Gateway: {lobby_id, players: ["Alice", "Bob"],<br/>leader: "Alice"}
    Gateway->>Frontend: Set-Cookie: jwt=...<br/>{lobby_id, players: [...]}
    
    Frontend->>Browser: Zeigt Lobby-Ansicht<br/>Spieler: Alice (👑), Bob
    
    Frontend->>SSE: GET /events/lobby/{lobby_id}<br/>Cookie: jwt=...
    Note over SSE: Validiert JWT,<br/>registriert Bob's Connection
    SSE-->>Frontend: SSE Stream opened
    
    Note over Frontend,SSE: Bob empfängt nun auch<br/>alle zukünftigen Events
```

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
    participant SSE_Connections as SSE Connections (alle Spieler)
    participant Frontend_All as Frontend (alle Spieler)

    Frontend_Alice->>Gateway: POST /api/lobby/{lobby_id}/start<br/>Cookie: jwt=...
    
    Gateway->>Gateway: Validiert JWT
    Note over Gateway: Extrahiert User-ID von Alice
    
    Gateway->>Lobby: POST /internal/lobby/{lobby_id}/start<br/>Header: X-User-ID: alice-123
    
    Lobby->>DB_Lobby: SELECT lobby, players WHERE lobby_id
    DB_Lobby->>Lobby: {lobby, players: ["Alice", "Bob", "Charlie"]}
    
    Note over Lobby: Validiert:<br/>- Alice ist Lobby-Leiter<br/>- Status = "waiting"<br/>- Min 2 Spieler
    
    Lobby->>Lobby: Generiert zufällige Zugreihenfolge<br/>z.B. [Charlie, Alice, Bob]
    
    Lobby->>Game: POST /internal/game/create<br/>{lobby_id, players: [<br/>{user_id, username, order: 1},<br/>{user_id, username, order: 2}, ...]}
    
    Game->>DB_Game: INSERT INTO games<br/>(lobby_id, status: "running",<br/>current_player_index: 0)
    Game->>DB_Game: INSERT INTO scores<br/>(game_id, user_id, fields: {})
    DB_Game->>Game: {game_id}
    
    Game->>Game: Initialisiert Spielzustand:<br/>- Würfel: [0,0,0,0,0]<br/>- Fixierungen: [false, false, ...]<br/>- Wurf-Count: 0<br/>- Aktueller Spieler: Charlie
    
    Game->>Lobby: {game_id, current_player: "Charlie"}
    
    Lobby->>DB_Lobby: UPDATE lobbies<br/>SET status = "running", game_id
    DB_Lobby->>Lobby: OK
    
    Lobby->>SSE: POST /internal/broadcast<br/>{event: "game_started",<br/>lobby_id, game_id,<br/>turn_order: [...],<br/>current_player: "Charlie"}
    
    SSE->>SSE: Findet alle Connections für lobby_id
    SSE->>SSE_Connections: event: game_started<br/>data: {game_id, turn_order, current_player}
    
    SSE_Connections->>Frontend_All: Event empfangen
    
    Frontend_All->>Frontend_All: Navigiert zu /game/{game_id}
    Frontend_All->>Frontend_All: Lädt initiale Spieldaten<br/>Zeigt Kniffel-Block + Würfel<br/>Hebt aktuellen Spieler hervor
    
    SSE->>Lobby: OK
    
    Lobby->>Gateway: {game_id, status: "running"}
    Gateway->>Frontend_Alice: {game_id}
    
    Note over Frontend_All: Alle Spieler sind jetzt<br/>auf der Spielseite.<br/>Charlie ist am Zug.
```

## Flow 4: Spieler kicken

### Szenario
Alice (Lobby-Leiter) kickt Bob aus der Lobby, während sich noch im "waiting"-Status befinden. Bob wird benachrichtigt und zur Startseite weitergeleitet.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Frontend_Alice as Frontend (Alice)
    participant Gateway as API Gateway
    participant Lobby as Lobby Service
    participant DB as PostgreSQL (lobby_db)
    participant SSE as SSE Service
    participant SSE_Bob as SSE Connection (Bob)
    participant Frontend_Bob as Frontend (Bob)
    participant SSE_Others as SSE Connections (andere Spieler)
    participant Frontend_Others as Frontend (andere Spieler)

    Frontend_Alice->>Gateway: POST /api/lobby/{lobby_id}/kick<br/>{user_id: "bob-456"}<br/>Cookie: jwt=...
    
    Gateway->>Gateway: Validiert JWT
    Note over Gateway: Extrahiert Alice's User-ID
    
    Gateway->>Lobby: POST /internal/lobby/{lobby_id}/kick<br/>{target_user_id: "bob-456"}<br/>Header: X-User-ID: alice-123
    
    Lobby->>DB: SELECT lobby, players WHERE lobby_id
    DB->>Lobby: {lobby, leader: alice-123, status: "waiting",<br/>players: [...]}
    
    Note over Lobby: Validiert:<br/>- Alice ist Lobby-Leiter<br/>- Status = "waiting"<br/>- Bob ist in der Lobby<br/>- Bob ist nicht der Leiter
    
    Lobby->>DB: DELETE FROM players<br/>WHERE lobby_id AND user_id = "bob-456"
    DB->>Lobby: OK
    
    Lobby->>SSE: POST /internal/broadcast<br/>{event: "player_kicked",<br/>lobby_id, kicked_user_id: "bob-456",<br/>username: "Bob"}
    
    SSE->>SSE: Findet alle Connections für lobby_id
    
    SSE->>SSE_Bob: event: player_kicked<br/>data: {kicked_user_id: "bob-456",<br/>reason: "kicked_by_leader"}
    SSE_Bob->>Frontend_Bob: Event empfangen
    Frontend_Bob->>Frontend_Bob: Zeigt Notification:<br/>"Du wurdest aus der Lobby entfernt"
    Frontend_Bob->>Frontend_Bob: Navigiert zu Startseite
    Frontend_Bob->>SSE_Bob: Schließt SSE-Verbindung
    
    SSE->>SSE_Others: event: player_left<br/>data: {username: "Bob",<br/>player_count: 2}
    SSE_Others->>Frontend_Others: Event empfangen
    Frontend_Others->>Frontend_Others: Aktualisiert Spielerliste<br/>(Bob entfernt)
    
    SSE->>SSE: Entfernt Bob's Connection aus Map
    
    SSE->>Lobby: OK
    
    Lobby->>Gateway: {status: "ok", remaining_players: 2}
    Gateway->>Frontend_Alice: {status: "ok"}
    
    Frontend_Alice->>Frontend_Alice: Aktualisiert Spielerliste
    
    Note over Frontend_Bob: Bob ist jetzt auf der Startseite<br/>und kann neue Lobby erstellen/joinen
```

## Flow 5: Spielzug komplett

### Szenario
Charlie ist am Zug. Er würfelt dreimal, fixiert Würfel zwischen den Würfen, wählt dann ein Feld aus. Danach ist Alice an der Reihe. Alle Spieler sehen die Updates in Echtzeit.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Frontend_Charlie as Frontend (Charlie)
    participant Gateway as API Gateway
    participant Game as Game Service
    participant DB as PostgreSQL (game_db)
    participant SSE as SSE Service
    participant SSE_All as SSE Connections (alle)
    participant Frontend_All as Frontend (alle)

    Note over Frontend_Charlie: Charlie ist am Zug<br/>Wurf-Count: 0

    %% Erster Wurf
    Frontend_Charlie->>Gateway: POST /api/game/{game_id}/roll<br/>Cookie: jwt=...
    
    Gateway->>Game: POST /internal/game/{game_id}/roll<br/>Header: X-User-ID: charlie-789
    
    Game->>DB: SELECT game_state WHERE game_id
    DB->>Game: {current_player: charlie-789,<br/>roll_count: 0, dice: [0,0,0,0,0]}
    
    Note over Game: Validiert:<br/>- Charlie ist aktueller Spieler<br/>- roll_count < 3
    
    Game->>Game: Würfelt nicht-fixierte Würfel<br/>Ergebnis: [3,3,5,2,1]
    
    Game->>DB: UPDATE game_state<br/>SET dice=[3,3,5,2,1],<br/>roll_count=1,<br/>timer_reset=NOW()
    DB->>Game: OK
    
    Game->>SSE: POST /internal/broadcast<br/>{event: "dice_rolled",<br/>game_id, player: "Charlie",<br/>dice: [3,3,5,2,1],<br/>roll_count: 1}
    
    SSE->>SSE_All: event: dice_rolled<br/>data: {...}
    SSE_All->>Frontend_All: Event empfangen
    Frontend_All->>Frontend_All: Zeigt Würfel-Animation<br/>dann Ergebnis [3,3,5,2,1]
    
    SSE->>Game: OK
    Game->>Gateway: {dice: [3,3,5,2,1], roll_count: 1}
    Gateway->>Frontend_Charlie: {dice: [...]}

    Note over Frontend_Charlie: Charlie sieht Würfel:<br/>[3,3,5,2,1]

    %% Würfel fixieren
    Frontend_Charlie->>Gateway: POST /api/game/{game_id}/toggle-dice<br/>{dice_indices: [0,1]}<br/>Cookie: jwt=...
    
    Gateway->>Game: POST /internal/game/{game_id}/toggle-dice<br/>{dice_indices: [0,1]}<br/>Header: X-User-ID: charlie-789
    
    Game->>DB: SELECT game_state
    DB->>Game: {current_player: charlie-789, ...}
    
    Note over Game: Validiert:<br/>- Charlie ist aktueller Spieler<br/>- roll_count > 0 und < 3
    
    Game->>Game: Togglet Fixierung für Index 0,1<br/>locked: [true, true, false, false, false]
    
    Game->>DB: UPDATE game_state<br/>SET locked=[true,true,false,false,false],<br/>timer_reset=NOW()
    DB->>Game: OK
    
    Game->>SSE: POST /internal/broadcast<br/>{event: "dice_locked",<br/>game_id, dice_indices: [0,1],<br/>locked: [true,true,false,false,false]}
    
    SSE->>SSE_All: event: dice_locked<br/>data: {...}
    SSE_All->>Frontend_All: Event empfangen
    Frontend_All->>Frontend_All: Zeigt Würfel 0,1 als fixiert<br/>(visueller Rahmen)
    
    SSE->>Game: OK
    Game->>Gateway: {locked: [...]}
    Gateway->>Frontend_Charlie: OK

    %% Zweiter Wurf
    Frontend_Charlie->>Gateway: POST /api/game/{game_id}/roll
    
    Gateway->>Game: POST /internal/game/{game_id}/roll<br/>Header: X-User-ID: charlie-789
    
    Game->>DB: SELECT game_state
    DB->>Game: {dice: [3,3,5,2,1],<br/>locked: [true,true,false,false,false],<br/>roll_count: 1}
    
    Game->>Game: Würfelt nur nicht-fixierte (Index 2,3,4)<br/>Neue Würfel: [3,3,6,6,3]
    
    Game->>DB: UPDATE game_state<br/>SET dice=[3,3,6,6,3],<br/>roll_count=2,<br/>timer_reset=NOW()
    DB->>Game: OK
    
    Game->>SSE: POST /internal/broadcast<br/>{event: "dice_rolled",<br/>dice: [3,3,6,6,3],<br/>roll_count: 2}
    
    SSE->>SSE_All: event: dice_rolled
    SSE_All->>Frontend_All: Aktualisiert Würfel
    
    SSE->>Game: OK
    Game->>Gateway: {dice: [3,3,6,6,3], roll_count: 2}
    Gateway->>Frontend_Charlie: OK

    Note over Frontend_Charlie: Charlie entscheidet sich,<br/>kein 3. Mal zu würfeln

    %% Feld wählen
    Frontend_Charlie->>Gateway: POST /api/game/{game_id}/select-field<br/>{field: "dreier"}<br/>Cookie: jwt=...
    
    Gateway->>Game: POST /internal/game/{game_id}/select-field<br/>{field: "dreier"}<br/>Header: X-User-ID: charlie-789
    
    Game->>DB: SELECT game_state, scores
    DB->>Game: {current_player: charlie-789,<br/>dice: [3,3,6,6,3],<br/>charlie_scores: {...}}
    
    Note over Game: Validiert:<br/>- Charlie ist aktueller Spieler<br/>- Feld "dreier" noch leer
    
    Game->>Game: Berechnet Punkte für "dreier"<br/>Würfel [3,3,6,6,3] → 9 Punkte
    
    Game->>DB: UPDATE scores<br/>SET dreier=9<br/>WHERE game_id AND user_id=charlie-789
    DB->>Game: OK
    
    Game->>Game: Nächster Spieler bestimmen<br/>current_player_index: 0 → 1<br/>Nächster: Alice
    
    Game->>DB: UPDATE game_state<br/>SET current_player=alice-123,<br/>current_player_index=1,<br/>dice=[0,0,0,0,0],<br/>locked=[false,false,false,false,false],<br/>roll_count=0,<br/>timer_reset=NOW()
    DB->>Game: OK
    
    Game->>SSE: POST /internal/broadcast<br/>{event: "field_selected",<br/>player: "Charlie",<br/>field: "dreier",<br/>points: 9}<br/><br/>{event: "turn_changed",<br/>current_player: "Alice",<br/>next_player_index: 1}
    
    SSE->>SSE_All: Zwei Events senden
    SSE_All->>Frontend_All: Events empfangen
    
    Frontend_All->>Frontend_All: 1. Aktualisiert Kniffel-Block<br/>(Charlie: Dreier = 9 Punkte)
    Frontend_All->>Frontend_All: 2. Hebt Alice als aktiven Spieler hervor<br/>Resettet Würfel-Anzeige
    
    SSE->>Game: OK
    Game->>Gateway: {status: "ok",<br/>next_player: "Alice"}
    Gateway->>Frontend_Charlie: OK

    Note over Frontend_All: Alice ist jetzt am Zug<br/>Wurf-Count: 0<br/>Würfel: [0,0,0,0,0]
```

## Flow 6: Timeout-Handling

### Szenario
Alice ist am Zug, interagiert aber 40 Sekunden lang nicht. Der Game Service erkennt das Timeout, überspringt Alice's Zug, markiert sie als "inaktiv" und gibt den Zug an Bob weiter.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Timer as Timer (Game Service)
    participant Game as Game Service
    participant DB as PostgreSQL (game_db)
    participant SSE as SSE Service
    participant SSE_All as SSE Connections (alle)
    participant Frontend_All as Frontend (alle)
    participant Frontend_Alice as Frontend (Alice)

    Note over Game: Alice ist am Zug<br/>Letzter timer_reset:<br/>10:00:00

    Timer->>Game: Check-Zyklus (alle 5 Sekunden)
    
    Game->>DB: SELECT game_state<br/>WHERE status='running'
    DB->>Game: {game_id, current_player: alice-123,<br/>timer_reset: "10:00:00",<br/>player_status: {alice: "active"}}
    
    Game->>Game: Berechnet Zeitdifferenz<br/>NOW() - timer_reset = 45 Sekunden
    
    Note over Game: Timeout erkannt!<br/>(> 40 Sekunden ohne Interaktion)
    
    Game->>DB: UPDATE players<br/>SET status='inactive'<br/>WHERE user_id=alice-123
    DB->>Game: OK
    
    Note over Game: Bestimmt nächsten aktiven Spieler<br/>Überspringt inaktive Spieler
    
    Game->>Game: Findet nächsten Spieler:<br/>current_player_index: 1 → 2<br/>Alice (inactive) → Bob (active)
    
    Game->>DB: UPDATE game_state<br/>SET current_player=bob-456,<br/>current_player_index=2,<br/>dice=[0,0,0,0,0],<br/>locked=[false,false,false,false,false],<br/>roll_count=0,<br/>timer_reset=NOW()
    DB->>Game: OK
    
    Game->>SSE: POST /internal/broadcast<br/>{event: "player_timeout",<br/>player: "Alice",<br/>reason: "inactivity"}<br/><br/>{event: "player_status_changed",<br/>player: "Alice",<br/>status: "inactive"}<br/><br/>{event: "turn_changed",<br/>current_player: "Bob",<br/>skipped_players: ["Alice"]}
    
    SSE->>SSE_All: Drei Events senden
    SSE_All->>Frontend_All: Events empfangen
    
    Frontend_All->>Frontend_All: 1. Zeigt Notification:<br/>"Alice wurde wegen Inaktivität übersprungen"
    Frontend_All->>Frontend_All: 2. Markiert Alice visuell als inaktiv<br/>(z.B. ausgegraut, ⏸️ Icon)
    Frontend_All->>Frontend_All: 3. Hebt Bob als aktiven Spieler hervor<br/>Resettet Würfel-Anzeige
    
    SSE_All->>Frontend_Alice: Spezielle Notification für Alice
    Frontend_Alice->>Frontend_Alice: Zeigt Warnung:<br/>"Du wurdest übersprungen!<br/>Interagiere, um wieder aktiv zu werden."
    
    SSE->>Game: OK

    Note over Frontend_All: Bob ist jetzt am Zug<br/>Alice ist inaktiv markiert

    %% Optional: Alice wird wieder aktiv
    Note over Frontend_Alice: Alice kehrt zurück<br/>und klickt irgendwo

    Frontend_Alice->>Gateway: POST /api/game/{game_id}/reactivate<br/>Cookie: jwt=...
    
    Gateway->>Game: POST /internal/game/{game_id}/reactivate<br/>Header: X-User-ID: alice-123
    
    Game->>DB: UPDATE players<br/>SET status='active'<br/>WHERE user_id=alice-123
    DB->>Game: OK
    
    Game->>SSE: POST /internal/broadcast<br/>{event: "player_status_changed",<br/>player: "Alice",<br/>status: "active"}
    
    SSE->>SSE_All: event: player_status_changed
    SSE_All->>Frontend_All: Event empfangen
    Frontend_All->>Frontend_All: Entfernt Inaktiv-Markierung<br/>von Alice
    
    SSE->>Game: OK
    Game->>Gateway: {status: "active"}
    Gateway->>Frontend_Alice: OK
    
    Frontend_Alice->>Frontend_Alice: Zeigt Bestätigung:<br/>"Du bist wieder aktiv!<br/>Warte auf deinen nächsten Zug."

    Note over Frontend_All: Alice ist wieder aktiv<br/>wird in nächster Runde nicht übersprungen
```

## Flow 7: Spieler Disconnect/Reconnect

### Szenario
Bob verliert während des laufenden Spiels die Internetverbindung (oder macht einen Page-Refresh). Seine SSE-Verbindung bricht ab. Das Spiel läuft weiter. Bob stellt die Verbindung wieder her und wird automatisch zurück ins Spiel gebracht.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Browser_Bob as Browser (Bob)
    participant Frontend_Bob as Frontend (Bob)
    participant SSE_Bob as SSE Connection (Bob)
    participant SSE as SSE Service
    participant Gateway as API Gateway
    participant Game as Game Service
    participant DB as PostgreSQL (game_db)
    participant SSE_Others as SSE Connections (andere)
    participant Frontend_Others as Frontend (andere)

    Note over Browser_Bob,Frontend_Bob: Bob ist im Spiel<br/>Verbindung wird unterbrochen

    %% Disconnect
    Browser_Bob->>Frontend_Bob: Verbindung verloren<br/>(Netzwerk aus / Page-Refresh)
    
    Frontend_Bob->>SSE_Bob: SSE-Verbindung bricht ab
    
    SSE_Bob->>SSE: Connection closed (detected)
    
    Note over SSE: Entfernt Bob's Connection<br/>aus lobby_id → connections Map
    
    SSE->>Game: POST /internal/player-disconnected<br/>{game_id, user_id: bob-456}
    
    Game->>DB: UPDATE players<br/>SET status='disconnected',<br/>last_seen=NOW()<br/>WHERE user_id=bob-456
    DB->>Game: OK
    
    Game->>SSE: POST /internal/broadcast<br/>{event: "player_disconnected",<br/>player: "Bob",<br/>status: "disconnected"}
    
    SSE->>SSE_Others: event: player_disconnected
    SSE_Others->>Frontend_Others: Event empfangen
    Frontend_Others->>Frontend_Others: Markiert Bob als disconnected<br/>(z.B. 📡 Icon, ausgegraut)
    
    SSE->>Game: OK

    Note over Frontend_Others: Spiel läuft weiter<br/>Bob wird übersprungen wenn er dran ist

    %% Zwischenzeit: Spiel läuft
    Note over Game: Wenn Bob am Zug wäre:<br/>Überspringt ihn automatisch<br/>(ähnlich wie Timeout)

    %% Reconnect
    Note over Browser_Bob: Bob stellt Verbindung wieder her<br/>(Netzwerk zurück / Page neu laden)

    Browser_Bob->>Frontend_Bob: Seite lädt neu / App startet
    
    Frontend_Bob->>Frontend_Bob: Prüft: JWT-Cookie vorhanden?<br/>game_id im localStorage/Cookie?
    
    Note over Frontend_Bob: Ja! Bob war in Spiel<br/>game_id: abc-123

    Frontend_Bob->>Gateway: GET /api/game/{game_id}/state<br/>Cookie: jwt=...
    
    Gateway->>Gateway: Validiert JWT
    Note over Gateway: Bob's Token noch gültig
    
    Gateway->>Game: GET /internal/game/{game_id}/state<br/>Header: X-User-ID: bob-456
    
    Game->>DB: SELECT game_state, scores, players<br/>WHERE game_id
    DB->>Game: {current_player: alice-123,<br/>dice: [2,4,6,1,3],<br/>roll_count: 2,<br/>scores: {...},<br/>bob_status: "disconnected"}
    
    Game->>Game: Bereitet vollständigen Spielstand auf:<br/>- Kniffel-Block aller Spieler<br/>- Aktueller Spieler & Würfel<br/>- Spieler-Status
    
    Game->>Gateway: {game_state: {...},<br/>current_player: "Alice",<br/>your_status: "disconnected"}
    Gateway->>Frontend_Bob: {game_state: {...}}
    
    Frontend_Bob->>Frontend_Bob: Rendert Spielansicht:<br/>- Zeigt Kniffel-Block<br/>- Hebt aktuellen Spieler hervor<br/>- Zeigt eigenen Status: "Reconnecting..."

    Frontend_Bob->>SSE: GET /events/game/{game_id}<br/>Cookie: jwt=...
    
    SSE->>SSE: Validiert JWT<br/>Extrahiert Bob's User-ID
    
    SSE->>SSE: Registriert neue Connection:<br/>game_id → [..., bob-connection]
    
    SSE-->>Frontend_Bob: SSE Stream opened<br/>event: connected<br/>data: {status: "ok"}
    
    SSE->>Game: POST /internal/player-reconnected<br/>{game_id, user_id: bob-456}
    
    Game->>DB: UPDATE players<br/>SET status='active',<br/>last_seen=NOW()<br/>WHERE user_id=bob-456
    DB->>Game: OK
    
    Game->>SSE: POST /internal/broadcast<br/>{event: "player_reconnected",<br/>player: "Bob",<br/>status: "active"}
    
    SSE->>SSE_Others: event: player_reconnected
    SSE_Others->>Frontend_Others: Event empfangen
    Frontend_Others->>Frontend_Others: Entfernt Disconnect-Markierung<br/>Bob ist wieder da ✓
    
    SSE->>SSE_Bob: event: player_reconnected (auch an Bob)
    SSE_Bob->>Frontend_Bob: Event empfangen
    Frontend_Bob->>Frontend_Bob: Zeigt Bestätigung:<br/>"Verbindung wiederhergestellt!"<br/>Entfernt "Reconnecting..."-Status
    
    SSE->>Game: OK

    Note over Frontend_Bob,Frontend_Others: Bob ist wieder vollständig im Spiel<br/>Empfängt alle zukünftigen Events
```

## Flow 8: Spielende erkennen & Rangliste

### Szenario
Alice füllt ihr letztes (13.) Feld aus. Der Game Service erkennt, dass alle Spieler fertig sind, berechnet die Endpunkte, erstellt eine Rangliste und leitet alle Spieler zur Endeseite weiter.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Frontend_Alice as Frontend (Alice)
    participant Gateway as API Gateway
    participant Game as Game Service
    participant DB as PostgreSQL (game_db)
    participant SSE as SSE Service
    participant SSE_All as SSE Connections (alle)
    participant Frontend_All as Frontend (alle)

    Note over Frontend_Alice: Alice ist am Zug<br/>Würfel: [5,5,5,5,5]<br/>Letztes freies Feld: "Kniffel"

    Frontend_Alice->>Gateway: POST /api/game/{game_id}/select-field<br/>{field: "kniffel"}<br/>Cookie: jwt=...
    
    Gateway->>Game: POST /internal/game/{game_id}/select-field<br/>{field: "kniffel"}<br/>Header: X-User-ID: alice-123
    
    Game->>DB: SELECT game_state, scores
    DB->>Game: {current_player: alice-123,<br/>dice: [5,5,5,5,5],<br/>alice_scores: {<br/>  einser: 5, zweier: 10, ...,<br/>  kniffel: null,<br/>  filled_fields: 12<br/>}}
    
    Game->>Game: Berechnet Punkte für "kniffel"<br/>5 gleiche → 50 Punkte
    
    Game->>DB: UPDATE scores<br/>SET kniffel=50,<br/>filled_fields=13<br/>WHERE game_id AND user_id=alice-123
    DB->>Game: OK
    
    Note over Game: Prüft: Sind alle Spieler fertig?
    
    Game->>DB: SELECT COUNT(*) FROM scores<br/>WHERE game_id AND filled_fields < 13
    DB->>Game: {count: 0}
    
    Note over Game: Ja! Alle haben 13 Felder gefüllt<br/>→ Spielende!
    
    Game->>DB: SELECT scores FROM scores<br/>WHERE game_id
    DB->>Game: {<br/>  alice: {einser:5, zweier:10, ..., kniffel:50},<br/>  bob: {...},<br/>  charlie: {...}<br/>}
    
    Game->>Game: Berechnet Endpunkte für alle:<br/><br/>Alice:<br/>- Oben: 75 (≥63 → Bonus +35)<br/>- Unten: 120<br/>- Total: 230<br/><br/>Bob:<br/>- Oben: 58 (kein Bonus)<br/>- Unten: 95<br/>- Total: 153<br/><br/>Charlie:<br/>- Oben: 68 (≥63 → Bonus +35)<br/>- Unten: 135<br/>- Total: 238
    
    Game->>Game: Erstellt Rangliste (sortiert):<br/>1. Charlie (238)<br/>2. Alice (230)<br/>3. Bob (153)
    
    Game->>DB: UPDATE games<br/>SET status='finished',<br/>finished_at=NOW(),<br/>winner_user_id=charlie-789,<br/>final_rankings=JSON(...)
    DB->>Game: OK
    
    Game->>SSE: POST /internal/broadcast<br/>{event: "field_selected",<br/>player: "Alice",<br/>field: "kniffel",<br/>points: 50}<br/><br/>{event: "game_ended",<br/>game_id,<br/>rankings: [<br/>  {rank: 1, username: "Charlie", points: 238},<br/>  {rank: 2, username: "Alice", points: 230},<br/>  {rank: 3, username: "Bob", points: 153}<br/>],<br/>winner: "Charlie"}
    
    SSE->>SSE_All: Zwei Events senden
    SSE_All->>Frontend_All: Events empfangen
    
    Frontend_All->>Frontend_All: 1. Aktualisiert Kniffel-Block<br/>(Alice: Kniffel = 50)
    
    Note over Frontend_All: Kurze Pause (2 Sekunden)<br/>damit Spieler Endstand sehen
    
    Frontend_All->>Frontend_All: 2. Navigiert zu /game/{game_id}/results
    Frontend_All->>Frontend_All: Zeigt Endeseite:<br/>🏆 Charlie (238)<br/>🥈 Alice (230)<br/>🥉 Bob (153)
    
    SSE->>Game: OK
    Game->>Gateway: {status: "finished",<br/>winner: "Charlie"}
    Gateway->>Frontend_Alice: OK

    Note over Frontend_All: Alle Spieler sehen Rangliste<br/>Buttons: "Nochmal spielen" | "Zur Startseite"
```

---

## Wichtige Aspekte in diesem Flow:

### **Spielende-Erkennung**
1. Nach jeder Feldwahl: **Prüfung ob Spieler fertig** (`filled_fields = 13`)
2. Wenn Spieler fertig: **Prüfung ob ALLE fertig** (`COUNT(*) WHERE filled_fields < 13`)
3. Wenn alle fertig: **Spielende-Logik triggern**

### **Endpunkte-Berechnung**
```
Oberer Teil:
  Einser + Zweier + Dreier + Vierer + Fünfer + Sechser
  WENN Summe ≥ 63 → Bonus +35

Unterer Teil:
  Dreierpasch + Viererpasch + Full House + 
  Kleine Straße + Große Straße + Kniffel + Chance

Gesamtpunktzahl:
  Oberer Teil + Bonus + Unterer Teil
```

## Flow 9: Nochmal spielen / Neue Runde

### Szenario
Die Spieler sind auf der Endeseite. Alice (ursprünglicher Lobby-Leiter) klickt auf "Nochmal spielen". Eine neue Lobby wird mit denselben Spielern erstellt, und alle werden zurück zur Lobby-Ansicht gebracht.

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
    participant SSE_All as SSE Connections (alle)
    participant Frontend_All as Frontend (alle)

    Note over Frontend_All: Alle auf Endeseite<br/>game_id: abc-123<br/>Rangliste wird angezeigt

    Frontend_Alice->>Gateway: POST /api/game/{game_id}/rematch<br/>Cookie: jwt=...
    
    Gateway->>Gateway: Validiert JWT
    Note over Gateway: Alice's User-ID: alice-123
    
    Gateway->>Lobby: POST /internal/game/{game_id}/rematch<br/>Header: X-User-ID: alice-123
    
    Lobby->>DB_Lobby: SELECT lobby, players<br/>WHERE game_id='abc-123'
    DB_Lobby->>Lobby: {<br/>  old_lobby_id: lobby-001,<br/>  leader: alice-123,<br/>  players: [alice-123, bob-456, charlie-789]<br/>}
    
    Note over Lobby: Validiert:<br/>- Alice war ursprünglicher Leiter<br/>- Spiel ist beendet (status='finished')
    
    Lobby->>Lobby: Erstellt neue Lobby:<br/>- Neuer Join-Code: "XYZ789"<br/>- Status: "waiting"<br/>- Leader: Alice (bleibt Leiter)<br/>- Reference: old_game_id
    
    Lobby->>DB_Lobby: INSERT INTO lobbies<br/>(lobby_id: lobby-002,<br/>join_code: "XYZ789",<br/>status: "waiting",<br/>leader_user_id: alice-123,<br/>created_from_game: abc-123)
    DB_Lobby->>Lobby: {new_lobby_id: lobby-002}
    
    Lobby->>DB_Lobby: INSERT INTO players<br/>(lobby-002, alice-123, "Alice")<br/>(lobby-002, bob-456, "Bob")<br/>(lobby-002, charlie-789, "Charlie")
    DB_Lobby->>Lobby: OK
    
    Note over Lobby: Alle Spieler sind automatisch<br/>in der neuen Lobby
    
    Lobby->>Game: POST /internal/game/{game_id}/close<br/>{game_id: abc-123}
    
    Game->>DB_Game: UPDATE games<br/>SET rematch_lobby_id='lobby-002',<br/>archived=true<br/>WHERE game_id='abc-123'
    DB_Game->>Game: OK
    
    Game->>Lobby: OK (Game archiviert)
    
    Lobby->>SSE: POST /internal/broadcast<br/>{event: "rematch_created",<br/>old_game_id: "abc-123",<br/>new_lobby_id: "lobby-002",<br/>join_code: "XYZ789",<br/>players: ["Alice", "Bob", "Charlie"]}
    
    SSE->>SSE: Findet Connections für old_game_id
    
    SSE->>SSE_All: event: rematch_created<br/>data: {new_lobby_id, join_code, ...}
    
    SSE_All->>Frontend_All: Event empfangen
    
    Frontend_All->>Frontend_All: Zeigt Notification:<br/>"Neue Runde wird erstellt..."
    
    Frontend_All->>Frontend_All: Navigiert zu /lobby/{new_lobby_id}
    Frontend_All->>Frontend_All: Zeigt Lobby-Ansicht:<br/>Join-Code: XYZ789<br/>Spieler: Alice (👑), Bob, Charlie
    
    SSE->>Lobby: OK
    
    Lobby->>Gateway: {<br/>  new_lobby_id: "lobby-002",<br/>  join_code: "XYZ789",<br/>  players: ["Alice", "Bob", "Charlie"]<br/>}
    Gateway->>Frontend_Alice: {new_lobby_id, join_code, ...}

    Note over Frontend_All: Alle in neuer Lobby<br/>Warten auf Spielstart

    %% Alte SSE-Verbindungen schließen, neue aufbauen
    Frontend_All->>SSE_All: Schließt alte SSE-Verbindung<br/>(für old_game_id)
    
    Frontend_All->>SSE: GET /events/lobby/{new_lobby_id}<br/>Cookie: jwt=...
    
    SSE->>SSE: Registriert neue Connections<br/>für new_lobby_id
    
    SSE-->>Frontend_All: SSE Stream opened
    
    Note over Frontend_All,SSE: Neue SSE-Verbindungen aktiv<br/>für die neue Lobby

    Note over Frontend_Alice: Alice kann jetzt wieder<br/>"Spiel starten" klicken<br/>(wie in Flow 3)
```

## Flow 10: SSE-Connection-Management & Event-Broadcasting

### Szenario
Dies ist kein User-Story-Flow, sondern zeigt die technische Infrastruktur des SSE Service. Wir zeigen den kompletten Lifecycle: Connection aufbauen, Events empfangen, Broadcasting, Connection schließen.

### Sequenzdiagramm

```mermaid
sequenceDiagram
    participant Frontend as Frontend (Client)
    participant SSE as SSE Service
    participant Auth as Auth Service
    participant Service as Service (Lobby/Game)
    participant Connections as In-Memory Map

    %% Connection aufbauen
    Note over Frontend: User ist in Lobby/Game<br/>lobby_id: lobby-123

    Frontend->>SSE: GET /events/lobby/lobby-123<br/>Cookie: jwt=eyJhbG...
    
    SSE->>SSE: Extrahiert JWT aus Cookie
    
    SSE->>Auth: POST /internal/validate<br/>{token: "eyJhbG..."}
    Auth->>Auth: Validiert JWT-Signatur<br/>Prüft Expiry
    Auth->>SSE: {valid: true,<br/>user_id: "alice-123",<br/>username: "Alice"}
    
    Note over SSE: JWT gültig!<br/>Erstellt SSE-Connection
    
    SSE->>SSE: Erstellt Connection-Objekt:<br/>{<br/>  connection_id: uuid(),<br/>  user_id: "alice-123",<br/>  username: "Alice",<br/>  lobby_id: "lobby-123",<br/>  opened_at: NOW(),<br/>  response_writer: w<br/>}
    
    SSE->>Connections: Map[lobby-123].add(connection)
    
    Note over Connections: In-Memory Struktur:<br/>Map[lobby_id] → []Connection<br/><br/>lobby-123 → [<br/>  {user: Alice, conn: ...},<br/>  {user: Bob, conn: ...}<br/>]
    
    Connections->>SSE: OK
    
    SSE-->>Frontend: HTTP 200 OK<br/>Content-Type: text/event-stream<br/>Cache-Control: no-cache<br/>Connection: keep-alive<br/><br/>event: connected<br/>data: {"status": "ok"}<br/><br/>
    
    Note over Frontend,SSE: SSE-Stream ist offen<br/>Verbindung bleibt bestehen

    %% Event von Service empfangen und broadcasten
    Note over Service: Ein Event tritt ein<br/>(z.B. Spieler würfelt)

    Service->>SSE: POST /internal/broadcast<br/>{<br/>  lobby_id: "lobby-123",<br/>  event_type: "dice_rolled",<br/>  data: {<br/>    player: "Bob",<br/>    dice: [3,4,6,2,1],<br/>    roll_count: 1<br/>  }<br/>}
    
    SSE->>SSE: Validiert Request<br/>(kommt von internem Service)
    
    SSE->>Connections: connections = Map[lobby-123].get()
    Connections->>SSE: [<br/>  {user: Alice, conn: ...},<br/>  {user: Bob, conn: ...},<br/>  {user: Charlie, conn: ...}<br/>]
    
    Note over SSE: Broadcasting an alle<br/>3 Connections in Lobby

    SSE->>SSE: FOR EACH connection IN connections:
    
    loop Für jede Connection
        SSE->>SSE: Formatiert SSE-Message:<br/>event: dice_rolled<br/>data: {"player":"Bob",...}<br/>id: msg-{counter}<br/><br/>
        
        SSE->>Frontend: Sendet über response_writer
        
        Note over SSE: Prüft: Senden erfolgreich?
        
        alt Connection aktiv
            Frontend->>Frontend: Event empfangen ✓
        else Connection tot
            SSE->>SSE: Markiert Connection als "broken"
            SSE->>Connections: Map[lobby-123].remove(connection)
            Note over SSE: Cleanup: Tote Connection entfernt
        end
    end
    
    SSE->>Service: HTTP 200 OK<br/>{status: "broadcasted",<br/>recipients: 3}

    Note over Frontend: Client verarbeitet Event<br/>im EventSource.onmessage

    %% Heartbeat / Keep-Alive
    Note over SSE: Periodisch (alle 30s):<br/>Keep-Alive senden

    SSE->>Frontend: :keepalive\n\n
    Note over SSE,Frontend: Verhindert Timeout,<br/>erkennt tote Connections

    %% Connection schließen (Client-initiiert)
    Note over Frontend: User navigiert weg<br/>oder schließt Browser

    Frontend->>Frontend: EventSource.close()
    
    Frontend->>SSE: HTTP Connection closed
    
    SSE->>SSE: Erkennt Connection-Close
    
    SSE->>Connections: Map[lobby-123].remove(connection)
    
    Note over Connections: Alice's Connection entfernt<br/><br/>lobby-123 → [<br/>  {user: Bob, conn: ...},<br/>  {user: Charlie, conn: ...}<br/>]
    
    SSE->>SSE: Cleanup:<br/>- Response-Writer schließen<br/>- Goroutine beenden<br/>- Memory freigeben

    %% Connection schließen (Server-initiiert)
    Note over SSE: Alternative: Server schließt<br/>(z.B. bei Spielende)

    Service->>SSE: POST /internal/close-lobby<br/>{lobby_id: "lobby-123"}
    
    SSE->>Connections: connections = Map[lobby-123].getAll()
    Connections->>SSE: [Bob, Charlie]
    
    loop Für jede Connection
        SSE->>Frontend: event: lobby_closed<br/>data: {"reason":"game_ended"}<br/><br/>
        SSE->>Frontend: HTTP Connection close
    end
    
    SSE->>Connections: Map[lobby-123].delete()
    
    Note over Connections: Gesamte Lobby<br/>aus Map entfernt
    
    SSE->>Service: OK

    %% Automatisches Cleanup
    Note over SSE: Background-Task (alle 5 Min):<br/>Aufräumen von Zombie-Connections

    SSE->>Connections: FOR EACH lobby IN Map:
    loop Für jede Lobby
        SSE->>SSE: Prüft alle Connections:<br/>- Letzter Ping > 2 Minuten?<br/>- Response-Writer tot?
        
        alt Connection tot
            SSE->>Connections: remove(connection)
            Note over SSE: Zombie entfernt
        end
    end
```