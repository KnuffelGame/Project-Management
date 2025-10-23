# Microservices-Architektur: Knuffel
## Multiplayer Kniffel Web-Anwendung

**Version:** 1.0  
**Datum:** 23.10.2025  
**Status:** ✅ Finalisiert

---

## 1. Architektur-Übersicht

### 1.1 Service-Landschaft (MVP)

```
┌─────────────────────────────────────────────────────────────────┐
│                          Knuffel System                          │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   API    │  │   Auth   │  │  Lobby   │  │   Game   │       │
│  │ Gateway  │  │ Service  │  │ Service  │  │ Service  │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│                                     │            │               │
│                                     ▼            ▼               │
│  ┌──────────┐              ┌──────────┐  ┌──────────┐         │
│  │   SSE    │              │ lobby_db │  │ game_db  │         │
│  │ Service  │              └──────────┘  └──────────┘         │
│  └──────────┘                     └──────────┘                  │
│                                   PostgreSQL                    │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     Frontend (React)                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Service-Übersicht

| Service | Verantwortlichkeit | Technologie | Datenhaltung |
|---------|-------------------|-------------|--------------|
| **Frontend** | React UI, SSE-Client, REST-Client | React + Nginx/Apache | - |
| **API Gateway** | Routing, zentrale JWT-Validierung | Go | - |
| **Auth Service** | JWT erstellen/validieren | Go | - (stateless) |
| **Lobby Service** | Lobbies, Users, Join-Codes | Go | PostgreSQL (lobby_db) |
| **Game Service** | Spiellogik, Würfeln, Punkte | Go | PostgreSQL (game_db) |
| **SSE Service** | Event-Broadcasting zentral | Go | In-Memory (Connections) |

---

## 2. System-Architektur-Diagramm

### 2.1 Komponenten & Kommunikation

```mermaid
graph TB
    subgraph External
        Browser[Browser/Client]
    end
    
    subgraph "Traefik Reverse Proxy"
        Traefik[Traefik Router]
    end
    
    subgraph "Frontend Layer"
        Frontend[Frontend<br/>React + Nginx]
    end
    
    subgraph "API Layer"
        Gateway[API Gateway<br/>Routing + Auth]
    end
    
    subgraph "Service Layer"
        Auth[Auth Service<br/>JWT Operations]
        Lobby[Lobby Service<br/>Lobbies + Users]
        Game[Game Service<br/>Spiellogik]
        SSE[SSE Service<br/>Event Broadcasting]
    end
    
    subgraph "Data Layer"
        DB[(PostgreSQL<br/>lobby_db + game_db)]
    end
    
    Browser -->|HTTPS| Traefik
    Traefik -->|/| Frontend
    Traefik -->|/api/*| Gateway
    Traefik -->|/events/*| SSE
    
    Frontend -->|REST API| Gateway
    Frontend -.->|SSE Stream| SSE
    
    Gateway -->|validate JWT| Auth
    Gateway -->|REST| Lobby
    Gateway -->|REST| Game
    
    Lobby -->|publish event| SSE
    Game -->|publish event| SSE
    
    Lobby -->|SQL| DB
    Game -->|SQL| DB
    
    style Browser fill:#e1f5ff
    style Frontend fill:#fff4e1
    style Gateway fill:#ffe1e1
    style Auth fill:#f0e1ff
    style Lobby fill:#e1ffe1
    style Game fill:#e1ffe1
    style SSE fill:#ffe1f0
    style DB fill:#e1e1e1
```

### 2.2 Subdomain-Routing

```
┌─────────────────────────────────────────────────────────────┐
│                    Traefik Reverse Proxy                     │
│                     (Port 80/443, SSL)                       │
└───────────────────┬───────────────────┬────────────────────┘
                    │                   │
    ┌───────────────┼───────────────────┼───────────────┐
    │               │                   │               │
    ▼               ▼                   ▼               ▼
┌─────────┐  ┌──────────┐       ┌──────────┐   ┌──────────┐
│Frontend │  │   API    │       │   SSE    │   │  Others  │
│ (Nginx) │  │ Gateway  │       │ Service  │   │ (Future) │
└─────────┘  └──────────┘       └──────────┘   └──────────┘
     │             │                   │
     │             │                   │
knuffel.      api.knuffel.        events.knuffel.
uni.de           uni.de               uni.de
```

**Routing-Regeln:**
- `https://knuffel.uni.de` → Frontend (Nginx)
- `https://api.knuffel.uni.de/*` → API Gateway
- `https://events.knuffel.uni.de/lobby/:id` → SSE Service
- `https://events.knuffel.uni.de/game/:id` → SSE Service

---

## 3. Service-Verantwortlichkeiten

### 3.1 API Gateway

**Hauptaufgaben:**
- ✅ Routing von Client-Requests zu Backend-Services
- ✅ Zentrale JWT-Validierung (via Auth Service)
- ✅ Request-Weiterleitung mit User-Context
- ✅ CORS-Handling
- ✅ Rate-Limiting (optional)

**Technische Details:**
- Sprache: Go
- Framework: `gorilla/mux` oder `chi`
- Keine Datenhaltung (stateless)

**Endpoints (extern):**
```
POST   /api/auth/login          → Auth Service
POST   /api/auth/guest          → Auth Service

POST   /api/lobby/create        → Lobby Service
POST   /api/lobby/:id/join      → Lobby Service
POST   /api/lobby/:id/start     → Lobby Service
POST   /api/lobby/:id/kick/:uid → Lobby Service
GET    /api/lobby/:id/state     → Lobby Service

POST   /api/game/:id/roll       → Game Service
POST   /api/game/:id/toggle/:i  → Game Service
POST   /api/game/:id/select     → Game Service
POST   /api/game/:id/leave      → Game Service
POST   /api/game/:id/end        → Game Service
GET    /api/game/:id/state      → Game Service
```

**Auth-Flow:**
```
1. Request kommt mit Cookie: jwt=xxx
2. Gateway extrahiert JWT
3. Gateway → Auth Service: POST /internal/validate
4. Auth Service response: {valid: true, user_id: "abc", username: "Alice"}
5. Gateway fügt Header hinzu:
   X-User-ID: abc
   X-Username: Alice
6. Gateway leitet Request weiter zu Service
```

---

### 3.2 Auth Service

**Hauptaufgaben:**
- ✅ JWT erstellen (für Gast-Accounts)
- ✅ JWT validieren (für alle Requests)
- ✅ Token-Refresh (für OIDC, Stretch-Goal)

**Technische Details:**
- Sprache: Go
- Library: `golang-jwt/jwt`
- Keine Datenhaltung (stateless)
- Secret-Key aus Environment-Variable

**Endpoints (intern):**
```
POST /internal/create
  Request:  {"user_id": "abc", "username": "Alice"}
  Response: {"token": "eyJhbG..."}

POST /internal/validate
  Request:  {"token": "eyJhbG..."}
  Response: {"valid": true, "user_id": "abc", "username": "Alice"}
            {"valid": false, "error": "token expired"}
```

**JWT-Claims:**
```json
{
  "sub": "user-id-123",           // Subject (User-ID)
  "name": "Alice",                // Username
  "iat": 1698345600,              // Issued At
  "exp": 1698432000,              // Expiry (24h)
  "iss": "knuffel-auth-service"   // Issuer
}
```

**Stretch-Goal: OIDC-Integration**
- Zusätzlicher Endpoint: `POST /internal/oidc-exchange`
- Google OAuth Token → Knuffel JWT

---

### 3.3 Lobby Service

**Hauptaufgaben:**
- ✅ Lobby-Lifecycle (erstellen, Status, löschen)
- ✅ User-Management (Gast-Accounts)
- ✅ Join-Code-Generierung
- ✅ Spieler-zu-Lobby-Mapping
- ✅ Lobby-Leiter-Verwaltung
- ✅ Spiel-Start initiieren (→ Game Service)
- ✅ Event-Publishing (→ SSE Service)

**Technische Details:**
- Sprache: Go
- Datenbank: PostgreSQL (lobby_db)
- Tabellen: users, lobbies, players

**Endpoints (extern via Gateway):**
```
POST /api/lobby/create
  Request:  {"username": "Alice"}
  Response: {"lobby_id": "abc123", "join_code": "XYZ789", "jwt": "..."}

POST /api/lobby/:id/join
  Request:  {"username": "Bob"}
  Response: {"lobby_id": "abc123", "jwt": "..."}

POST /api/lobby/:id/start
  Response: {"game_id": "abc123"}

POST /api/lobby/:id/kick/:user_id
  Response: 204 No Content

GET /api/lobby/:id/state
  Response: {
    "lobby_id": "abc123",
    "join_code": "XYZ789",
    "status": "waiting",
    "leader_id": "user-1",
    "players": [...]
  }
```

**Endpoints (intern):**
```
POST /internal/lobby/delete/:id
  (Von Game Service, wenn Spiel beendet)
```

**Event-Publishing:**
```
Lobby Service → SSE Service: POST /internal/broadcast
{
  "lobby_id": "abc123",
  "event_type": "player_joined",
  "data": {"username": "Bob", "player_count": 3}
}

Events:
- player_joined
- player_left
- player_kicked
- leader_changed
- game_started
```

---

### 3.4 Game Service

**Hauptaufgaben:**
- ✅ Spiellogik (Würfeln, Fixieren, Feld-Auswahl)
- ✅ Punkteberechnung (inkl. Bonus, Mehrfach-Kniffel)
- ✅ Spielzustand-Verwaltung (wer ist dran, Würfelwerte)
- ✅ Timeout-Mechanismus (40s, resettet bei Interaktion)
- ✅ Spielende-Erkennung
- ✅ Event-Publishing (→ SSE Service)

**Technische Details:**
- Sprache: Go
- Datenbank: PostgreSQL (game_db)
- Tabellen: games, scores, dice_state
- Würfel-Logik: Internes Package `dice/`

**Endpoints (extern via Gateway):**
```
POST /api/game/:id/roll
  Response: {"values": [3,5,1,4,2], "roll_count": 1}

POST /api/game/:id/toggle/:index
  Request:  {"index": 2}  // Würfel 0-4
  Response: {"fixed": [false, false, true, false, false]}

POST /api/game/:id/select
  Request:  {"field": "threes", "value": 9}
  Response: {"next_player": "user-2"}

POST /api/game/:id/leave
  Response: 204 No Content

POST /api/game/:id/end
  (Nur Lobby-Leiter)
  Response: {"final_scores": [...]}

GET /api/game/:id/state
  Response: {
    "game_id": "abc123",
    "current_player": "user-1",
    "turn": 5,
    "dice": {"values": [...], "fixed": [...], "roll_count": 2},
    "scores": {...},
    "timeout_remaining": 32
  }
```

**Endpoints (intern):**
```
POST /internal/game/create
  Request: {
    "lobby_id": "abc123",
    "players": [{"id": "u1", "username": "Alice"}, ...]
  }
  Response: {"game_id": "abc123", "turn_order": [...]}
```

**Event-Publishing:**
```
Game Service → SSE Service: POST /internal/broadcast
{
  "lobby_id": "abc123",
  "event_type": "dice_rolled",
  "data": {"values": [3,5,1,4,2], "player": "Alice"}
}

Events:
- dice_rolled
- dice_toggled
- field_selected
- turn_changed
- player_inactive
- player_reconnected
- timeout_warning (bei 10s verbleibend)
- game_ended
```

**Spiellogik-Komponenten:**
```
game/
├── dice/
│   ├── roller.go       // Zufallszahlen 1-6
│   └── state.go        // Fixierung-Management
├── scoring/
│   ├── calculator.go   // Punkteberechnung
│   ├── validator.go    // Regelvalidierung
│   └── bonus.go        // Bonus + Mehrfach-Kniffel
├── timeout/
│   ├── manager.go      // 40s Timer pro Spieler
│   └── handler.go      // Timeout-Actions
└── state/
    ├── game.go         // Game-State-Machine
    └── turn.go         // Zugverwaltung
```

---

### 3.5 SSE Service

**Hauptaufgaben:**
- ✅ Zentrale Verwaltung aller SSE-Verbindungen
- ✅ Event-Broadcasting zu Clients
- ✅ Connection-Management (Subscribe/Unsubscribe)
- ✅ Event-Routing nach Lobby-ID

**Technische Details:**
- Sprache: Go
- Datenhaltung: In-Memory (Map: lobby_id → []SSE-Connections)
- Keine Datenbank
- Kein Redis (für MVP, Single-Instance)

**Architektur:**
```
┌─────────────────────────────────────────────────┐
│              SSE Service                         │
│                                                  │
│  ┌─────────────────────────────────────────┐   │
│  │  Connection Manager (In-Memory)          │   │
│  │                                          │   │
│  │  Map[lobby_id] → []SSE-Connections       │   │
│  │                                          │   │
│  │  lobby-abc123:                           │   │
│  │    - Connection 1 (User Alice)           │   │
│  │    - Connection 2 (User Bob)             │   │
│  │    - Connection 3 (User Charlie)         │   │
│  └─────────────────────────────────────────┘   │
│                                                  │
│  ┌─────────────────────────────────────────┐   │
│  │  Event Broadcaster                       │   │
│  │  - Receives events from Services         │   │
│  │  - Routes to correct lobby connections   │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**Endpoints (Client):**
```
GET /events/lobby/:id
  Response: SSE-Stream (text/event-stream)
  
  event: player_joined
  data: {"username": "Bob", "player_count": 3}
  
  event: game_started
  data: {"game_id": "abc123"}

GET /events/game/:id
  Response: SSE-Stream (text/event-stream)
  
  event: dice_rolled
  data: {"values": [3,5,1,4,2], "player": "Alice"}
  
  event: field_selected
  data: {"player": "Alice", "field": "threes", "points": 9}
```

**Endpoints (intern - Services):**
```
POST /internal/broadcast
  Request: {
    "lobby_id": "abc123",
    "event_type": "player_joined",
    "source_service": "lobby",
    "data": {...}
  }
  Response: 204 No Content
```

**Connection-Flow:**
```
1. Client öffnet SSE-Verbindung
   GET /events/lobby/abc123
   Cookie: jwt=xxx

2. SSE Service:
   - Validiert JWT (optional, kann auch API Gateway vorher)
   - Extrahiert User-ID
   - Registriert Connection: lobby_abc123 → [conn1, conn2, ...]

3. Event kommt von Service:
   POST /internal/broadcast {"lobby_id": "abc123", ...}

4. SSE Service:
   - Lookup: lobby_abc123 → [conn1, conn2, conn3]
   - Sendet Event an alle Connections:
     event: player_joined\ndata: {...}\n\n

5. Client schließt Verbindung:
   - SSE Service entfernt Connection aus Map
```

**Stretch-Goal: Redis Pub/Sub für Multi-Instance:**
```
Falls SSE Service horizontal skaliert:
- Events werden zu Redis-Channel published
- Alle SSE-Instanzen subscriben Channel
- Jede Instanz broadcastet zu ihren lokalen Connections
```

---

### 3.6 Frontend

**Hauptaufgaben:**
- ✅ React-UI für alle Screens
- ✅ REST-API-Client (zu API Gateway)
- ✅ SSE-Client (zu SSE Service)
- ✅ State-Management (React Context/Redux)
- ✅ Routing (React Router)

**Technische Details:**
- Sprache: TypeScript + React
- Styling: Tailwind CSS
- HTTP-Client: Axios (mit Cookie-Support)
- SSE-Client: Native EventSource API
- Build: Vite oder Create-React-App
- Deployment: Nginx oder Apache (Docker)

**Architektur:**
```
Frontend (React)
├── Components/
│   ├── Lobby/
│   │   ├── LobbyCreate.tsx
│   │   ├── LobbyJoin.tsx
│   │   ├── LobbyWaiting.tsx
│   │   └── PlayerList.tsx
│   ├── Game/
│   │   ├── DiceSlotMachine.tsx
│   │   ├── ScoreBoard.tsx
│   │   ├── FieldSelector.tsx
│   │   └── TurnIndicator.tsx
│   └── EndGame/
│       ├── FinalRanking.tsx
│       └── PlayAgainButton.tsx
├── Services/
│   ├── apiClient.ts       // Axios-Config
│   ├── sseClient.ts       // EventSource-Wrapper
│   └── authService.ts     // JWT-Handling
├── Hooks/
│   ├── useLobbyEvents.ts  // SSE-Hook für Lobby
│   ├── useGameEvents.ts   // SSE-Hook für Game
│   └── useAuth.ts         // Auth-Context-Hook
└── State/
    ├── AuthContext.tsx
    ├── LobbyContext.tsx
    └── GameContext.tsx
```

**SSE-Integration Beispiel:**
```typescript
// hooks/useGameEvents.ts
import { useEffect } from 'react';

export function useGameEvents(gameId: string) {
  useEffect(() => {
    const eventSource = new EventSource(
      `https://events.knuffel.uni.de/game/${gameId}`,
      { withCredentials: true } // Sendet Cookie!
    );

    eventSource.addEventListener('dice_rolled', (e) => {
      const data = JSON.parse(e.data);
      // Update React State
      setDiceValues(data.values);
    });

    eventSource.addEventListener('field_selected', (e) => {
      const data = JSON.parse(e.data);
      // Update ScoreBoard
      updateScore(data.player, data.field, data.points);
    });

    eventSource.addEventListener('turn_changed', (e) => {
      const data = JSON.parse(e.data);
      setCurrentPlayer(data.player);
    });

    return () => eventSource.close();
  }, [gameId]);
}
```

---

## 4. Service-Kommunikation

### 4.1 Kommunikations-Matrix

| Von ↓ / Zu → | API Gateway | Auth | Lobby | Game | SSE | DB |
|--------------|-------------|------|-------|------|-----|----|
| **Frontend** | REST | - | - | - | SSE | - |
| **API Gateway** | - | REST | REST | REST | - | - |
| **Auth** | - | - | - | - | - | - |
| **Lobby** | - | - | - | REST | REST | SQL |
| **Game** | - | - | - | - | REST | SQL |
| **SSE** | - | - | - | - | - | - |

### 4.2 Protokolle

| Kommunikation | Protokoll | Technologie |
|---------------|-----------|-------------|
| Frontend → API Gateway | REST (HTTPS) | Axios |
| Frontend → SSE Service | SSE (HTTPS) | EventSource |
| API Gateway → Auth | REST (HTTP) | Go http.Client |
| API Gateway → Lobby | REST (HTTP) | Go http.Client |
| API Gateway → Game | REST (HTTP) | Go http.Client |
| Lobby → Game | REST (HTTP) | Go http.Client |
| Lobby → SSE | REST (HTTP) | Go http.Client |
| Game → SSE | REST (HTTP) | Go http.Client |
| Services → PostgreSQL | SQL | `lib/pq` Driver |

### 4.3 Inter-Service-Kommunikation

**Service Discovery:**
- MVP: Statische Config (Docker-Compose Service-Namen)
- Stretch: Consul/Eureka

**Beispiel Docker-Compose:**
```yaml
services:
  lobby:
    environment:
      - GAME_SERVICE_URL=http://game:8080
      - SSE_SERVICE_URL=http://sse:8080
      - AUTH_SERVICE_URL=http://auth:8080
```

**Retry-Logik:**
- Exponential Backoff bei Service-Ausfällen
- Circuit-Breaker (optional, Stretch-Goal)

---

## 5. Datenfluss-Beispiele

### 5.1 Lobby erstellen & beitreten

```mermaid
sequenceDiagram
    participant C as Client
    participant T as Traefik
    participant F as Frontend
    participant G as API Gateway
    participant A as Auth Service
    participant L as Lobby Service
    participant S as SSE Service
    participant DB as PostgreSQL

    Note over C,DB: 1. Lobby erstellen
    C->>T: HTTPS Request (knuffel.uni.de)
    T->>F: Route to Frontend
    F->>C: React SPA
    
    C->>F: Click "Lobby erstellen", Enter "Alice"
    F->>T: POST api.knuffel.uni.de/api/lobby/create
    T->>G: Route to Gateway
    
    Note over G,A: Auth-Check (für neuen User: Skip)
    
    G->>L: POST /lobby/create {"username": "Alice"}
    L->>DB: INSERT users (username='Alice')
    L->>DB: INSERT lobbies (join_code='XYZ789', leader_id=1)
    L->>A: POST /internal/create {"user_id": "1", "username": "Alice"}
    A->>L: {"token": "eyJhbG..."}
    L->>S: POST /internal/broadcast (lobby_created event)
    L->>G: {"lobby_id": "abc123", "join_code": "XYZ789", "jwt": "..."}
    G->>F: Response with Set-Cookie: jwt=...
    F->>C: Redirect to /lobby/abc123
    
    Note over C,S: 2. SSE-Verbindung öffnen
    F->>T: GET events.knuffel.uni.de/lobby/abc123
    T->>S: Route to SSE Service (Cookie: jwt=...)
    S->>F: SSE-Stream established
    
    Note over C,DB: 3. Zweiter Spieler tritt bei
    C->>F: Enter Join-Code "XYZ789", Username "Bob"
    F->>T: POST api.knuffel.uni.de/api/lobby/abc123/join
    T->>G: Route to Gateway
    G->>A: POST /internal/validate (JWT from Cookie)
    A->>G: Skip (new user, no token yet)
    G->>L: POST /lobby/abc123/join {"username": "Bob"}
    L->>DB: INSERT users (username='Bob')
    L->>DB: INSERT players (lobby_id, user_id=2)
    L->>A: POST /internal/create {"user_id": "2", "username": "Bob"}
    A->>L: {"token": "eyJhbG..."}
    L->>S: POST /internal/broadcast {"event": "player_joined", "data": {"username": "Bob"}}
    S->>F: SSE Event: player_joined
    L->>G: {"lobby_id": "abc123", "jwt": "..."}
    G->>F: Response with Set-Cookie: jwt=...
    F->>C: Update UI (Bob added to player list)
```

### 5.2 Spiel starten

```mermaid
sequenceDiagram
    participant C as Client (Alice)
    participant F as Frontend
    participant G as API Gateway
    participant A as Auth
    participant L as Lobby Service
    participant GM as Game Service
    participant S as SSE Service
    participant DB as PostgreSQL

    C->>F: Click "Spiel starten"
    F->>G: POST /api/lobby/abc123/start (Cookie: jwt=...)
    G->>A: POST /internal/validate
    A->>G: {"valid": true, "user_id": "1", "username": "Alice"}
    G->>L: POST /lobby/abc123/start (Header: X-User-ID: 1)
    
    L->>DB: SELECT * FROM lobbies WHERE id='abc123'
    L->>DB: SELECT * FROM players WHERE lobby_id='abc123'
    
    Note over L: Validierung: Min. 2 Spieler, User ist Leader
    
    L->>DB: UPDATE lobbies SET status='running'
    L->>GM: POST /internal/game/create {"lobby_id": "abc123", "players": [...]}
    GM->>DB: INSERT games (lobby_id, current_player, turn_order)
    GM->>DB: INSERT dice_state (game_id, values, fixed, roll_count=0)
    GM->>L: {"game_id": "abc123", "first_player": "1"}
    
    L->>S: POST /internal/broadcast {"event": "game_started", "data": {"game_id": "abc123"}}
    S->>F: SSE Event: game_started
    
    L->>G: {"game_id": "abc123"}
    G->>F: Response
    F->>C: Redirect to /game/abc123
    
    Note over F,S: Frontend öffnet neue SSE-Connection
    F->>S: GET /events/game/abc123 (Cookie: jwt=...)
    S->>F: SSE-Stream established
```

### 5.3 Spielzug (Würfeln & Feld wählen)

```mermaid
sequenceDiagram
    participant C as Client (Alice)
    participant F as Frontend
    participant G as API Gateway
    participant A as Auth
    participant GM as Game Service
    participant S as SSE Service
    participant DB as PostgreSQL

    Note over C,DB: 1. Würfeln
    C->>F: Click "Würfeln"
    F->>G: POST /api/game/abc123/roll (Cookie: jwt=...)
    G->>A: POST /internal/validate
    A->>G: {"valid": true, "user_id": "1"}
    G->>GM: POST /game/abc123/roll (Header: X-User-ID: 1)
    
    GM->>DB: SELECT * FROM games WHERE id='abc123'
    Note over GM: Validierung: User ist aktueller Spieler, roll_count < 3
    
    Note over GM: Würfel-Logik (5 Zufallszahlen 1-6)
    GM->>DB: UPDATE dice_state SET values=[3,5,1,4,2], roll_count=1
    GM->>S: POST /internal/broadcast {"event": "dice_rolled", "data": {...}}
    S->>F: SSE Event: dice_rolled
    GM->>G: {"values": [3,5,1,4,2], "roll_count": 1}
    G->>F: Response
    F->>C: Animate Slot Machine, Show [3,5,1,4,2]
    
    Note over C,DB: 2. Würfel fixieren
    C->>F: Click on Dice #2 (value: 1)
    F->>G: POST /api/game/abc123/toggle/2
    G->>A: POST /internal/validate
    A->>G: {"valid": true, "user_id": "1"}
    G->>GM: POST /game/abc123/toggle/2
    GM->>DB: UPDATE dice_state SET fixed=[false,false,true,false,false]
    GM->>S: POST /internal/broadcast {"event": "dice_toggled", "data": {...}}
    S->>F: SSE Event: dice_toggled
    GM->>G: {"fixed": [false,false,true,false,false]}
    G->>F: Response
    F->>C: Show Dice #2 with border (fixed)
    
    Note over C,DB: 3. Nochmal würfeln (nicht-fixierte)
    C->>F: Click "Würfeln" (2. Mal)
    F->>G: POST /api/game/abc123/roll
    GM->>DB: UPDATE dice_state SET values=[3,5,1,6,2], roll_count=2
    Note over GM: Nur Dice 0,1,3,4 neu gewürfelt
    GM->>S: POST /internal/broadcast {"event": "dice_rolled", ...}
    S->>F: SSE Event: dice_rolled
    
    Note over C,DB: 4. Feld auswählen
    C->>F: Click on "Dreierpasch"
    F->>G: POST /api/game/abc123/select {"field": "three_of_a_kind"}
    G->>A: POST /internal/validate
    A->>G: {"valid": true, "user_id": "1"}
    G->>GM: POST /game/abc123/select {"field": "three_of_a_kind"}
    
    GM->>DB: SELECT * FROM scores WHERE game_id='abc123' AND player_id='1'
    Note over GM: Validierung: Feld noch nicht ausgefüllt
    Note over GM: Punkteberechnung: 3+5+1+6+2 = 17
    GM->>DB: INSERT scores (game_id, player_id, field, value=17)
    
    Note over GM: Nächster Spieler bestimmen
    GM->>DB: UPDATE games SET current_player='2'
    GM->>DB: UPDATE dice_state SET roll_count=0, fixed=[false,false,false,false,false]
    
    GM->>S: POST /internal/broadcast {"event": "field_selected", "data": {...}}
    GM->>S: POST /internal/broadcast {"event": "turn_changed", "data": {"player": "2"}}
    S->>F: SSE Events: field_selected, turn_changed
    GM->>G: {"next_player": "2"}
    G->>F: Response
    F->>C: Update ScoreBoard, Show "Bob's turn"
```

### 5.4 Timeout & Spielende

```mermaid
sequenceDiagram
    participant C as Client
    participant F as Frontend
    participant GM as Game Service
    participant S as SSE Service
    participant L as Lobby Service
    participant DB as PostgreSQL

    Note over GM: Timeout-Manager (läuft kontinuierlich)
    loop Every 1 second
        GM->>DB: SELECT games WHERE status='running'
        Note over GM: Check: last_action_at + 40s < now?
        alt Timeout detected
            GM->>DB: UPDATE games SET current_player=next_player
            GM->>S: POST /internal/broadcast {"event": "timeout", "data": {...}}
            GM->>S: POST /internal/broadcast {"event": "turn_changed", ...}
            S->>F: SSE Events
            F->>C: Show "Alice's turn skipped (timeout)"
        else 10 seconds remaining
            GM->>S: POST /internal/broadcast {"event": "timeout_warning", ...}
            S->>F: SSE Event
            F->>C: Show countdown in red
        end
    end
    
    Note over C,DB: Spielende (letztes Feld)
    C->>F: Alice wählt letztes Feld
    F->>GM: POST /api/game/abc123/select
    GM->>DB: INSERT scores (...)
    
    Note over GM: Check: Alle Spieler 13 Felder ausgefüllt?
    GM->>DB: SELECT COUNT(*) FROM scores WHERE game_id='abc123' GROUP BY player_id
    Note over GM: Ja! Spielende.
    
    Note over GM: Endpunkte berechnen
    GM->>DB: SELECT * FROM scores WHERE game_id='abc123'
    Note over GM: Calculate: Summe oben, Bonus, Gesamt
    
    GM->>DB: UPDATE games SET status='finished', end_time=NOW()
    GM->>S: POST /internal/broadcast {"event": "game_ended", "data": {"final_scores": [...]}}
    S->>F: SSE Event: game_ended
    F->>C: Redirect to /end/abc123
    
    Note over F: Frontend zeigt Endeseite mit Rangliste
```

---

## 6. Deployment-Architektur

### 6.1 Docker-Compose Übersicht

```yaml
version: '3.8'

services:
  # Reverse Proxy
  traefik:
    image: traefik:v2.10
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.yml:/traefik.yml
      - ./certs:/certs
    labels:
      - "traefik.enable=true"
  
  # Frontend
  frontend:
    build: ./frontend
    image: knuffel-frontend:latest
    labels:
      - "traefik.http.routers.frontend.rule=Host(`knuffel.uni.de`)"
      - "traefik.http.routers.frontend.entrypoints=websecure"
      - "traefik.http.routers.frontend.tls=true"
  
  # API Gateway
  gateway:
    build: ./gateway
    image: knuffel-gateway:latest
    environment:
      - AUTH_SERVICE_URL=http://auth:8080
      - LOBBY_SERVICE_URL=http://lobby:8080
      - GAME_SERVICE_URL=http://game:8080
    labels:
      - "traefik.http.routers.gateway.rule=Host(`api.knuffel.uni.de`)"
      - "traefik.http.routers.gateway.entrypoints=websecure"
      - "traefik.http.routers.gateway.tls=true"
  
  # Auth Service
  auth:
    build: ./auth
    image: knuffel-auth:latest
    environment:
      - JWT_SECRET=${JWT_SECRET}
  
  # Lobby Service
  lobby:
    build: ./lobby
    image: knuffel-lobby:latest
    environment:
      - DATABASE_URL=postgres://user:pass@postgres:5432/lobby_db
      - GAME_SERVICE_URL=http://game:8080
      - SSE_SERVICE_URL=http://sse:8080
      - AUTH_SERVICE_URL=http://auth:8080
    depends_on:
      - postgres
  
  # Game Service
  game:
    build: ./game
    image: knuffel-game:latest
    environment:
      - DATABASE_URL=postgres://user:pass@postgres:5432/game_db
      - SSE_SERVICE_URL=http://sse:8080
    depends_on:
      - postgres
  
  # SSE Service
  sse:
    build: ./sse
    image: knuffel-sse:latest
    labels:
      - "traefik.http.routers.sse.rule=Host(`events.knuffel.uni.de`)"
      - "traefik.http.routers.sse.entrypoints=websecure"
      - "traefik.http.routers.sse.tls=true"
  
  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=knuffel
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"

volumes:
  postgres_data:
```

### 6.2 Netzwerk-Topologie

```
┌────────────────────────────────────────────────────────────┐
│                     Uni Linux Server                        │
│                  (2 Cores, 4GB RAM)                         │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              Docker Bridge Network                    │ │
│  │                                                       │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────┐ │ │
│  │  │Frontend │  │ Gateway │  │  Lobby  │  │  Game  │ │ │
│  │  │  :80    │  │  :8080  │  │  :8080  │  │ :8080  │ │ │
│  │  └─────────┘  └─────────┘  └─────────┘  └────────┘ │ │
│  │                                                       │ │
│  │  ┌─────────┐  ┌─────────┐  ┌────────────────────┐  │ │
│  │  │  Auth   │  │   SSE   │  │    PostgreSQL      │  │ │
│  │  │  :8080  │  │  :8080  │  │      :5432         │  │ │
│  │  └─────────┘  └─────────┘  └────────────────────┘  │ │
│  │                                   │                  │ │
│  │                           Volume: postgres_data     │ │
│  └───────────────────────────────────────────────────── │ │
│                                                          │ │
│  ┌──────────────────────────────────────────────────────┐│
│  │                  Traefik (Ports 80/443)              ││
│  │  - SSL Termination                                   ││
│  │  - Subdomain Routing                                 ││
│  │  - Load Balancing (später)                           ││
│  └──────────────────────────────────────────────────────┘│
└────────────────────────────────────────────────────────────┘
                          │
                          │ Internet
                          ▼
                    ┌──────────┐
                    │  Browser │
                    └──────────┘
```

### 6.3 Resource-Allocation (geschätzt)

| Service | CPU | RAM | Replicas | Notes |
|---------|-----|-----|----------|-------|
| Frontend | 0.1 | 128MB | 1 | Nginx (statisch) |
| Gateway | 0.2 | 256MB | 1 | Routing + Auth-Checks |
| Auth | 0.1 | 128MB | 1 | Stateless, schnell |
| Lobby | 0.3 | 512MB | 1 | DB-Zugriffe |
| Game | 0.5 | 512MB | 1 | Spiellogik + Timer |
| SSE | 0.3 | 512MB | 1 | Connection-Management |
| PostgreSQL | 0.5 | 1GB | 1 | Datenbank |
| Traefik | 0.1 | 256MB | 1 | Reverse Proxy |
| **Total** | **2.1** | **3.3GB** | - | Passt auf Server |

**Spieler-Kapazität (grob geschätzt):**
- 100 parallele Lobbies
- 600 gleichzeitige Spieler (100 Lobbies × 6 Spieler)
- SSE-Connections: 600 (eine pro Spieler + Game)

---

## 7. Security & Resilience

### 7.1 Security-Maßnahmen

**JWT-Security:**
- HTTP-Only Cookies (kein JS-Zugriff)
- Secure Flag (nur HTTPS)
- SameSite=Strict (CSRF-Schutz)
- Token-Expiry: 24h
- HMAC-SHA256 Signierung
- Secret aus Environment-Variable (nicht im Code)

**API-Security:**
- CORS-Policy (nur eigene Domains)
- Rate-Limiting im API Gateway (10 req/sec pro User)
- Input-Validierung in jedem Service
- SQL-Prepared-Statements (gegen Injection)

**Network-Security:**
- Interne Services nicht von außen erreichbar
- Nur Traefik exposed (Ports 80/443)
- Service-zu-Service über Docker-Netzwerk (unverschlüsselt, da intern)
- PostgreSQL nicht exposed (nur intern)

**Secrets-Management:**
```bash
# .env (nicht in Git!)
JWT_SECRET=xxx...
DB_PASSWORD=yyy...
```

### 7.2 Error-Handling

**Service-Ausfälle:**
- Retry-Logik mit Exponential Backoff (max 3 Versuche)
- Timeout für Inter-Service-Calls (5 Sekunden)
- Graceful Degradation:
  - SSE-Service down → Frontend zeigt "Live-Updates unavailable"
  - Auth-Service down → Gateway cached alte JWT-Validierungen (5 Min)

**Database-Ausfälle:**
- Connection-Pool mit Retry
- Read-Replicas (Stretch-Goal)

**Logging:**
- Strukturiertes Logging (JSON)
- Log-Level: DEBUG (Dev), INFO (Prod)
- Centralized Logging (Stretch: ELK-Stack)

---

## 8. CI/CD Pipeline

### 8.1 GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy Knuffel

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Unit Tests
        run: |
          cd gateway && go test ./...
          cd ../auth && go test ./...
          cd ../lobby && go test ./...
          cd ../game && go test ./...
          cd ../sse && go test ./...
      
      - name: Run Integration Tests
        run: docker-compose -f docker-compose.test.yml up --abort-on-container-exit

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Docker Images
        run: |
          docker build -t knuffel-gateway:${{ github.sha }} ./gateway
          docker build -t knuffel-auth:${{ github.sha }} ./auth
          docker build -t knuffel-lobby:${{ github.sha }} ./lobby
          docker build -t knuffel-game:${{ github.sha }} ./game
          docker build -t knuffel-sse:${{ github.sha }} ./sse
          docker build -t knuffel-frontend:${{ github.sha }} ./frontend
      
      - name: Push to Registry
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          docker push knuffel-gateway:${{ github.sha }}
          # ... weitere Images

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Uni Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/knuffel
            git pull origin main
            docker-compose down
            docker-compose pull
            docker-compose up -d
            docker-compose ps
```

### 8.2 Deployment-Strategie

**Rolling Update:**
1. Pull neues Image
2. Stoppe alten Container
3. Starte neuen Container
4. Health-Check (HTTP /health Endpoint)
5. Bei Fehler: Rollback

**Zero-Downtime (Stretch):**
- Blue-Green Deployment
- 2 Instanzen pro Service
- Load-Balancer switched nach erfolgreicher Deploy

---

## 9. Monitoring & Observability (Stretch-Goal)

### 9.1 Metrics

**Service-Metriken:**
- Request-Count pro Endpoint
- Response-Time (p50, p95, p99)
- Error-Rate (4xx, 5xx)
- SSE-Connection-Count
- Database-Query-Time

**Business-Metriken:**
- Aktive Lobbies
- Aktive Spiele
- Durchschnittliche Spieldauer
- Reconnect-Rate

**Tools:**
- Prometheus (Metrics-Collection)
- Grafana (Dashboards)

### 9.2 Health-Checks

**Jeder Service exposed:**
```
GET /health
Response: {
  "status": "healthy",
  "uptime": 3600,
  "version": "1.0.0"
}
```

**Traefik Health-Check:**
```yaml
healthcheck:
  test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/health"]
  interval: 30s
  timeout: 5s
  retries: 3
```

---

## 10. Offene Punkte & Nächste Schritte

### 10.1 Für Implementierung benötigt

**Noch zu spezifizieren:**
- [ ] Detaillierte API-Endpunkt-Schemas (Request/Response)
- [ ] SSE-Event-Format (finales Schema)
- [ ] Datenbank-Tabellen-Design (lobby_db, game_db)
- [ ] Fehler-Codes & Error-Messages
- [ ] Frontend-Komponenten-Struktur
- [ ] Testing-Strategie (Unit/Integration/E2E)

**Technische Details:**
- [ ] Traefik-Konfiguration (traefik.yml)
- [ ] SSL-Zertifikate (Let's Encrypt Setup)
- [ ] Environment-Variables (.env.example)
- [ ] Database-Migrations (Flyway/Goose)
- [ ] Docker-Multistage-Builds (für kleinere Images)

### 10.2 Empfohlene Reihenfolge

**Phase 1: Core-Services (Woche 1-4)**
1. Auth Service (JWT-Operationen)
2. Lobby Service (Gast-Accounts + Lobbies)
3. API Gateway (Routing + Auth)
4. Frontend (Startseite + Lobby-Ansicht)
5. PostgreSQL (lobby_db Schema)

**Phase 2: Game-Logic (Woche 5-8)**
6. Game Service (Würfeln + Punkteberechnung)
7. Frontend (Spielseite + ScoreBoard)
8. PostgreSQL (game_db Schema)

**Phase 3: Real-Time (Woche 9-12)**
9. SSE Service (Event-Broadcasting)
10. Frontend (SSE-Integration)
11. Timeout-Mechanismus

**Phase 4: Polish (Woche 13-16)**
12. Endeseite + Rangliste
13. Testing + Bug-Fixes
14. Deployment + CI/CD
15. Dokumentation

---

## 11. Zusammenfassung

### 11.1 Architektur-Prinzipien

✅ **Microservices-Konformität:**
- Services nach Bounded Contexts getrennt
- Lose Kopplung (REST-APIs)
- Hohe Kohäsion (Service = eine Verantwortlichkeit)
- Service-eigene Datenbanken

✅ **Folien-Konformität:**
- REST-Architektur (Folie 06_REST)
- SOA-Prinzipien (Folie 05_SOA)
- MVC/MVP-Trennung (Frontend = View, Services = Controller+Model)
- API Gateway Pattern

✅ **Best Practices:**
- Stateless Services (außer SSE)
- JWT-Auth mit HTTP-Only Cookies
- Zentrale Validierung (API Gateway)
- Event-Driven-Kommunikation (SSE)

### 11.2 Key-Decisions

| Entscheidung | Begründung |
|--------------|------------|
| **SSE statt WebSockets** | Passt zu REST-Architektur, einfacher Auth-Flow |
| **Zentraler SSE Service** | Bessere Service-Trennung, folgt Folie-Struktur |
| **API Gateway mit Auth** | Single Entry Point, zentrale Security |
| **Subdomain-Routing** | Klare Service-Trennung, professioneller |
| **2 PostgreSQL DBs** | Service-Isolation, unabhängiges Scaling |
| **Kein Persistence Service** | Vermeidet Shared-Database Anti-Pattern |
| **Kein Würfel Service** | Zu feinkörnig, gehört zur Game-Logik |

---

**Dokument-Status:** ✅ Architektur finalisiert  
**Nächster Schritt:** API-Spezifikation oder Datenbank-Schema  
**Version:** 1.0
