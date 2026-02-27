# Snake.io MMO - System Architecture

> **Reference Document** for implementing a Snake.io MMO clone.
> Combines snake.io game design (mass-consuming boost, pellet system, circular map)
> with slither.io network architecture (WebSocket, server authority, spatial hash).

---

## Table of Contents

1. [Directory Structure](#1-directory-structure)
2. [Server Architecture](#2-server-architecture)
3. [Client Architecture](#3-client-architecture)
4. [Network Protocol](#4-network-protocol)
5. [Game Physics](#5-game-physics)
6. [Data Flow Diagrams](#6-data-flow-diagrams)
7. [Configuration Constants](#7-configuration-constants)
8. [Scalability](#8-scalability)

---

## 1. Directory Structure

Monorepo layout with `client/` (Next.js + TypeScript) and `server/` (Go).

```
snake-io-mmo/
├── docs/
│   ├── ARCHITECTURE.md          # This document
│   └── SNAKEIO_ANALYSIS.md      # Game mechanics analysis
│
├── client/                      # Next.js + TypeScript frontend
│   ├── package.json
│   ├── tsconfig.json
│   ├── next.config.js
│   ├── public/
│   │   ├── favicon.ico
│   │   └── assets/
│   │       ├── skins/           # Snake skin sprite sheets
│   │       └── sounds/          # SFX (boost, eat, death)
│   │
│   ├── src/
│   │   ├── app/                 # Next.js App Router
│   │   │   ├── layout.tsx       # Root layout
│   │   │   ├── page.tsx         # Landing / lobby page
│   │   │   └── game/
│   │   │       └── page.tsx     # Game page (canvas mount point)
│   │   │
│   │   ├── game/                # Game engine (non-React, pure TS)
│   │   │   ├── Engine.ts        # Main game loop (rAF, delta time)
│   │   │   ├── Renderer.ts      # Canvas 2D rendering pipeline
│   │   │   ├── Camera.ts        # Viewport, follow, zoom
│   │   │   ├── InputHandler.ts  # Mouse/touch → angle + boost
│   │   │   ├── StateBuffer.ts   # Interpolation buffer (2-state)
│   │   │   └── types.ts         # Shared game types (Snake, Pellet, etc.)
│   │   │
│   │   ├── network/             # Networking layer
│   │   │   ├── WebSocketClient.ts   # WS connection, reconnection
│   │   │   ├── MessageHandler.ts    # Incoming message routing
│   │   │   └── Protocol.ts         # Message type definitions
│   │   │
│   │   ├── ui/                  # React overlay components
│   │   │   ├── Leaderboard.tsx  # Top-10 leaderboard
│   │   │   ├── Minimap.tsx      # Corner minimap
│   │   │   ├── ScoreDisplay.tsx # Current score / length
│   │   │   ├── DeathScreen.tsx  # Death overlay + respawn
│   │   │   ├── BoostIndicator.tsx # Boost meter / indicator
│   │   │   ├── NameInput.tsx    # Pre-game name entry
│   │   │   └── HUD.tsx         # HUD container component
│   │   │
│   │   ├── hooks/               # React hooks
│   │   │   ├── useGameEngine.ts # Engine lifecycle hook
│   │   │   └── useWebSocket.ts  # WebSocket connection hook
│   │   │
│   │   └── utils/               # Shared utilities
│   │       ├── math.ts          # Vec2, lerp, clamp, angleDiff
│   │       └── constants.ts     # Client-side constants
│   │
│   └── styles/
│       └── globals.css          # Global styles
│
├── server/                      # Go backend
│   ├── go.mod
│   ├── go.sum
│   ├── main.go                  # Entry point, HTTP + WS server
│   │
│   ├── internal/
│   │   ├── game/                # Core game logic
│   │   │   ├── room.go          # Room struct, game loop, tick
│   │   │   ├── room_manager.go  # Room lifecycle, player assignment
│   │   │   ├── snake.go         # Snake data model + movement
│   │   │   ├── pellet.go        # Pellet types + spawning
│   │   │   ├── collision.go     # Collision detection logic
│   │   │   ├── spatial_hash.go  # Spatial hash grid
│   │   │   ├── bot.go           # Bot AI state machine
│   │   │   ├── leaderboard.go   # Leaderboard tracking
│   │   │   └── physics.go       # Movement physics, angle steering
│   │   │
│   │   ├── network/             # Networking
│   │   │   ├── handler.go       # WebSocket handler, upgrade
│   │   │   ├── client.go        # Client connection struct
│   │   │   ├── message.go       # Message types + serialization
│   │   │   └── interest.go      # Viewport interest management
│   │   │
│   │   ├── config/              # Configuration
│   │   │   └── constants.go     # All game constants
│   │   │
│   │   └── util/                # Utilities
│   │       ├── vec2.go          # Vec2 math operations
│   │       └── id.go            # ID generation
│   │
│   └── cmd/                     # CLI tools (optional)
│       └── loadtest/
│           └── main.go          # Load testing client
│
├── .gitignore
└── README.md
```

---

## 2. Server Architecture

### 2.1 Game Loop

The server runs a **fixed-timestep game loop at 30 TPS** (ticks per second). Each tick has a budget of **33ms**.

#### Tick Budget Breakdown

| Phase                  | Budget  | Description                                    |
|------------------------|---------|------------------------------------------------|
| Input Processing       | ~2ms    | Drain input queue, apply player inputs          |
| Physics / Movement     | ~5ms    | Angle steering, position updates, body follow   |
| Collision Detection    | ~8ms    | Spatial hash queries, head-body/wall/pellet     |
| State Serialization    | ~5ms    | Build per-player viewport-filtered state        |
| Network Send           | ~5ms    | Write state messages to WebSocket connections   |
| Overhead / Slack       | ~8ms    | GC pauses, scheduling jitter, buffer            |
| **Total**              | **33ms**|                                                 |

#### Game Loop Implementation (Go)

```go
func (r *Room) Run() {
    ticker := time.NewTicker(time.Duration(TickInterval) * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            r.tick()
        case <-r.stopCh:
            return
        }
    }
}

func (r *Room) tick() {
    r.currentTick++
    dt := float64(TickInterval) / 1000.0 // 0.033s

    // 1. Drain input queue
    r.processInputs()

    // 2. Move all snakes
    r.updatePhysics(dt)

    // 3. Check collisions
    r.checkCollisions()

    // 4. Process deaths and growth
    r.processDeaths()
    r.processGrowth()

    // 5. Respawn natural pellets
    r.respawnPellets()

    // 6. Update spatial hash
    r.rebuildSpatialHash()

    // 7. Send state to clients
    r.broadcastState()

    // 8. Periodic leaderboard update
    if r.currentTick%LeaderboardInterval == 0 {
        r.broadcastLeaderboard()
    }
}
```

### 2.2 Room Manager

The **Room Manager** handles room lifecycle and player assignment.

```go
type RoomManager struct {
    mu       sync.RWMutex
    rooms    map[string]*Room
    maxRooms int
}
```

**Responsibilities:**

- **Room Creation**: Create a new room when all existing rooms are full or above a soft cap.
- **Player Assignment**: Assign connecting players to the room with the most available slots (to keep rooms populated).
- **Bot Filling**: Each room maintains a minimum population. Bots fill slots to reach `TargetEntitiesPerRoom` (50). As real players join, bots are removed. As players leave, bots are added.
- **Room Destruction**: When a room has zero human players for a configurable timeout (e.g., 60 seconds), destroy it and free resources.
- **Max Players**: Each room supports up to `MaxPlayersPerRoom` (50) entities (humans + bots combined).

```go
func (rm *RoomManager) AssignPlayer(client *Client) *Room {
    rm.mu.Lock()
    defer rm.mu.Unlock()

    // Find room with available space, prefer most populated
    var bestRoom *Room
    for _, room := range rm.rooms {
        if room.HumanCount() < MaxPlayersPerRoom {
            if bestRoom == nil || room.HumanCount() > bestRoom.HumanCount() {
                bestRoom = room
            }
        }
    }

    // Create new room if none available
    if bestRoom == nil {
        bestRoom = NewRoom(rm.generateRoomID())
        rm.rooms[bestRoom.ID] = bestRoom
        go bestRoom.Run()
    }

    bestRoom.AddPlayer(client)
    bestRoom.AdjustBots() // Remove a bot to make room
    return bestRoom
}
```

### 2.3 Snake Data Model

```go
type Snake struct {
    ID          string
    Name        string
    Skin        int
    Segments    []Vec2    // Head is Segments[0], tail is last element
    Angle       float64   // Current facing angle (radians)
    TargetAngle float64   // Desired angle from client input
    Speed       float64   // Current speed (units/s)
    Score       int
    Boosting    bool
    Alive       bool
    IsBot       bool
    LastInput   time.Time // For timeout detection
}
```

**Segments** are a series of `Vec2` points. The head is always `Segments[0]`. Each segment follows the one ahead of it, maintaining a spacing of `SegmentSpacing` (10 units).

```go
type Vec2 struct {
    X float64
    Y float64
}
```

**Initial state** when a snake spawns:
- Score: `InitialScore` (10)
- Segments: `InitialSegments` (10) segments, spaced `SegmentSpacing` apart, extending behind the head
- Speed: `BaseSpeed` (200 units/s)
- Position: Random position within the map, at least 500 units from the wall

### 2.4 Pellet System

Three types of pellets exist in the game:

| Type            | Spawn Condition                        | Value Range | Color         |
|-----------------|----------------------------------------|-------------|---------------|
| Natural         | Random, maintain density of ~1000/room | 1-5         | Random bright |
| Death Drop      | Snake dies, proportional to mass       | 5           | Snake's color |
| Boost Trail     | Behind boosting snake, every 3 ticks   | 1           | Snake's color |

```go
type Pellet struct {
    ID    uint32
    X     float64
    Y     float64
    Value int
    Color int // Color index
}
```

**Natural Pellet Spawning:**
```go
func (r *Room) respawnPellets() {
    deficit := NaturalPelletCount - len(r.naturalPellets)
    for i := 0; i < deficit; i++ {
        angle := rand.Float64() * 2 * math.Pi
        dist := rand.Float64() * (MapRadius - 100) // Keep away from wall
        pellet := &Pellet{
            ID:    r.nextPelletID(),
            X:     dist * math.Cos(angle),
            Y:     dist * math.Sin(angle),
            Value: NaturalPelletMinValue + rand.Intn(NaturalPelletMaxValue-NaturalPelletMinValue+1),
            Color: rand.Intn(8),
        }
        r.pellets[pellet.ID] = pellet
        r.spatialHash.Insert(pellet)
    }
}
```

**Death Drop Pellets:**
```go
func (r *Room) dropDeathPellets(snake *Snake) {
    totalValue := int(float64(snake.Score) * DeathDropRatio) // 80% of score
    numPellets := totalValue / DeathPelletValue

    for i := 0; i < numPellets; i++ {
        // Scatter around the snake's body
        segIdx := rand.Intn(len(snake.Segments))
        seg := snake.Segments[segIdx]
        offset := Vec2{
            X: (rand.Float64() - 0.5) * 60,
            Y: (rand.Float64() - 0.5) * 60,
        }
        pellet := &Pellet{
            ID:    r.nextPelletID(),
            X:     seg.X + offset.X,
            Y:     seg.Y + offset.Y,
            Value: DeathPelletValue,
            Color: snake.Skin,
        }
        r.pellets[pellet.ID] = pellet
        r.spatialHash.Insert(pellet)
    }
}
```

### 2.5 Spatial Hash Grid

A spatial hash grid provides **O(1) average-case** entity lookups for collision detection and interest management.

```go
type SpatialHash struct {
    CellSize float64
    Cells    map[CellKey][]Entity // Entity can be *Snake segment or *Pellet
}

type CellKey struct {
    X int
    Y int
}

type Entity interface {
    GetPosition() Vec2
    GetID() string
}
```

**Cell size: 200x200 units.** This means:
- Map radius 4000 → grid spans roughly 40x40 cells
- Each cell is large enough to contain the max collision check radius

**Operations:**

```go
// Insert an entity into the grid
func (sh *SpatialHash) Insert(pos Vec2, entity Entity) {
    key := CellKey{
        X: int(math.Floor(pos.X / sh.CellSize)),
        Y: int(math.Floor(pos.Y / sh.CellSize)),
    }
    sh.Cells[key] = append(sh.Cells[key], entity)
}

// Remove an entity from the grid
func (sh *SpatialHash) Remove(pos Vec2, entity Entity) {
    key := CellKey{
        X: int(math.Floor(pos.X / sh.CellSize)),
        Y: int(math.Floor(pos.Y / sh.CellSize)),
    }
    entities := sh.Cells[key]
    for i, e := range entities {
        if e.GetID() == entity.GetID() {
            sh.Cells[key] = append(entities[:i], entities[i+1:]...)
            break
        }
    }
}

// Query all entities within a rectangular region
func (sh *SpatialHash) Query(minX, minY, maxX, maxY float64) []Entity {
    var results []Entity
    startX := int(math.Floor(minX / sh.CellSize))
    startY := int(math.Floor(minY / sh.CellSize))
    endX := int(math.Floor(maxX / sh.CellSize))
    endY := int(math.Floor(maxY / sh.CellSize))

    for x := startX; x <= endX; x++ {
        for y := startY; y <= endY; y++ {
            key := CellKey{X: x, Y: y}
            results = append(results, sh.Cells[key]...)
        }
    }
    return results
}
```

### 2.6 Interest Management

Only entities within the player's **viewport + margin** are sent in state updates. This dramatically reduces bandwidth.

```go
func (r *Room) getVisibleEntities(player *Snake) (snakes []*Snake, pellets []*Pellet) {
    head := player.Segments[0]
    zoom := calculateZoom(player.Score)

    // Viewport dimensions scaled by zoom and interest margin
    halfW := (ViewportWidth / zoom) * InterestMargin / 2
    halfH := (ViewportHeight / zoom) * InterestMargin / 2

    minX := head.X - halfW
    maxX := head.X + halfW
    minY := head.Y - halfH
    maxY := head.Y + halfH

    // Query spatial hash for entities in viewport
    entities := r.spatialHash.Query(minX, minY, maxX, maxY)

    for _, e := range entities {
        switch v := e.(type) {
        case *SnakeSegmentRef:
            // Add the full snake if any segment is visible
            snakes = appendUnique(snakes, v.Snake)
        case *Pellet:
            pellets = append(pellets, v)
        }
    }
    return
}
```

**Viewport calculation:**
- Base viewport: 1920x1080 (reference resolution)
- Zoom factor scales the effective viewport (zoomed out = larger viewport = more entities)
- Interest margin: 1.5x viewport in each direction (to prevent pop-in)

### 2.7 Bot AI

Bots use a simple **state machine** with four states:

```go
type BotState int

const (
    BotWander BotState = iota  // Random wandering
    BotChase                    // Moving toward nearby pellets
    BotFlee                     // Running from larger snakes
    BotHunt                     // Pursuing smaller snakes
)

type BotAI struct {
    Snake       *Snake
    State       BotState
    TargetPos   Vec2
    StateTimer  float64
}
```

**State Transitions:**

```
                  ┌──────────────┐
                  │    WANDER    │
                  └──────┬───────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
        ┌──────────┐ ┌────────┐ ┌──────┐
        │  CHASE   │ │  FLEE  │ │ HUNT │
        │ (pellets)│ │(bigger)│ │(small)│
        └──────────┘ └────────┘ └──────┘
```

**Decision Logic (evaluated every 10 ticks):**

```go
func (bot *BotAI) evaluate(room *Room) {
    head := bot.Snake.Segments[0]
    nearbySnakes := room.spatialHash.QuerySnakes(head, 500) // 500 unit radius

    // Priority 1: Flee from larger snakes nearby
    for _, other := range nearbySnakes {
        if other.Score > bot.Snake.Score*1.5 && distance(head, other.Segments[0]) < 300 {
            bot.State = BotFlee
            bot.TargetPos = oppositeDirection(head, other.Segments[0])
            return
        }
    }

    // Priority 2: Hunt smaller snakes if we're big enough
    if bot.Snake.Score > 200 {
        for _, other := range nearbySnakes {
            if other.Score < bot.Snake.Score*0.5 && distance(head, other.Segments[0]) < 400 {
                bot.State = BotHunt
                bot.TargetPos = other.Segments[0]
                return
            }
        }
    }

    // Priority 3: Chase nearby pellets
    nearbyPellets := room.spatialHash.QueryPellets(head, 300)
    if len(nearbyPellets) > 0 {
        best := findHighestValuePellet(nearbyPellets)
        bot.State = BotChase
        bot.TargetPos = Vec2{best.X, best.Y}
        return
    }

    // Default: Wander
    bot.State = BotWander
    if bot.StateTimer <= 0 {
        bot.TargetPos = randomPointInMap(MapRadius * 0.8)
        bot.StateTimer = 3.0 // Wander for 3 seconds before re-evaluating
    }
}
```

**Steering:** Bots set `TargetAngle` toward `TargetPos` using `atan2`. The same physics system handles their movement (angle steering + speed).

### 2.8 WebSocket Handler

Uses `gorilla/websocket` for WebSocket support.

```go
func (h *Handler) ServeWS(w http.ResponseWriter, r *http.Request) {
    conn, err := h.upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println("Upgrade error:", err)
        return
    }

    client := &Client{
        Conn:    conn,
        Send:    make(chan []byte, 256),
        Handler: h,
    }

    // Start read/write pumps
    go client.readPump()
    go client.writePump()
}
```

**Connection Lifecycle:**

1. Client connects via WebSocket at `/ws`
2. Server creates `Client` struct with send channel
3. `readPump` goroutine reads incoming messages (input, join, respawn)
4. `writePump` goroutine writes outgoing messages from send channel
5. On disconnect: remove from room, convert snake to death pellets if alive

**Message Routing:**

```go
func (c *Client) handleMessage(data []byte) {
    var msg map[string]interface{}
    json.Unmarshal(data, &msg)

    switch msg["type"] {
    case "join":
        c.handleJoin(msg)
    case "input":
        c.handleInput(msg)
    case "respawn":
        c.handleRespawn(msg)
    }
}
```

### 2.9 Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                    HTTP Server                       │
│              (accepts WS upgrades)                   │
└─────────┬───────────────────────────────┬───────────┘
          │                               │
    ┌─────▼─────┐                   ┌─────▼─────┐
    │  Client    │                   │  Client    │
    │ readPump   │                   │ readPump   │
    │ (goroutine)│                   │ (goroutine)│
    └─────┬──────┘                   └─────┬──────┘
          │ input channel                  │ input channel
          ▼                                ▼
    ┌─────────────────────────────────────────────────┐
    │              Room Game Loop                      │
    │          (single goroutine per room)              │
    │                                                  │
    │  1. Drain input queue (channel read)             │
    │  2. Process physics (single-threaded)            │
    │  3. Check collisions (single-threaded)           │
    │  4. Serialize state                              │
    │  5. Send to client write channels                │
    └─────────────────────────────────────────────────┘
          │ send channel                   │ send channel
          ▼                                ▼
    ┌─────────────┐                  ┌─────────────┐
    │  Client      │                  │  Client      │
    │  writePump   │                  │  writePump   │
    │  (goroutine) │                  │  (goroutine) │
    └──────────────┘                  └──────────────┘
```

**Key design decisions:**
- **Single goroutine per room**: All game logic for a room runs in one goroutine, avoiding locks on game state.
- **Channel-based input queue**: Each client's `readPump` sends input events to the room's input channel. The game loop drains this channel at the start of each tick.
- **Mutex for room state read**: Only needed if external goroutines need to read room state (e.g., room manager querying player counts). The game loop goroutine owns all writes.
- **Buffered send channels**: Each client has a buffered send channel (256 messages). If the buffer fills (slow client), the connection is dropped.

---

## 3. Client Architecture

### 3.1 Engine

The game engine runs a `requestAnimationFrame` loop, independent of React's render cycle.

```typescript
class Engine {
  private lastTime: number = 0;
  private running: boolean = false;
  private renderer: Renderer;
  private camera: Camera;
  private stateBuffer: StateBuffer;
  private inputHandler: InputHandler;

  start(): void {
    this.running = true;
    this.lastTime = performance.now();
    requestAnimationFrame(this.loop.bind(this));
  }

  private loop(now: number): void {
    if (!this.running) return;

    const dt = (now - this.lastTime) / 1000; // seconds
    this.lastTime = now;

    this.update(dt);
    this.render();

    requestAnimationFrame(this.loop.bind(this));
  }

  private update(dt: number): void {
    this.inputHandler.update();
    this.stateBuffer.update(dt);
    this.camera.update(dt);
  }

  private render(): void {
    this.renderer.render(
      this.stateBuffer.getInterpolatedState(),
      this.camera
    );
  }
}
```

### 3.2 Renderer

The renderer uses **Canvas 2D** with a layered drawing pipeline.

```typescript
class Renderer {
  private ctx: CanvasRenderingContext2D;
  private canvas: HTMLCanvasElement;

  render(state: InterpolatedState, camera: Camera): void {
    const ctx = this.ctx;
    const { width, height } = this.canvas;

    // 1. Clear
    ctx.clearRect(0, 0, width, height);

    // 2. Apply camera transform
    ctx.save();
    ctx.translate(width / 2, height / 2);
    ctx.scale(camera.zoom, camera.zoom);
    ctx.translate(-camera.x, -camera.y);

    // 3. Background grid
    this.drawGrid(camera);

    // 4. Map boundary
    this.drawMapBoundary();

    // 5. Pellets (sorted by position for consistent draw order)
    for (const pellet of state.pellets) {
      this.drawPellet(pellet);
    }

    // 6. Other snakes (draw order: smaller snakes first, player snake last)
    const sortedSnakes = state.snakes
      .filter(s => s.id !== state.playerId)
      .sort((a, b) => a.score - b.score);

    for (const snake of sortedSnakes) {
      this.drawSnake(snake, false);
    }

    // 7. Player snake (always on top)
    const playerSnake = state.snakes.find(s => s.id === state.playerId);
    if (playerSnake) {
      this.drawSnake(playerSnake, true);
    }

    ctx.restore();

    // 8. UI overlay is handled by React (outside canvas)
  }

  private drawSnake(snake: SnakeState, isPlayer: boolean): void {
    const ctx = this.ctx;
    const radius = this.getSnakeRadius(snake.score);

    // Draw body segments (from tail to head)
    for (let i = snake.segments.length - 1; i >= 0; i--) {
      const seg = snake.segments[i];
      ctx.beginPath();
      ctx.arc(seg.x, seg.y, radius, 0, Math.PI * 2);
      ctx.fillStyle = this.getSkinColor(snake.skin, i);
      ctx.fill();
    }

    // Draw head (slightly larger)
    const head = snake.segments[0];
    ctx.beginPath();
    ctx.arc(head.x, head.y, radius * 1.15, 0, Math.PI * 2);
    ctx.fillStyle = this.getSkinColor(snake.skin, 0);
    ctx.fill();

    // Draw eyes
    this.drawEyes(head, snake.angle, radius);

    // Draw name above head
    if (!isPlayer || true) {
      ctx.fillStyle = 'white';
      ctx.strokeStyle = 'black';
      ctx.lineWidth = 3;
      ctx.font = `${Math.max(12, radius)}px Arial`;
      ctx.textAlign = 'center';
      ctx.strokeText(snake.name, head.x, head.y - radius - 10);
      ctx.fillText(snake.name, head.x, head.y - radius - 10);
    }
  }

  private drawPellet(pellet: PelletState): void {
    const ctx = this.ctx;
    const glowRadius = PelletRadius * 2;

    // Glow effect
    const gradient = ctx.createRadialGradient(
      pellet.x, pellet.y, 0,
      pellet.x, pellet.y, glowRadius
    );
    gradient.addColorStop(0, this.getPelletColor(pellet.color, 0.8));
    gradient.addColorStop(1, this.getPelletColor(pellet.color, 0));
    ctx.beginPath();
    ctx.arc(pellet.x, pellet.y, glowRadius, 0, Math.PI * 2);
    ctx.fillStyle = gradient;
    ctx.fill();

    // Core
    ctx.beginPath();
    ctx.arc(pellet.x, pellet.y, PelletRadius, 0, Math.PI * 2);
    ctx.fillStyle = this.getPelletColor(pellet.color, 1);
    ctx.fill();
  }

  private drawGrid(camera: Camera): void {
    const ctx = this.ctx;
    const gridSize = 100;
    const startX = Math.floor((camera.x - camera.viewWidth / 2) / gridSize) * gridSize;
    const startY = Math.floor((camera.y - camera.viewHeight / 2) / gridSize) * gridSize;
    const endX = camera.x + camera.viewWidth / 2;
    const endY = camera.y + camera.viewHeight / 2;

    ctx.strokeStyle = 'rgba(255, 255, 255, 0.05)';
    ctx.lineWidth = 1;

    for (let x = startX; x <= endX; x += gridSize) {
      ctx.beginPath();
      ctx.moveTo(x, startY);
      ctx.lineTo(x, endY);
      ctx.stroke();
    }
    for (let y = startY; y <= endY; y += gridSize) {
      ctx.beginPath();
      ctx.moveTo(startX, y);
      ctx.lineTo(endX, y);
      ctx.stroke();
    }
  }

  private drawMapBoundary(): void {
    const ctx = this.ctx;
    ctx.beginPath();
    ctx.arc(0, 0, MapRadius, 0, Math.PI * 2);
    ctx.strokeStyle = 'rgba(255, 0, 0, 0.5)';
    ctx.lineWidth = 10;
    ctx.stroke();

    // Danger zone (inner warning ring)
    ctx.beginPath();
    ctx.arc(0, 0, MapRadius - 200, 0, Math.PI * 2);
    ctx.strokeStyle = 'rgba(255, 0, 0, 0.15)';
    ctx.lineWidth = 2;
    ctx.stroke();
  }

  private getSnakeRadius(score: number): number {
    return Math.min(MinRadiusBase + Math.sqrt(score) * 0.5, MaxRadius);
  }
}
```

### 3.3 Camera

```typescript
class Camera {
  x: number = 0;
  y: number = 0;
  zoom: number = 1.0;
  targetZoom: number = 1.0;

  // Computed from canvas dimensions and zoom
  get viewWidth(): number {
    return this.canvasWidth / this.zoom;
  }
  get viewHeight(): number {
    return this.canvasHeight / this.zoom;
  }

  update(dt: number, playerHead: Vec2 | null, playerScore: number): void {
    if (!playerHead) return;

    // Smooth follow (lerp toward player head)
    const lerpFactor = 1 - Math.pow(0.001, dt); // Smooth exponential lerp
    this.x += (playerHead.x - this.x) * lerpFactor;
    this.y += (playerHead.y - this.y) * lerpFactor;

    // Zoom based on snake size
    this.targetZoom = this.calculateZoom(playerScore);
    this.zoom += (this.targetZoom - this.zoom) * lerpFactor * 0.5;
  }

  private calculateZoom(score: number): number {
    // zoom = clamp(1.0 - (score / ZoomScoreMax) * 0.5, MinZoom, MaxZoom)
    const zoomRange = MaxZoom - MinZoom; // 1.0 - 0.5 = 0.5
    const scoreFactor = Math.min(score / ZoomScoreMax, 1.0);
    return Math.max(MinZoom, MaxZoom - scoreFactor * zoomRange);
  }

  // Convert world position to screen position
  worldToScreen(worldPos: Vec2): Vec2 {
    return {
      x: (worldPos.x - this.x) * this.zoom + this.canvasWidth / 2,
      y: (worldPos.y - this.y) * this.zoom + this.canvasHeight / 2,
    };
  }
}
```

**Zoom formula:** `zoom = clamp(1.0 - (score / 20000) * 0.5, 0.5, 1.0)`

- Score 0: zoom = 1.0 (fully zoomed in)
- Score 10000: zoom = 0.75 (medium zoom)
- Score 20000+: zoom = 0.5 (fully zoomed out)

### 3.4 Input Handler

```typescript
class InputHandler {
  private angle: number = 0;
  private boosting: boolean = false;
  private canvas: HTMLCanvasElement;
  private camera: Camera;

  // Touch support
  private touchActive: boolean = false;
  private touchStart: Vec2 | null = null;
  private touchCurrent: Vec2 | null = null;

  constructor(canvas: HTMLCanvasElement, camera: Camera) {
    this.canvas = canvas;
    this.camera = camera;

    // Mouse events
    canvas.addEventListener('mousemove', this.onMouseMove.bind(this));
    canvas.addEventListener('mousedown', this.onMouseDown.bind(this));
    canvas.addEventListener('mouseup', this.onMouseUp.bind(this));

    // Keyboard events
    window.addEventListener('keydown', this.onKeyDown.bind(this));
    window.addEventListener('keyup', this.onKeyUp.bind(this));

    // Touch events (virtual joystick)
    canvas.addEventListener('touchstart', this.onTouchStart.bind(this));
    canvas.addEventListener('touchmove', this.onTouchMove.bind(this));
    canvas.addEventListener('touchend', this.onTouchEnd.bind(this));
  }

  private onMouseMove(e: MouseEvent): void {
    const rect = this.canvas.getBoundingClientRect();
    const centerX = rect.width / 2;
    const centerY = rect.height / 2;
    const dx = e.clientX - rect.left - centerX;
    const dy = e.clientY - rect.top - centerY;
    this.angle = Math.atan2(dy, dx);
  }

  private onKeyDown(e: KeyboardEvent): void {
    if (e.code === 'Space') {
      this.boosting = true;
    }
  }

  private onKeyUp(e: KeyboardEvent): void {
    if (e.code === 'Space') {
      this.boosting = false;
    }
  }

  private onMouseDown(e: MouseEvent): void {
    if (e.button === 0) { // Left click = boost
      this.boosting = true;
    }
  }

  private onMouseUp(e: MouseEvent): void {
    if (e.button === 0) {
      this.boosting = false;
    }
  }

  // Touch: virtual joystick
  private onTouchStart(e: TouchEvent): void {
    e.preventDefault();
    const touch = e.touches[0];
    this.touchActive = true;
    this.touchStart = { x: touch.clientX, y: touch.clientY };
    this.touchCurrent = { x: touch.clientX, y: touch.clientY };

    // Two-finger touch = boost
    if (e.touches.length >= 2) {
      this.boosting = true;
    }
  }

  private onTouchMove(e: TouchEvent): void {
    e.preventDefault();
    if (!this.touchActive) return;
    const touch = e.touches[0];
    this.touchCurrent = { x: touch.clientX, y: touch.clientY };

    const dx = this.touchCurrent.x - this.touchStart!.x;
    const dy = this.touchCurrent.y - this.touchStart!.y;
    if (Math.sqrt(dx * dx + dy * dy) > 10) { // Dead zone
      this.angle = Math.atan2(dy, dx);
    }
  }

  private onTouchEnd(e: TouchEvent): void {
    if (e.touches.length === 0) {
      this.touchActive = false;
      this.boosting = false;
    }
  }

  getInput(): { angle: number; boost: boolean } {
    return { angle: this.angle, boost: this.boosting };
  }
}
```

### 3.5 Network Layer

```typescript
class WebSocketClient {
  private ws: WebSocket | null = null;
  private url: string;
  private reconnectAttempts: number = 0;
  private messageHandler: MessageHandler;

  constructor(url: string, messageHandler: MessageHandler) {
    this.url = url;
    this.messageHandler = messageHandler;
  }

  connect(): void {
    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      console.log('Connected to server');
      this.reconnectAttempts = 0;
    };

    this.ws.onmessage = (event: MessageEvent) => {
      const msg = JSON.parse(event.data);
      this.messageHandler.handle(msg);
    };

    this.ws.onclose = () => {
      console.log('Disconnected from server');
      this.attemptReconnect();
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
  }

  send(msg: object): void {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(msg));
    }
  }

  private attemptReconnect(): void {
    if (this.reconnectAttempts >= MaxReconnectAttempts) {
      console.error('Max reconnection attempts reached');
      return;
    }
    this.reconnectAttempts++;
    setTimeout(() => this.connect(), ReconnectInterval);
  }

  disconnect(): void {
    this.ws?.close();
  }
}
```

**Input sending** runs at the same rate as the server tick (30 TPS):

```typescript
// In the game engine, send input every tick interval
setInterval(() => {
  const input = this.inputHandler.getInput();
  this.wsClient.send({
    type: 'input',
    angle: input.angle,
    boost: input.boost,
  });
}, TickInterval);
```

### 3.6 State Buffer + Interpolation

The client maintains a **buffer of the last 2 server states** and interpolates between them with a **100ms render delay**.

```typescript
class StateBuffer {
  private states: ServerState[] = [];
  private renderDelay: number = RenderDelay; // 100ms

  addState(state: ServerState): void {
    state.receivedAt = performance.now();
    this.states.push(state);

    // Keep only last 3 states (for safety)
    if (this.states.length > 3) {
      this.states.shift();
    }
  }

  getInterpolatedState(): InterpolatedState | null {
    if (this.states.length < 2) return null;

    const renderTime = performance.now() - this.renderDelay;
    const prev = this.states[this.states.length - 2];
    const curr = this.states[this.states.length - 1];

    // Calculate interpolation factor
    const duration = curr.receivedAt - prev.receivedAt;
    if (duration <= 0) return this.toInterpolated(curr, 1);

    const elapsed = renderTime - prev.receivedAt;
    const t = Math.max(0, Math.min(1, elapsed / duration));

    return this.interpolate(prev, curr, t);
  }

  private interpolate(
    prev: ServerState,
    curr: ServerState,
    t: number
  ): InterpolatedState {
    const snakes: SnakeState[] = [];

    for (const currSnake of curr.snakes) {
      const prevSnake = prev.snakes.find(s => s.id === currSnake.id);

      if (prevSnake) {
        // Interpolate each segment position
        const segments = currSnake.segments.map((seg, i) => {
          const prevSeg = prevSnake.segments[i] || seg;
          return {
            x: lerp(prevSeg.x, seg.x, t),
            y: lerp(prevSeg.y, seg.y, t),
          };
        });

        snakes.push({
          ...currSnake,
          segments,
          // Angle is interpolated using shortest-path angular lerp
          angle: lerpAngle(prevSnake.angle, currSnake.angle, t),
        });
      } else {
        // New snake, no interpolation
        snakes.push(currSnake);
      }
    }

    return {
      snakes,
      pellets: curr.pellets, // Pellets: use latest state (no lerp needed)
      playerId: curr.playerId,
    };
  }
}
```

**Important:** Snake **eye direction** is rendered using the **local input angle** (instant, no delay), not the interpolated server angle. This provides immediate visual feedback for the player's own snake:

```typescript
// In the renderer, for the player's own snake:
if (isPlayerSnake) {
  // Use local input angle for eyes (instant feedback)
  this.drawEyes(head, this.inputHandler.getInput().angle, radius);
} else {
  // Use interpolated server angle for other snakes
  this.drawEyes(head, snake.angle, radius);
}
```

### 3.7 UI Overlay

React components are rendered **on top of the canvas** using absolute positioning.

```typescript
// HUD.tsx - Container for all UI overlay components
const HUD: React.FC<HUDProps> = ({
  score,
  leaderboard,
  isDead,
  killerName,
  mapRadius,
  playerPos,
  snakePositions,
  boostAvailable,
  onRespawn,
}) => {
  return (
    <div className="hud-overlay">
      {/* Top-right: Leaderboard */}
      <Leaderboard entries={leaderboard.top} playerRank={leaderboard.rank} />

      {/* Bottom-left: Minimap */}
      <Minimap
        mapRadius={mapRadius}
        playerPos={playerPos}
        snakePositions={snakePositions}
      />

      {/* Bottom-center: Score display */}
      <ScoreDisplay score={score} />

      {/* Bottom: Boost indicator */}
      <BoostIndicator
        score={score}
        minBoostScore={MinBoostScore}
        available={boostAvailable}
      />

      {/* Center: Death screen (conditional) */}
      {isDead && (
        <DeathScreen
          score={score}
          killerName={killerName}
          onRespawn={onRespawn}
        />
      )}
    </div>
  );
};
```

---

## 4. Network Protocol

All messages are **JSON** in the MVP. A future optimization is switching to a binary format (MessagePack or custom).

### 4.1 Client to Server Messages

#### `input` - Player Input (sent every tick, ~30/s)

```json
{
  "type": "input",
  "angle": 1.5708,
  "boost": false
}
```

| Field   | Type    | Description                              |
|---------|---------|------------------------------------------|
| `type`  | string  | Always `"input"`                         |
| `angle` | number  | Target angle in radians (-PI to PI)      |
| `boost` | boolean | Whether the player is holding boost      |

#### `join` - Join Game

```json
{
  "type": "join",
  "name": "Player1",
  "skin": 3
}
```

| Field  | Type   | Description                    |
|--------|--------|--------------------------------|
| `type` | string | Always `"join"`                |
| `name` | string | Player display name (max 15)   |
| `skin` | number | Skin/color index (0-7)         |

#### `respawn` - Respawn After Death

```json
{
  "type": "respawn",
  "name": "Player1",
  "skin": 3
}
```

| Field  | Type   | Description                    |
|--------|--------|--------------------------------|
| `type` | string | Always `"respawn"`             |
| `name` | string | Player display name            |
| `skin` | number | Skin/color index               |

### 4.2 Server to Client Messages

#### `welcome` - Connection Accepted

```json
{
  "type": "welcome",
  "id": "abc123",
  "mapRadius": 4000,
  "tickRate": 30
}
```

| Field       | Type   | Description                    |
|-------------|--------|--------------------------------|
| `type`      | string | Always `"welcome"`             |
| `id`        | string | Assigned player ID             |
| `mapRadius` | number | Map radius in world units      |
| `tickRate`  | number | Server tick rate (30)          |

#### `state` - Game State Update (sent every tick, ~30/s)

```json
{
  "type": "state",
  "tick": 12345,
  "snakes": [
    {
      "id": "abc123",
      "segments": [[100.5, 200.3], [95.2, 198.1], [90.0, 196.5]],
      "angle": 1.5708,
      "score": 150,
      "name": "Player1",
      "skin": 3,
      "boosting": false,
      "alive": true
    }
  ],
  "pellets": [
    { "id": 5001, "x": 300.0, "y": 450.0, "value": 3, "color": 2 }
  ],
  "removedPellets": [4998, 4999, 5000]
}
```

**Snake entry fields:**

| Field      | Type       | Description                              |
|------------|------------|------------------------------------------|
| `id`       | string     | Snake ID                                 |
| `segments` | number[][] | Array of [x, y] coordinate pairs         |
| `angle`    | number     | Current facing angle (radians)           |
| `score`    | number     | Current score                            |
| `name`     | string     | Display name                             |
| `skin`     | number     | Skin/color index                         |
| `boosting` | boolean    | Currently boosting                       |
| `alive`    | boolean    | Is alive                                 |

**Pellet entry fields:**

| Field   | Type   | Description              |
|---------|--------|--------------------------|
| `id`    | number | Unique pellet ID         |
| `x`     | number | X position               |
| `y`     | number | Y position               |
| `value` | number | Point value (1-5)        |
| `color` | number | Color index              |

**`removedPellets`**: Array of pellet IDs that were consumed since last tick. The client removes these from its local pellet cache.

#### `death` - Player Died

```json
{
  "type": "death",
  "killerId": "def456",
  "score": 1500
}
```

| Field      | Type   | Description                          |
|------------|--------|--------------------------------------|
| `type`     | string | Always `"death"`                     |
| `killerId` | string | ID of the snake that killed you (empty if wall) |
| `score`    | number | Final score at time of death         |

#### `leaderboard` - Leaderboard Update (every 2 seconds)

```json
{
  "type": "leaderboard",
  "top": [
    { "name": "ProSnake", "score": 5000 },
    { "name": "Player1", "score": 3200 }
  ],
  "rank": 5,
  "total": 47
}
```

| Field   | Type     | Description                        |
|---------|----------|------------------------------------|
| `type`  | string   | Always `"leaderboard"`             |
| `top`   | object[] | Top 10 players [{name, score}]     |
| `rank`  | number   | Current player's rank              |
| `total` | number   | Total players in room              |

### 4.3 Bandwidth Estimation

#### Downstream (Server to Client)

**State message size per tick (worst case, no interest management):**

| Component               | Size Estimate                                    |
|--------------------------|--------------------------------------------------|
| 50 snakes                | 50 x (id: 4B + segments: 20 x 8B + angle: 4B + score: 4B + flags: 4B) |
| Per snake                | ~180 bytes                                       |
| All snakes               | 50 x 180 = **~9,000 bytes**                     |
| 200 visible pellets      | 200 x (id: 4B + x: 4B + y: 4B + value: 1B + color: 1B) = **~2,800 bytes** |
| Message overhead         | ~200 bytes (JSON keys, formatting)               |
| **Total per tick**       | **~12 KB** (JSON, uncompressed)                  |

**With interest management** (viewport filtering):
- Typical visible: ~15-25 snakes, ~100-150 pellets
- Estimated: **~5-8 KB per tick**

**Downstream bandwidth:**
- Without interest management: 12 KB x 30 TPS = **~360 KB/s**
- With interest management: 6 KB x 30 TPS = **~180 KB/s**
- With future binary protocol: **~60-70 KB/s**

#### Upstream (Client to Server)

| Component    | Size       |
|--------------|------------|
| Input message| ~20 bytes  |
| At 30 TPS    | **~600 B/s** |

---

## 5. Game Physics

All physics are computed **server-side only**. The client is purely a renderer with input collection.

### 5.1 Movement Model

**Angle-based steering:**

```go
func (s *Snake) UpdateMovement(dt float64) {
    // 1. Steer current angle toward target angle
    angleDiff := normalizeAngle(s.TargetAngle - s.Angle)
    maxTurn := TurnRate * dt // 3.0 rad/s * dt

    if math.Abs(angleDiff) < maxTurn {
        s.Angle = s.TargetAngle
    } else if angleDiff > 0 {
        s.Angle += maxTurn
    } else {
        s.Angle -= maxTurn
    }
    s.Angle = normalizeAngle(s.Angle)

    // 2. Determine speed
    speed := BaseSpeed
    if s.Boosting && s.Score > MinBoostScore {
        speed = BoostSpeed
    } else {
        s.Boosting = false // Force stop if below min score
    }

    // 3. Move head forward
    dx := math.Cos(s.Angle) * speed * dt
    dy := math.Sin(s.Angle) * speed * dt
    s.Segments[0].X += dx
    s.Segments[0].Y += dy

    // 4. Body follows: each segment lerps toward the one in front
    for i := 1; i < len(s.Segments); i++ {
        prev := s.Segments[i-1]
        curr := &s.Segments[i]
        dx := prev.X - curr.X
        dy := prev.Y - curr.Y
        dist := math.Sqrt(dx*dx + dy*dy)

        if dist > SegmentSpacing {
            // Move segment toward the one in front, maintaining spacing
            ratio := SegmentSpacing / dist
            curr.X = prev.X - dx*ratio
            curr.Y = prev.Y - dy*ratio
        }
    }
}
```

**Key parameters:**
- Turn rate: **3.0 rad/s** max (smooth, not instant turning)
- Base speed: **200 units/s**
- Boost speed: **400 units/s** (2x base)
- Segment spacing: **10 units** between each body segment

### 5.2 Collision Detection

All collision detection uses the spatial hash grid for efficient lookups.

#### Head-to-Body Collision

```go
func (r *Room) checkHeadBodyCollisions() {
    for _, snake := range r.snakes {
        if !snake.Alive continue

        head := snake.Segments[0]
        snakeRadius := getRadius(snake.Score)

        // Query spatial hash for nearby segments
        nearby := r.spatialHash.QuerySegments(
            head.X - snakeRadius*2,
            head.Y - snakeRadius*2,
            head.X + snakeRadius*2,
            head.Y + snakeRadius*2,
        )

        for _, segRef := range nearby {
            // Skip own body (self-collision disabled per snake.io rules)
            if segRef.SnakeID == snake.ID continue

            other := r.snakes[segRef.SnakeID]
            if !other.Alive continue

            otherRadius := getRadius(other.Score)
            collisionDist := snakeRadius + otherRadius

            seg := other.Segments[segRef.Index]
            dx := head.X - seg.X
            dy := head.Y - seg.Y
            dist := math.Sqrt(dx*dx + dy*dy)

            if dist < collisionDist {
                // Snake dies!
                snake.Alive = false
                snake.KilledBy = other.ID
                other.Score += snake.Score / 5 // Killer bonus
                break
            }
        }
    }
}
```

#### Head-to-Wall Collision

```go
func (r *Room) checkWallCollisions() {
    for _, snake := range r.snakes {
        if !snake.Alive { continue }

        head := snake.Segments[0]
        snakeRadius := getRadius(snake.Score)
        distFromCenter := math.Sqrt(head.X*head.X + head.Y*head.Y)

        if distFromCenter > MapRadius - snakeRadius {
            snake.Alive = false
            snake.KilledBy = "" // Wall kill
        }
    }
}
```

#### Head-to-Pellet Collision

```go
func (r *Room) checkPelletCollisions() {
    for _, snake := range r.snakes {
        if !snake.Alive { continue }

        head := snake.Segments[0]
        snakeRadius := getRadius(snake.Score)
        collectDist := snakeRadius + PelletRadius

        // Query spatial hash for nearby pellets
        nearby := r.spatialHash.QueryPellets(
            head.X - collectDist,
            head.Y - collectDist,
            head.X + collectDist,
            head.Y + collectDist,
        )

        for _, pellet := range nearby {
            dx := head.X - pellet.X
            dy := head.Y - pellet.Y
            dist := math.Sqrt(dx*dx + dy*dy)

            if dist < collectDist {
                snake.Score += pellet.Value
                r.removedPellets = append(r.removedPellets, pellet.ID)
                delete(r.pellets, pellet.ID)
                r.spatialHash.Remove(pellet)
            }
        }
    }
}
```

#### Head-to-Head Collision

```go
func (r *Room) checkHeadHeadCollisions() {
    snakeList := r.getAliveSnakes()

    for i := 0; i < len(snakeList); i++ {
        for j := i + 1; j < len(snakeList); j++ {
            a := snakeList[i]
            b := snakeList[j]

            headA := a.Segments[0]
            headB := b.Segments[0]

            radiusA := getRadius(a.Score)
            radiusB := getRadius(b.Score)
            collisionDist := radiusA + radiusB

            dx := headA.X - headB.X
            dy := headA.Y - headB.Y
            dist := math.Sqrt(dx*dx + dy*dy)

            if dist < collisionDist {
                if a.Score > b.Score {
                    b.Alive = false
                    b.KilledBy = a.ID
                } else if b.Score > a.Score {
                    a.Alive = false
                    a.KilledBy = b.ID
                } else {
                    // Equal score: both die
                    a.Alive = false
                    b.Alive = false
                }
            }
        }
    }
}
```

#### Self-Collision

**DISABLED** per snake.io rules. This allows the coiling/encircling strategy where players wrap around other snakes or pellet clusters.

### 5.3 Snake Radius Formula

```go
func getRadius(score int) float64 {
    radius := MinRadiusBase + math.Sqrt(float64(score)) * 0.5
    if radius > MaxRadius {
        return MaxRadius
    }
    return radius
}
```

| Score  | Radius | Visual Size  |
|--------|--------|--------------|
| 10     | 11.6   | Tiny         |
| 100    | 15.0   | Small        |
| 500    | 21.2   | Medium       |
| 1000   | 25.8   | Large        |
| 5000   | 40.0   | Maximum      |

### 5.4 Growth Mechanic

```go
func (r *Room) processGrowth() {
    for _, snake := range r.snakes {
        if !snake.Alive { continue }

        // Calculate expected segments based on score
        expectedSegments := InitialSegments + (snake.Score / GrowthThreshold)

        // Add segments at tail if needed
        for len(snake.Segments) < expectedSegments {
            tail := snake.Segments[len(snake.Segments)-1]
            snake.Segments = append(snake.Segments, Vec2{X: tail.X, Y: tail.Y})
        }
    }
}
```

**Growth rate:** 1 new segment per `GrowthThreshold` (50) score points.

### 5.5 Boost Mechanic

```go
func (r *Room) processBoost(dt float64) {
    for _, snake := range r.snakes {
        if !snake.Alive || !snake.Boosting { continue }

        // Check minimum score
        if snake.Score <= MinBoostScore {
            snake.Boosting = false
            continue
        }

        // Consume mass
        snake.Score -= MassLossRate // 2 per tick

        // Spawn trail pellet every N ticks
        if r.currentTick % BoostTrailInterval == 0 {
            tail := snake.Segments[len(snake.Segments)-1]
            pellet := &Pellet{
                ID:    r.nextPelletID(),
                X:     tail.X + (rand.Float64()-0.5)*10,
                Y:     tail.Y + (rand.Float64()-0.5)*10,
                Value: BoostTrailValue,
                Color: snake.Skin,
            }
            r.pellets[pellet.ID] = pellet
            r.spatialHash.Insert(pellet)
        }
    }
}
```

**Boost details:**
- Speed doubles: 200 -> 400 units/s
- Mass loss: 2 score per tick (60 score/second)
- Trail pellets: spawned every 3 ticks behind the tail, value 1 each
- Minimum score to boost: 50 (boosting auto-stops below this)

---

## 6. Data Flow Diagrams

### 6.1 Player Join Flow

```
Client                          Server
  │                               │
  │──── WebSocket Connect ───────>│
  │                               │ Accept connection
  │                               │ Create Client struct
  │                               │
  │──── { type: "join",      ────>│
  │       name: "Player1",        │ Assign to Room
  │       skin: 3 }               │ Create Snake (random pos)
  │                               │ Add to spatial hash
  │                               │
  │<─── { type: "welcome",  ─────│
  │       id: "abc123",           │
  │       mapRadius: 4000,        │
  │       tickRate: 30 }          │
  │                               │
  │<─── { type: "state", ... } ──│ (next tick includes new snake)
  │                               │
  │     [Game begins]             │
```

### 6.2 Game Tick Flow (every 33ms)

```
┌──────────────────────────────────────────────────────┐
│                    GAME TICK                          │
│                                                      │
│  Step 1: DRAIN INPUT QUEUE                           │
│  ├── Read all pending player inputs from channel     │
│  └── Apply: update targetAngle, boost state          │
│                                                      │
│  Step 2: PHYSICS UPDATE                              │
│  ├── For each snake:                                 │
│  │   ├── Steer angle toward targetAngle              │
│  │   ├── Move head by speed × dt                     │
│  │   └── Body segments follow head                   │
│  └── Process boost (mass loss, trail pellets)        │
│                                                      │
│  Step 3: COLLISION DETECTION (spatial hash queries)   │
│  ├── Head-to-body collisions                         │
│  ├── Head-to-wall collisions                         │
│  ├── Head-to-pellet collisions                       │
│  └── Head-to-head collisions                         │
│                                                      │
│  Step 4: PROCESS DEATHS                              │
│  ├── Mark dead snakes                                │
│  └── Convert dead snakes to pellet clusters          │
│       └── totalPelletValue = score × 0.8             │
│                                                      │
│  Step 5: PROCESS GROWTH                              │
│  ├── Update scores from consumed pellets             │
│  └── Add tail segments (1 per 50 score)              │
│                                                      │
│  Step 6: RESPAWN NATURAL PELLETS                     │
│  └── If pellet count < 1000, spawn to fill           │
│                                                      │
│  Step 7: UPDATE SPATIAL HASH                         │
│  └── Rebuild grid with new entity positions          │
│                                                      │
│  Step 8: BROADCAST STATE                             │
│  ├── For each connected player:                      │
│  │   ├── Calculate viewport (head pos + zoom)        │
│  │   ├── Filter entities by viewport × 1.5 margin   │
│  │   └── Send filtered state message                 │
│  └── Every 60 ticks (2s): send leaderboard           │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 6.3 Client Render Frame (every ~16ms at 60fps)

```
┌──────────────────────────────────────────────────────┐
│                  RENDER FRAME                         │
│                                                      │
│  Step 1: INTERPOLATION                               │
│  ├── renderTime = now - 100ms (render delay)         │
│  ├── Find bracketing states: prev and curr           │
│  ├── t = (renderTime - prev.time) / (curr - prev)   │
│  └── For each snake: lerp segment positions by t     │
│                                                      │
│  Step 2: LOCAL INPUT (instant, no delay)             │
│  ├── Player snake eye direction = local mouse angle  │
│  └── No interpolation on player's own eye direction  │
│                                                      │
│  Step 3: CAMERA TRANSFORM                            │
│  ├── Lerp camera position toward player head         │
│  ├── Calculate zoom from player score                │
│  └── Apply translate + scale to canvas context       │
│                                                      │
│  Step 4: DRAW LAYERS (back to front)                 │
│  ├── Layer 1: Background grid                        │
│  ├── Layer 2: Map boundary ring                      │
│  ├── Layer 3: Pellets (with glow effect)             │
│  ├── Layer 4: Other snakes (smaller first)           │
│  ├── Layer 5: Player snake (always on top)           │
│  └── Layer 6: React UI overlay (separate DOM)        │
│                                                      │
│  Step 5: UPDATE UI COMPONENTS                        │
│  ├── Score display                                   │
│  ├── Leaderboard (when new data arrives)             │
│  ├── Minimap                                         │
│  └── Boost indicator                                 │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 6.4 Death / Respawn Flow

```
Server                          Client
  │                               │
  │ Detect collision              │
  │ Mark snake as dead            │
  │ Convert body to pellets       │
  │   (score × 0.8 total value)   │
  │                               │
  │──── { type: "death",    ─────>│
  │       killerId: "def456",     │ Show death screen
  │       score: 1500 }           │ Display final score
  │                               │ Show killer name
  │                               │
  │        ... player clicks      │
  │            "Play Again" ...   │
  │                               │
  │<─── { type: "respawn",  ─────│
  │       name: "Player1",        │
  │       skin: 3 }               │ Hide death screen
  │                               │
  │ Create new snake              │
  │ Random position               │
  │ Score = 10, segments = 10     │
  │                               │
  │──── { type: "state", ... } ──>│ (next tick includes new snake)
  │                               │
```

---

## 7. Configuration Constants

```go
package config

// =============================================================================
// Map
// =============================================================================

const MapRadius = 4000.0 // Circular arena radius in world units
// MapCenter is implicitly {0, 0}

// =============================================================================
// Game Loop
// =============================================================================

const TickRate = 30      // Ticks per second
const TickInterval = 33  // Milliseconds per tick (1000 / TickRate)

// =============================================================================
// Room
// =============================================================================

const MaxPlayersPerRoom = 50    // Maximum entities (humans + bots) per room
const MinBotsPerRoom = 20       // Minimum bots to keep room feeling active
const TargetEntitiesPerRoom = 50 // Bots fill to reach this count
const RoomDestroyTimeout = 60   // Seconds with 0 humans before room is destroyed

// =============================================================================
// Snake
// =============================================================================

const InitialScore = 10         // Starting score for new snakes
const InitialSegments = 10      // Starting body segment count
const SegmentSpacing = 10.0     // Distance between body segments (world units)
const BaseSpeed = 200.0         // Normal movement speed (units/second)
const BoostSpeed = 400.0        // Boosting movement speed (units/second)
const TurnRate = 3.0            // Maximum turn speed (radians/second)
const MinRadiusBase = 10.0      // Minimum snake body radius
const MaxRadius = 40.0          // Maximum snake body radius (cap)
const GrowthThreshold = 50      // Score points per new body segment

// Radius formula: radius = MinRadiusBase + sqrt(score) * 0.5, capped at MaxRadius

// =============================================================================
// Boost
// =============================================================================

const MassLossRate = 2          // Score consumed per tick while boosting
const BoostTrailInterval = 3    // Spawn a trail pellet every N ticks while boosting
const MinBoostScore = 50        // Minimum score required to activate/maintain boost
const BoostTrailValue = 1       // Point value of each trail pellet

// =============================================================================
// Pellets
// =============================================================================

const NaturalPelletCount = 1000 // Target number of natural pellets per room
const NaturalPelletMinValue = 1 // Minimum value of a natural pellet
const NaturalPelletMaxValue = 5 // Maximum value of a natural pellet
const PelletRadius = 5.0        // Collision/render radius of pellets
const DeathDropRatio = 0.8      // 80% of score converted to pellets on death
const DeathPelletValue = 5      // Point value of each death-drop pellet

// =============================================================================
// Spatial Hash
// =============================================================================

const SpatialCellSize = 200.0   // Grid cell size in world units

// =============================================================================
// Network / Interest Management
// =============================================================================

const ViewportWidth = 1920.0    // Reference viewport width (pixels)
const ViewportHeight = 1080.0   // Reference viewport height (pixels)
const InterestMargin = 1.5      // Viewport multiplier for interest management
const LeaderboardInterval = 60  // Send leaderboard every N ticks (60 ticks = 2 seconds)
const LeaderboardSize = 10      // Number of top players in leaderboard

// =============================================================================
// Camera (client-side, mirrored here for server interest calculation)
// =============================================================================

const MinZoom = 0.5             // Maximum zoom out (smallest objects)
const MaxZoom = 1.0             // Maximum zoom in (largest objects)
const ZoomScoreMax = 20000      // Score at which zoom reaches MinZoom

// Zoom formula: zoom = clamp(1.0 - (score / ZoomScoreMax) * 0.5, MinZoom, MaxZoom)

// =============================================================================
// Client-Side Constants (defined in client/src/utils/constants.ts)
// =============================================================================

const RenderDelay = 100         // Milliseconds of render delay for interpolation
const ReconnectInterval = 3000  // Milliseconds between reconnection attempts
const MaxReconnectAttempts = 5  // Maximum number of reconnection attempts
```

---

## 8. Scalability

### 8.1 Single Server Capacity Estimation

#### CPU

| Metric                          | Estimate                                    |
|---------------------------------|---------------------------------------------|
| Per-room tick cost              | 50 entities x (physics + collision) ~ **2ms** |
| Ticks per second                | 30                                          |
| CPU time per room per second    | 2ms x 30 = **60ms** (6% of one core)       |
| Rooms per CPU core              | ~16 rooms (with overhead)                   |
| 4-core server                   | ~50-60 rooms                                |
| Players per server (4 cores)    | 50-60 rooms x 50 players = **2,500-3,000**  |

#### Memory

| Component                       | Size Per Room                               |
|---------------------------------|---------------------------------------------|
| 50 snakes x 200 segments x 16B  | 160 KB                                      |
| 50 snakes x metadata            | 5 KB                                        |
| 1000 pellets x 20B              | 20 KB                                       |
| Spatial hash overhead            | 10 KB                                       |
| **Total per room**              | **~200 KB**                                 |

| Scale                          | Memory Usage                                |
|--------------------------------|---------------------------------------------|
| 50 rooms                       | 10 MB                                       |
| 200 rooms                      | 40 MB                                       |
| Go runtime overhead             | ~50 MB                                      |
| **Total (200 rooms)**          | **~90 MB**                                  |

#### Network

| Metric                          | Estimate                                    |
|---------------------------------|---------------------------------------------|
| Per player downstream           | ~150 KB/s (with interest management, JSON)  |
| Per player upstream             | ~600 B/s                                    |
| Per room (50 players)           | 50 x 150 KB/s = **7.5 MB/s**               |
| 50 rooms                        | 375 MB/s = **3 Gbps**                       |
| 200 rooms                       | 1.5 GB/s = **12 Gbps**                      |

**Bottleneck:** Network bandwidth is the primary limiting factor for scaling a single server.

### 8.2 Scaling Path

#### Phase 1: Vertical Scaling

- Upgrade to larger instance (more CPU cores, 10Gbps NIC)
- Capacity: up to ~50-60 rooms (~2,500-3,000 concurrent players)
- Cost-effective for MVP and early growth

#### Phase 2: Binary Protocol (60% bandwidth reduction)

Switch from JSON to a binary format (MessagePack or custom binary):

| Format      | State Message Size | Bandwidth (30 TPS) |
|-------------|-------------------|---------------------|
| JSON        | ~6 KB             | ~180 KB/s           |
| MessagePack | ~3 KB             | ~90 KB/s            |
| Custom binary| ~2.5 KB          | ~75 KB/s            |

This alone doubles effective server capacity from a bandwidth perspective.

#### Phase 3: Delta Compression (70% further reduction)

Instead of sending full state each tick, only send **changes**:

- New pellets (spawned this tick)
- Removed pellets (consumed this tick)
- Changed snake positions (delta from previous)
- New/removed snakes

Expected result: **~20-30 KB/s** per player, allowing 5-6x more rooms per server.

#### Phase 4: Horizontal Scaling

```
                    ┌──────────────┐
                    │ Load Balancer │
                    │  (nginx/HAProxy)
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Game      │ │ Game      │ │ Game      │
        │ Server 1  │ │ Server 2  │ │ Server 3  │
        │ (50 rooms)│ │ (50 rooms)│ │ (50 rooms)│
        └──────────┘ └──────────┘ └──────────┘
```

**Key properties:**
- **Stateless room assignment**: A lobby/matchmaking service assigns players to the least-loaded server.
- **No cross-server state**: Rooms are completely independent. No player can be in two rooms or interact across servers.
- **Simple health checks**: Each server reports room count and player count. Load balancer routes new connections to least-loaded server.
- **Session stickiness**: Once a player is assigned to a server, all WebSocket traffic goes to that server (WebSocket connections are inherently sticky).

#### Phase 5: Regional Deployment

Deploy game servers in multiple regions for latency optimization:

| Region      | Location         | Target Latency |
|-------------|------------------|----------------|
| US East     | Virginia         | <30ms (US)     |
| US West     | Oregon           | <30ms (US)     |
| EU          | Frankfurt        | <30ms (EU)     |
| Asia        | Tokyo / Singapore| <50ms (Asia)   |

**Matchmaking** routes players to the nearest region by default, with an option to select a specific region.

### 8.3 Summary of Scaling Milestones

| Milestone              | Players   | Servers | Key Change                    |
|------------------------|-----------|---------|-------------------------------|
| MVP Launch             | 500       | 1       | JSON, single server           |
| Binary Protocol        | 1,500     | 1       | MessagePack, same server      |
| Delta Compression      | 5,000     | 1       | Delta state, same server      |
| Horizontal Scale       | 20,000    | 4       | Multi-server, load balancer   |
| Regional               | 50,000+   | 12+     | Multi-region deployment       |

---

## Appendix: Technology Choices Summary

| Component          | Technology                  | Rationale                              |
|--------------------|-----------------------------|----------------------------------------|
| Server Language    | Go                          | Goroutines, low latency, efficient networking |
| WebSocket Library  | gorilla/websocket           | Mature, well-tested Go WS library      |
| Frontend Framework | Next.js + TypeScript        | SSR for landing page, strong typing    |
| Rendering          | Canvas 2D                   | Simple, performant for 2D circles      |
| UI Overlay         | React (JSX on top of canvas)| Rich UI components, state management   |
| Wire Format (MVP)  | JSON                        | Easy debugging, fast development       |
| Wire Format (Later)| MessagePack / custom binary | Bandwidth optimization                 |
| Spatial Indexing   | Custom spatial hash grid    | O(1) lookups, simple implementation    |

---

*This document serves as the definitive reference for implementing the Snake.io MMO clone. All systems, constants, and protocols defined here should be followed during implementation.*
