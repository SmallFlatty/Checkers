# Checkers 2.0

An online checkers (draughts) game with real-time multiplayer, an in-room chat, a single-player bot, a leaderboard and player reviews.

Built as a full-stack project: **Spring Boot** REST backend + **Vue 3** SPA, with **Ably** as the real-time pub/sub transport and **PostgreSQL** for persistence.

---

## Table of contents

- [Features](#features)
- [Tech stack](#tech-stack)
- [How it works](#how-it-works)
- [Project structure](#project-structure)
- [Getting started](#getting-started)
- [Configuration](#configuration)
- [API reference](#api-reference)
- [Real-time channels](#real-time-channels)
- [Database schema](#database-schema)
- [Implemented game rules](#implemented-game-rules)
- [Known limitations](#known-limitations)
- [Roadmap](#roadmap)

---

## Features

| Feature | Description |
|---|---|
| **Play with a friend** | Create a room, share the 6-digit code, play in real time. Moves are validated server-side. |
| **Play with a bot** | Fully offline single-player mode against a bot that prioritises captures. |
| **In-room chat** | Text chat inside the game room, delivered over the same real-time channel as the moves. |
| **Automatic colour assignment** | The first player to enter the room gets white, the second gets black — resolved through Ably presence. |
| **Leaderboard** | Wins are aggregated per player name and ranked. |
| **Ratings & comments** | After a win, the player can leave a 1–5 rating and a comment; the average rating is shown on the home page. |
| **Room lifecycle** | Rooms are created with a random unique 6-digit ID and deleted together with their player records when the game ends. |

---

## Tech stack

**Backend**
- Java 23
- Spring Boot 3.4.4 (Web, Data JPA, WebSocket starter)
- Hibernate / Jakarta Persistence 3.1
- PostgreSQL
- Ably Java SDK 1.2.12
- Maven (with wrapper)

**Frontend**
- Vue 3 (Options API, Vue CLI 5)
- Vue Router 4
- Vuex 4
- Axios
- Ably JS SDK 2.x

---

## How it works

The two game modes are deliberately built differently.

**Bot mode (`/game-with-bot`)** runs entirely in the browser. The board, the move generator, the forced-capture logic and the bot itself all live in `GameWithBot.vue`. No network calls, no persistence — the fastest possible path to a playable game.

**Multiplayer mode (`/game-room/:roomId`)** splits responsibilities:

```
Player A (Vue)                Spring Boot                 Player B (Vue)
     |                             |                            |
     |  POST /api/game/move        |                            |
     |---------------------------->|                            |
     |                             | validate against the       |
     |                             | in-memory board for gameId |
     |                             |                            |
     |   { valid, captured,        |  publish "move" + "turn"   |
     |     becameQueen, nextTurn } |  to game-room-{roomId}     |
     |<----------------------------|--------------------------->|
     |                                          (via Ably)      |
```

The server is the single source of truth for the board: it owns a `Map<String, Piece[][]>` keyed by room ID and rejects illegal moves, out-of-turn moves and moves that ignore a mandatory capture. Ably is used purely as a transport for broadcasting the already-validated result — the client never trusts another client's move directly.

Board state is held **in memory only**, so a backend restart resets all active games. Only the results (wins, ratings, comments, rooms) go to PostgreSQL.

---

## Project structure

```
Checkers/
├── src/main/java/com/example/demo/
│   ├── DemoApplication.java          # Spring Boot entry point
│   ├── RoomLogic/                    # Multiplayer game engine
│   │   ├── GameController.java       # /api/game — state, move, reset + Ably publish
│   │   ├── GameService.java          # Board state, move validation, win detection
│   │   ├── Piece.java                # Colour + queen flag
│   │   ├── Move.java                 # startX/startY → endX/endY
│   │   ├── MoveRequest.java          # Incoming move payload
│   │   └── CheckerDto.java           # Board cell sent to the client
│   ├── controller/                   # REST controllers (rooms, scores, rate, comments)
│   ├── service/                      # Service interfaces + JPA implementations
│   ├── repository/                   # Spring Data JPA repositories
│   ├── model/                        # JPA entities
│   ├── GameState/GameState.java      # PLAYING / WON / DRAW enum
│   └── Config/                       # CORS configuration
├── src/main/resources/
│   └── application.properties
└── frontend/Checkers/
    └── src/
        ├── views/
        │   ├── HomePage.vue          # Menu, rules, average rating
        │   ├── GameWithFriend.vue    # Create / join a room
        │   ├── GameRoom.vue          # Multiplayer board, chat, end-game panel
        │   ├── GameWithBot.vue       # Single-player board + bot
        │   ├── LeaderBoard.vue       # Win ranking
        │   └── CommentBoard.vue      # Player comments
        ├── router/index.js
        └── assets/                   # Board and piece sprites
```

---

## Getting started

### Prerequisites

- JDK 23
- Node.js 16+ and npm
- PostgreSQL 14+ running locally
- An [Ably](https://ably.com) account (the free tier is enough)

### 1. Database

```sql
CREATE DATABASE postgres;  -- or use your existing one
```

Tables are generated automatically by Hibernate (`spring.jpa.hibernate.ddl-auto=update`) on first start.

### 2. Backend

```bash
./mvnw spring-boot:run          # Linux / macOS
mvnw.cmd spring-boot:run        # Windows
```

The API starts on **http://localhost:7000**.

### 3. Frontend

```bash
cd frontend/Checkers
npm install
npm run serve
```

The SPA starts on **http://localhost:8080**.

To play a multiplayer game locally, open the app in two browser windows (use an incognito window for the second player so the sessions stay separate), create a room in one and join it by code in the other.

---

## Configuration

`src/main/resources/application.properties`:

```properties
server.port=7000
spring.datasource.url=jdbc:postgresql://localhost:5432/postgres
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}
spring.jpa.hibernate.ddl-auto=update
ably.api.key=${ABLY_API_KEY}
```

The frontend reads its own Ably key from `frontend/Checkers/.env.local`:

```
VUE_APP_ABLY_KEY=your-ably-key
VUE_APP_API_URL=http://localhost:7000
```

> **Note:** the Ably key and the database credentials must be supplied through environment variables and never committed. Both `.env.local` and any local properties override belong in `.gitignore`.

---

## API reference

All endpoints are prefixed with `http://localhost:7000`. CORS is open to all origins in development.

### Game

| Method | Endpoint | Parameters | Description |
|---|---|---|---|
| `GET` | `/api/game/state` | `gameId` | Returns the current board (`checkers[]`) and whose turn it is. Creates a fresh board if the room has none. |
| `POST` | `/api/game/move` | JSON body | Validates a move, mutates the board, broadcasts `move` and `turn` over Ably. |
| `GET` | `/api/game/reset` | `gameId` | Resets the board and sets the turn back to white. |

<details>
<summary>Move request / response payloads</summary>

**Request**

```json
{
  "gameId": "123456",
  "playerName": "Alice",
  "playerColor": "white",
  "move": { "startX": 5, "startY": 0, "endX": 4, "endY": 1 }
}
```

**Response**

```json
{
  "valid": true,
  "becameQueen": false,
  "captured": [4, 1],
  "error": null,
  "nextTurn": "black",
  "type": "win",
  "winner": "white"
}
```

`type` and `winner` are only present when the game has ended. `captured` holds the `[row, col]` of the removed piece, or `null`.

</details>

### Rooms

| Method | Endpoint | Parameters | Description |
|---|---|---|---|
| `POST` | `/api/rooms/create` | JSON body | Creates a room with a unique random 6-digit ID. |
| `GET` | `/api/rooms/idRooms` | — | Lists all existing rooms (used to validate a join code). |
| `POST` | `/api/roomservice/deleteroom` | `roomId` | Deletes the room and its player-name record. |

### Players in a room

| Method | Endpoint | Parameters | Description |
|---|---|---|---|
| `POST` | `/api/users/writefirstnick` | `roomId`, `firstNick`, `secondNick` | Registers the room creator. The second slot is stored as a placeholder. |
| `POST` | `/api/users/writesecondplayer` | `roomId`, `secondNick` | Fills the second slot; fails if it is already taken. |
| `GET` | `/api/users/findnicknamesbyroomid` | `roomId` | Returns both nicknames for the room. |

### Scores, ratings, comments

| Method | Endpoint | Parameters | Description |
|---|---|---|---|
| `POST` | `/api/scores/add` | `playerName`, `type` | Records a game outcome (`win` / `lose`). |
| `GET` | `/api/scores/leaderboard` | — | Win counts per player, ordered descending. |
| `POST` | `/api/results/save` | `player1`, `type` | Alternative result-saving endpoint. |
| `GET` | `/api/results/all` | — | All stored results. |
| `POST` | `/api/rate/add` | JSON body | Saves a 1–5 rating. |
| `GET` | `/api/rate/average` | — | Average rating across all entries. |
| `POST` | `/api/comment/addcomment` | JSON body | Saves a comment with a timestamp. |
| `GET` | `/api/comment/getallcomment` | — | All comments. |

---

## Real-time channels

Every room uses a single Ably channel named `game-room-{roomId}`.

| Event | Published by | Payload | Purpose |
|---|---|---|---|
| `move` | Backend | `move`, `playerName`, `becameQueen`, `captured` | Tells the other client to re-render the board. |
| `turn` | Backend | `nextTurn` | Hands the turn over (or keeps it, during a capture chain). |
| `chat-message` | Client | `message`, `from` | In-room chat. |
| `game-over` | Client | `winner` | Triggers the end-game panel. |

**Presence** is used for colour assignment: on entering, a client checks how many other members are already present — an empty room means white, otherwise black.

---

## Database schema

| Table | Columns | Notes |
|---|---|---|
| `rooms` | `id` (PK, 6-digit), `created_at` | ID generated in `GameRoomService`, not by the DB. |
| `playernamesrooms` | `roomid` (PK), `playerwhite`, `playerblack` | One row per room; `playerblack = '1'` marks a free slot. |
| `leaderboard` | `id_player` (PK), `playername`, `type` | `type` is `win` or `lose`; the leaderboard counts `win` rows. |
| `rate` | `id_player` (PK), `playername`, `rate` | Rating from 1 to 5. |
| `comment` | `id_player` (PK), `playername`, `comment`, `data` | `data` is the creation timestamp. |

---

## Implemented game rules

- 8×8 board, 12 pieces per side, white moves first.
- Men move one square diagonally forward; they capture in **all four** directions.
- A man reaching the far rank is promoted to a queen. Promotion ends the turn.
- Queens move and capture any distance along a diagonal, provided the path is clear and the landing square is empty.
- **Captures are mandatory** — a move that ignores an available capture is rejected with `"Must capture!"`.
- **Chain captures**: if another capture is available from the landing square, the same player keeps the turn.
- The game ends when a side has no pieces left, has no legal moves, or is reduced to a single piece against three or more. If neither side can move, the result is a draw.

---

## Known limitations

These are known and intentional trade-offs of the current version, listed for transparency:

- **Board state is in-memory.** A backend restart wipes every game in progress. Persisting boards (or moving to Redis) is the natural next step.
- **`GameResult` and `ScoreEntity` map to the same `leaderboard` table** with different ID columns. One of them should be removed.
- **No authentication.** Players are identified by a nickname typed into an input field; nothing prevents impersonation.
- **The bot plays randomly.** It prioritises captures and follows chains, but picks among the available options at random — there is no evaluation function or lookahead.
- **Colour assignment is race-prone.** Two players entering a room at the same instant can both read an empty presence set.
- **`node_modules` is committed to the repository**, which inflates the clone size considerably.
- **The frontend holds an Ably key with publish rights.** A token-auth endpoint on the backend would be the correct approach.
- The `spring-boot-starter-websocket` dependency is declared but unused — the real-time layer is entirely Ably.

---

## Roadmap

- [ ] Move Ably authentication to backend-issued tokens
- [ ] Persist board state so games survive a restart
- [ ] Give the bot a minimax evaluation with selectable difficulty
- [ ] Player accounts and a per-player match history
- [ ] Reconnect handling — currently a dropped connection ends the game
- [ ] Dockerise the backend, frontend and database
- [ ] Unit tests for `GameService` (the move validator is the highest-value target)

---

## License

No license specified.
